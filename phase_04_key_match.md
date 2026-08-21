# Phase 04：Key 构造、参数化、Plan Match

## 本阶段重点

理解 plan cache 最核心的问题：什么情况下“可以复用同一个 plan”。

核心模型：

```text
SQL 文本 -> 参数化模板 -> key 粗筛 -> value/plan 精筛 -> 可复用 plan
```

这阶段不是背参数化规则，而是理解为什么需要多层判断。

## 不必深挖

先不要追：

- fast parser 每种 token 规则。
- not-param 所有边界 case。
- plan match 每个表达式比较细节。
- distributed plan 的所有分支。
- SPM/baseline 细节。

只抓住“粗筛 + 精筛”模型。

## 核心问题

读完只要求回答：

1. `ObPlanCacheKey` 主要由哪些维度组成？
2. 为什么 SQL 文本不同也可能同 key？
3. 为什么 key 相同仍可能不能复用？
4. 参数化 SQL 和 raw params 分别解决什么问题？
5. `ObPCVSet`、`ObPlanCacheValue`、`ObPlanSet` 大致分工是什么？
6. schema/sys var/db/weak read/参数类型如何影响复用？

## 关键入口

只重点看：

1. `src/sql/plan_cache/ob_plan_cache_struct.h`
   - `ObPlanCacheKey`
   - `PlanCacheMode`

2. `src/sql/plan_cache/ob_plan_cache.cpp`
   - `construct_fast_parser_result`
   - `construct_plan_cache_key`

3. `src/sql/plan_cache/ob_sql_parameterization.cpp`
   - `fast_parser`
   - `parameterize_syntax_tree`

4. SQL match 三件套：
   - `src/sql/plan_cache/ob_pcv_set.*`
   - `src/sql/plan_cache/ob_plan_cache_value.*`
   - `src/sql/plan_cache/ob_plan_set.*`

## 理解模型

### 1. key 不是单纯 SQL 字符串

`ObPlanCacheKey` 至少要理解这些维度：

```text
参数化 SQL name_
db_id_
mode_
sys vars / config
weak read
namespace
```

这说明 plan 是否可复用，不只看 SQL 长相，还看执行上下文。

### 2. 参数化解决“文本不同但结构相同”

```sql
select * from t where a = 1;
select * from t where a = 2;
```

可以变成：

```text
select * from t where a = ?
```

常量进入：

```text
raw_params_ = [1] / [2]
```

### 3. key 命中只是粗筛

即使 key 相同，还要继续确认：

- schema version 是否仍有效。
- sys vars/config 是否兼容。
- 参数类型是否匹配。
- not-param 条件是否一致。
- table location/constraint 是否匹配。
- plan 是否过期。

所以流程是：

```text
ObPlanCacheKey
  -> ObPCVSet
    -> ObPlanCacheValue
      -> ObPlanSet
        -> ObPhysicalPlan
```

## 必须理解的点

### 1. `pc_key_.name_` 是复用入口，不是执行参数

执行参数存在 `raw_params_`。如果只保存模板，没有 raw params，就不知道本次用 1 还是 2。

### 2. exact mode 会降低复用

exact mode 下可能直接用原 SQL 做 key。这样 `a=1` 和 `a=2` 不再共享模板。

### 3. 上下文改变会导致 miss

即使 SQL 文本相同，下面变化也可能 miss：

- 切 database。
- 改 sql mode。
- 改影响计划的 sys var。
- 强读/弱读变化。
- schema 变化。
- 参数类型变化。

### 4. 多层结构是为了避免错误复用

错误复用 plan 比 miss 更危险。miss 只是多花优化成本，错误复用可能执行错或性能极差。

## 判断树

```text
SQL 能参数化？
  no -> 用原 SQL 或退化路径
  yes -> 得到模板 SQL + raw_params

key 相等？
  no -> miss
  yes -> 进入 node

上下文/约束/参数/schema 匹配？
  no -> miss 或生成新 value/plan
  yes -> 返回 plan
```

## 验证方式

### 实验 1：文本不同但同模板

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

理解：文本不同，但模板可能相同。

### 实验 2：同 SQL，不同 database

```sql
create database if not exists db1;
create database if not exists db2;
use db1;
select * from t_pc where a = 1;
use db2;
select * from t_pc where a = 1;
```

理解：`db_id_` 是 key/context 维度，可能不能复用。

### 实验 3：schema 改变

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
alter table t_pc add column c int;
select * from t_pc where a = 1;
```

理解：schema version 变化后，旧 plan 不能盲目复用。

### 实验 4：sys var 改变

```sql
set sql_mode = '';
select * from t_pc where a = 1;
set sql_mode = 'ANSI_QUOTES';
select * from t_pc where a = 1;
```

理解：影响解析/计划的 sys var 改变后可能 miss。

## 阶段产出

1. 一张 key 维度表。
2. 一张 `SQL -> template -> key -> value -> plan` 图。
3. 一张“为什么 key 相同仍可能 miss”的原因表。
4. 一句话总结：plan cache 复用不是字符串相等，而是“模板相同 + 上下文安全 + plan 仍有效”。

## 自查问题

1. `a=1` 和 `a=2` 为什么能复用？
2. 同 SQL 为什么换 database 可能不能复用？
3. key 相同为什么还需要 `ObPlanCacheValue`？
4. 错误复用 plan 比 miss 更危险在哪里？
5. exact mode 对复用率有什么影响？
