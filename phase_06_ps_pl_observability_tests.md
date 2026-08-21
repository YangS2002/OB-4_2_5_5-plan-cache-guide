# Phase 06：PS、PL、观测与测试实验

## 本阶段重点

把 SQL plan cache 主线扩展到 PS、PL，并用虚拟表和测试做闭环。

本阶段不是再开很多新坑，而是回答：

```text
这套机制除了 text SQL，还服务谁？
用户如何观察它？
我如何证明前面理解是对的？
```

## 不必深挖

先不要追：

- PS prepare 协议所有字段。
- PL compiler 细节。
- package/function 所有失效分支。
- 虚拟表每一列来源。
- 所有 mysql test case。

只看它们如何接入 plan cache / Lib Cache。

## 核心问题

读完只要求回答：

1. PS cache 和 SQL plan cache 是不是一回事？
2. PS execute 如何复用 plan？
3. PL function/package/anonymous block 如何复用 Lib Cache？
4. 虚拟表如何观察 plan cache hit/memory/object？
5. flush plan/ps/pl/lib cache 有什么区别？
6. 哪些实验能证明整个理解闭环？

## Part A：PS Cache 重点

### 关键入口

- `src/sql/plan_cache/ob_prepare_stmt_struct.h`
  - `ObPsSqlKey`
  - `ObPsStmtItem`
  - `ObPsStmtInfo`

- `src/sql/plan_cache/ob_ps_cache.h/.cpp`
  - `get_or_add_stmt_item`
  - `get_or_add_stmt_info`
  - `ref_stmt_info`

- `src/sql/ob_sql.cpp`
  - `ObSql::handle_ps_prepare`
  - `ObSql::handle_ps_execute`

- `src/sql/plan_cache/ob_plan_cache.h/.cpp`
  - `ObPlanCache::get_ps_plan`
  - `ObPlanCache::add_ps_plan`

### 理解模型

```text
PREPARE
  -> 生成 stmt_id
  -> 保存 statement metadata

EXECUTE
  -> 根据 stmt_id 找 metadata
  -> 用 stmt_id / context 找 cached plan
  -> miss 则生成并 add ps plan
```

重点认识：PS cache 存 statement 信息，plan cache 存可执行 physical plan。两者有关，但不是一回事。

### 验证

```sql
prepare s from 'select * from t_pc where a = ?';
set @a = 1;
execute s using @a;
set @a = 2;
execute s using @a;
```

观察：同一个 stmt id 多次 execute，第二次应有复用机会。

## Part B：PL Cache 重点

### 关键入口

- `src/pl/pl_cache/ob_pl_cache.h/.cpp`
  - `ObPLObjectKey`
  - `ObPLObjectSet`

- `src/pl/pl_cache/ob_pl_cache_object.h`
  - `ObPLFunction`
  - `ObPLPackage`

- `src/pl/pl_cache/ob_pl_cache_mgr.h/.cpp`
  - `get_pl_cache`
  - `add_pl_cache`

- `src/pl/ob_pl.cpp`
  - anonymous / function cache 获取与编译。

- `src/pl/ob_pl_package_manager.cpp`
  - package cache 获取与加入。

### 理解模型

```text
PL execution
  -> 构造 ObPLObjectKey
  -> 按 namespace 找 PL object
  -> hit：使用已编译对象
  -> miss：编译 PL object，并 add 到 Lib Cache
```

重点认识：PL 复用的是 compiled PL object，不是普通 SQL text physical plan，但生命周期/namespace/ref/flush 复用 Lib Cache 框架。

### 验证

```sql
alter system flush pl cache;
begin null; end;
begin null; end;
```

预期：第二次 anonymous block 有 cache 复用机会。

## Part C：观测重点

### 关键入口

- `src/observer/virtual_table/ob_all_plan_cache_stat.cpp`
  - plan cache 总体统计。

- `src/observer/virtual_table/ob_gv_sql.cpp`
  - plan object / PL object 遍历。

- `src/observer/virtual_table/ob_plan_cache_plan_explain.cpp`
  - cached plan explain。

- `src/observer/mysql/obmp_query.cpp`
- `src/observer/mysql/obmp_stmt_execute.cpp`
  - audit 中 plan cache hit 信息。

### 理解模型

```text
执行 SQL
  -> plan cache stat 更新
  -> virtual table 遍历 plan cache object
  -> 展示 hit_count / memory / plan_id / sql_id / statement
```

虚拟表不是另一套数据源，它是观察 `ObPlanCache` 内部状态的窗口。

### 验证

```sql
select * from oceanbase.GV$OB_PLAN_CACHE_STAT;

select plan_id, sql_id, hit_count, statement
from oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT
where statement like '%t_pc%';

select *
from oceanbase.GV$OB_PLAN_CACHE_PLAN_EXPLAIN
where plan_id = xxx;
```

重点观察：

- hit_count 是否增加。
- flush 后记录是否消失。
- explain 是否能展示 cached plan。

## Part D：测试怎么用

### 只重点看这些测试意图

- `unittest/sql/plan_cache/test_plan_cache.cpp`
  - 基础 get/add/并发。

- `unittest/sql/plan_cache/test_plan_cache_value.cpp`
  - value match。

- `unittest/sql/plan_cache/test_sql_parameterization.cpp`
  - 参数化。

- `tools/deploy/mysql_test/test_suite/plan_cache/t/plan_cache_select_list.test`
  - 用 SQL 验证参数化与 hit_count。

- `tools/deploy/mysql_test/test_suite/plan_cache/t/plan_cache_multi_query.test`
  - multi query 行为。

不要逐行背测试。看它们验证了哪类行为。

## 最终闭环实验

### 1. Text SQL hit/miss

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

证明：miss -> add -> hit。

### 2. key/context 变化

```sql
set sql_mode = '';
select * from t_pc where a = 1;
set sql_mode = 'ANSI_QUOTES';
select * from t_pc where a = 1;
```

证明：上下文影响复用。

### 3. schema 失效

```sql
select * from t_pc where a = 1;
alter table t_pc add column c int;
select * from t_pc where a = 1;
```

证明：cached plan 需要有效性检查。

### 4. flush 生命周期

```sql
select * from t_pc where a = 1;
alter system flush plan cache;
select * from t_pc where a = 2;
```

证明：flush 后重新生成。

### 5. PS/PL 扩展

```sql
prepare s from 'select * from t_pc where a = ?';
set @a = 1;
execute s using @a;
set @a = 2;
execute s using @a;

alter system flush pl cache;
begin null; end;
begin null; end;
```

证明：PS/PL 也有复用链路。

## 阶段产出

1. 一张 SQL/PS/PL 对比表。
2. 一张虚拟表数据来源图。
3. 一份最终实验脚本。
4. 一句话总结：PS/PL/观测不是新主线，而是用同一套 cache 思想扩展和验证前面模型。

## 自查问题

1. PS cache 存 statement metadata，plan cache 存 physical plan，这句话能否举例说明？
2. PL cache 复用的对象和 SQL text plan 有什么不同？
3. 虚拟表为什么需要 ref cached object？
4. 你如何证明一次 SQL 是 hit 而不是 hard parse？
5. 如果只看源码不做虚拟表验证，会漏掉什么？
