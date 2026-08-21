# Answer 06：PS、PL、观测与测试实验（源码细化版）

## 1. 本阶段核心问题

前面阶段讲的是 SQL text plan cache 主线。本阶段回答：

```text
PS 如何接入？
PL 如何接入？
用户如何观察？
如何用测试验证？
```

核心结论：

```text
PS cache 存 prepared statement metadata，执行时仍可能用 plan cache 存 physical plan。
PL cache 存 compiled PL object，并通过 Lib Cache namespace 复用通用框架。
虚拟表是观察 ObPlanCache 内部状态的窗口。
```

## 2. PS cache 和 plan cache 的关系

不要把两者混为一谈。

```text
PS cache:
  保存 prepare 后的 statement 信息
  例如 stmt_id、参数信息、依赖 schema、raw sql

Plan cache:
  保存可执行 physical plan
```

关系：

```text
PREPARE 生成 metadata
EXECUTE 根据 stmt_id 找 metadata
EXECUTE 再尝试从 plan cache 找 physical plan
```

## 3. PS 关键源码

| 文件 | 重点 |
|---|---|
| `src/sql/plan_cache/ob_prepare_stmt_struct.h` | `ObPsSqlKey`、`ObPsStmtItem`、`ObPsStmtInfo` |
| `src/sql/plan_cache/ob_ps_cache.h/.cpp` | `ObPsCache`，stmt_id map 和 stmt_info map |
| `src/sql/ob_sql.cpp` | `handle_ps_prepare`、`handle_ps_execute` |
| `src/sql/plan_cache/ob_plan_cache.h/.cpp` | `get_ps_plan`、`add_ps_plan` |

## 4. PS 关键结构

### `ObPsSqlKey`

用于标识一条 prepared SQL。

重要维度：

```text
db_id
inc_id
ps_sql
flag
```

其中 `inc_id` 常用于临时表等 session/schema 区分场景。

### `ObPsStmtItem`

维护：

```text
stmt_id
ps_key
ref_count
expired/evicted 状态
```

它类似 statement id 的索引节点。

### `ObPsStmtInfo`

保存真正的 statement metadata：

```text
stmt type
raw sql
no_param_sql
raw params
参数字段
返回列字段
schema 依赖 dep_objs
schema version
```

重点认识：PS prepare 之后，执行阶段不应该每次从 SQL 文本重新解析完整信息。

## 5. PS 流程

```text
COM_STMT_PREPARE
  -> ObSql::handle_ps_prepare
    -> ObPsCache::get_or_add_stmt_item
    -> ObPsCache::get_or_add_stmt_info
    -> 返回 stmt_id

COM_STMT_EXECUTE
  -> ObSql::handle_ps_execute
    -> ObPsCache::ref_stmt_info(stmt_id)
    -> 构造 PC_PS_MODE 的 ObPlanCacheCtx
    -> ObPlanCache::get_ps_plan
      -> hit: 使用 cached ps plan
      -> miss: hard parse / generate plan / add_ps_plan
```

重点：

```text
PS prepare 缓存 metadata。
PS execute 缓存/复用 physical plan。
```

## 6. PS 生命周期认识

PS cache 也需要引用计数。

原因：

```text
session 可能持有 stmt info
flush/evict 可能同时发生
execute 可能正在使用 stmt info
```

所以它同样需要：

- ref stmt info。
- deref stmt info。
- evict 时检查引用。
- schema 变化时失效。

## 7. PS 验证

```sql
prepare s from 'select * from t_pc where a = ?';
set @a = 1;
execute s using @a;
set @a = 2;
execute s using @a;
```

观察：

- 同一个 stmt id 复用。
- 第二次 execute 有机会命中 cached ps plan。
- schema alter 后可能失效重建。

Schema 失效：

```sql
prepare s from 'select * from t_pc where a = ?';
execute s using @a;
alter table t_pc add column d int;
execute s using @a;
```

## 8. PL cache 缓存什么

PL cache 缓存的是已编译 PL 对象，不是普通 SQL text plan。

包括：

- anonymous block。
- function。
- procedure。
- package。
- trigger。
- call statement。

## 9. PL namespace 与源码

| Namespace | Object | 含义 |
|---|---|---|
| `NS_PRCR` | `ObPLFunction` | procedure |
| `NS_SFC` | `ObPLFunction` | function |
| `NS_ANON` | `ObPLFunction` | anonymous block |
| `NS_TRGR` | `ObPLPackage` | trigger |
| `NS_PKG` | `ObPLPackage` | package |
| `NS_CALLSTMT` | `ObCallProcedureInfo` | call statement |

关键源码：

| 文件 | 重点 |
|---|---|
| `src/pl/pl_cache/ob_pl_cache.h/.cpp` | `ObPLObjectKey`、`ObPLObjectSet` |
| `src/pl/pl_cache/ob_pl_cache_object.h` | `ObPLFunction`、`ObPLPackage` |
| `src/pl/pl_cache/ob_pl_cache_mgr.h/.cpp` | `get_pl_cache`、`add_pl_cache`、flush/evict |
| `src/pl/ob_pl.cpp` | anonymous/function/procedure 使用 cache |
| `src/pl/ob_pl_package_manager.cpp` | package 使用 cache |

## 10. PL cache 流程

```text
PL execution
  -> 构造 ObPLObjectKey
  -> ObPLCacheMgr::get_pl_cache
    -> ObPlanCache::get_cache_obj
  -> hit:
       使用 compiled PL object
  -> miss:
       编译 PL object
       ObPLCacheMgr::add_pl_cache
         -> ObPlanCache::add_cache_obj
```

共同点：

```text
PL 和 SQL 都走 Lib Cache key/node/object 生命周期框架。
```

不同点：

```text
SQL text object = ObPhysicalPlan
PL object       = ObPLFunction / ObPLPackage
```

## 11. PL 重点理解

PL cache 的 key 通常和这些有关：

- object id / schema id。
- database id。
- PL object name。
- session/sys vars。
- schema version。
- package/function 状态。

不要深挖 compiler。只需要理解：

```text
miss 时编译 PL object，hit 时复用已编译对象。
```

## 12. PL 验证

```sql
alter system flush pl cache;
begin null; end;
begin null; end;
```

预期：第二次 anonymous block 有 cache 复用机会。

function/package 可通过：

1. 创建 function/package。
2. 调用两次。
3. 查询 plan cache 相关虚拟表。
4. flush PL cache 后再次调用。

## 13. 观测：虚拟表看什么

虚拟表是 `ObPlanCache` 内部状态的外部窗口。

重要文件：

| 文件 | 作用 |
|---|---|
| `src/observer/virtual_table/ob_all_plan_cache_stat.cpp` | plan cache 总体统计 |
| `src/observer/virtual_table/ob_gv_sql.cpp` | cached SQL/PL object 汇总 |
| `src/observer/virtual_table/ob_plan_cache_plan_explain.cpp` | cached plan explain |
| `src/observer/mysql/obmp_query.cpp` | query audit 命中信息 |
| `src/observer/mysql/obmp_stmt_execute.cpp` | PS execute audit 命中信息 |

## 14. `GV$OB_PLAN_CACHE_STAT`

它用于看总体状态：

```text
access_count
hit_count
hit_rate
mem_used
mem_hold
plan_num
ref handle counts
```

重点：

- hit/miss 验证看 `access_count`、`hit_count`。
- 内存压力看 `mem_used`、`mem_hold`。
- 引用问题看 ref handle 列。

## 15. `GV$OB_PLAN_CACHE_PLAN_STAT`

它用于看具体 cached object：

```text
plan_id
sql_id
statement
hit_count
executions
ps_stmt_id
```

理解：它可能展示 SQL physical plan，也可能涉及 PL object，取决于遍历分支。

## 16. `GV$OB_PLAN_CACHE_PLAN_EXPLAIN`

它按 `plan_id` 展示 cached plan 的执行计划算子树。

关键点：

```text
虚拟表遍历 cached object 时，也必须 ref object。
```

否则 flush/evict 并发时可能访问已释放对象。

## 17. 观测 SQL

```sql
select * from oceanbase.GV$OB_PLAN_CACHE_STAT;

select plan_id, sql_id, hit_count, statement
from oceanbase.GV$OB_PLAN_CACHE_PLAN_STAT
where statement like '%t_pc%';

select *
from oceanbase.GV$OB_PLAN_CACHE_PLAN_EXPLAIN
where plan_id = xxx;
```

观察：

- 第二次 SQL 后 `hit_count` 是否增加。
- flush 后 plan stat 是否减少。
- explain 是否能看到 cached plan。

## 18. 测试怎么读

不要逐行背测试。看测试覆盖什么行为。

目录：`unittest/sql/plan_cache/`

| 测试 | 关注 |
|---|---|
| `test_plan_cache.cpp` | add/get/ref 基础路径 |
| `test_plan_cache_value.cpp` | PCV 匹配/序列化 |
| `test_pcv_set.cpp` | PCVSet node 行为 |
| `test_plan_set.cpp` | 多 plan 管理 |
| `test_sql_parameterization.cpp` | 参数化 |
| `test_pc_perf.cpp` | 性能压力 |

MySQL test：`tools/deploy/mysql_test/test_suite/plan_cache/`

| 测试 | 关注 |
|---|---|
| `plan_cache_select_list.test` | 参数化和 hit_count 验证 |
| `plan_cache_multi_query.test` | multi query cache 行为 |

## 19. 最终闭环实验

### Text SQL：miss -> add -> hit

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

### 上下文变化导致 miss

```sql
set sql_mode = '';
select * from t_pc where a = 1;
set sql_mode = 'ANSI_QUOTES';
select * from t_pc where a = 1;
```

### schema 变化导致失效

```sql
select * from t_pc where a = 1;
alter table t_pc add column c int;
select * from t_pc where a = 1;
```

### flush 后重新生成

```sql
select * from t_pc where a = 1;
alter system flush plan cache;
select * from t_pc where a = 2;
```

### PS/PL 复用

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

## 20. 本阶段参考答案

1. PS cache 和 plan cache 不同：前者存 stmt metadata，后者存 physical plan。
2. PS execute 通过 stmt id 找 metadata，再从 plan cache 查 ps plan。
3. PL cache 存 compiled PL object，复用 Lib Cache 的 key/node/object 框架。
4. 虚拟表是观察 `ObPlanCache` 内部统计和对象的窗口。
5. 虚拟表遍历也要引用保护，避免 flush/evict 并发释放对象。
6. 测试重点看行为验证，不是背每个测试实现。
