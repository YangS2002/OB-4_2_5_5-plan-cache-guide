# Phase 02：SQL Text 模式 Cache Hit 主链路

## 本阶段重点

理解一次 cache hit 是怎么发生的。不要背完整源码，只抓住：

```text
构造上下文 -> 生成 key -> 查 cache -> 拿到 plan -> 填回 result -> 执行
```

你要能解释第二次执行同类 SQL 为什么能跳过 parser/resolver/optimizer/codegen。

## 不必深挖

先不要细看：

- fast parser 的每种 token 处理。
- 所有权限检查分支。
- 所有错误码。
- 每个 ref count 加减点。
- plan match 的复杂条件。

这些后续阶段处理。

## 核心问题

读完只要求回答：

1. hit 主链路经过哪些函数？
2. `ObPlanCacheCtx` 在 hit 过程中保存什么？
3. `pc_key_.name_` 和 `raw_params_` 分别是什么？
4. hit 后哪个对象持有 plan？
5. hit 后 `ObResultSet` 如何得到 plan？
6. hit 后为什么还需要检查 plan 是否过期？

## 关键入口

只重点看 5 个函数：

1. `src/sql/ob_sql.cpp`
   - `ObSql::pc_get_plan_and_fill_result`
   - `ObSql::pc_get_plan`
   - `ObSql::execute_get_plan`

2. `src/sql/plan_cache/ob_plan_cache.cpp`
   - `ObPlanCache::get_plan`
   - `ObPlanCache::construct_fast_parser_result`
   - `ObPlanCache::get_plan_cache`

## 理解模型

### Hit 主链路

```text
ObSql::pc_get_plan_and_fill_result
  -> ObSql::pc_get_plan
    -> ObSql::execute_get_plan
      -> ObPlanCache::get_plan
        -> construct_fast_parser_result
           生成 pc_key_ 和 raw_params_
        -> get_plan_cache
           用 key 找 node/object
        -> check_after_get_plan
           检查 plan 是否还能用
  -> result.set_is_from_plan_cache(true)
  -> result.from_plan(plan, raw_params)
```

### 两个关键数据

```text
pc_key_.name_  = 参数化后的 SQL 模板
raw_params_    = 本次执行的真实常量参数
```

例子：

```sql
select * from t where a = 1;
select * from t where a = 2;
```

可能变成：

```text
pc_key_.name_ = select * from t where a = ?
raw_params_   = [1] 或 [2]
```

## 必须理解的点

### 1. Hit 不是直接用原 SQL 查 map

先通过 fast parser 把 SQL 变成可复用模板。模板相同，才有机会复用。

### 2. `ObCacheObjGuard` 是命中 plan 的保护壳

hit 后不是裸指针随便用，而是 guard 持有 cache object，避免执行期间被释放。

### 3. hit 后仍可能退化为 miss

原因：cached plan 可能已经不适合用了：

- schema 过期。
- statistics stale。
- plan 状态无效。
- JIT 或其他后置检查失败。

所以 `check_after_get_plan` 很重要。

### 4. `plan_cache_hit_` 与 `is_from_plan_cache_`

两者都描述命中，但位置不同：

- `plan_cache_hit_` 在 SQL context，用于执行统计/audit。
- `result.is_from_plan_cache_` 在 result set，用于当前执行流判断。

理解即可，不必死记所有赋值点。

## 验证方式

### 实验 1：基础 hit

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

验证：

- 第一次走 `handle_physical_plan`。
- 第二次走 `ObPlanCache::get_plan` 成功。
- `GV$OB_PLAN_CACHE_STAT.HIT_COUNT` 增长。

### 实验 2：参数复用

```sql
select * from t_pc where a = 1;
select * from t_pc where a = 100;
```

验证重点不是结果，而是理解：

```text
同一模板 + 不同 raw_params -> 复用 plan
```

### 实验 3：关闭 plan cache

```sql
set ob_enable_plan_cache = 0;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

验证：hit count 不增长或不走 cache。

## 阶段产出

1. 一张 hit 时序图。
2. 一张 `pc_key_.name_` vs `raw_params_` 示例表。
3. 一句话总结：hit 的本质是“同一参数化模板在当前上下文下找到仍有效的 cached physical plan”。

## 自查问题

1. 为什么 `a=1` 和 `a=2` 可能命中同一个 plan？
2. 为什么已经找到 plan 后还要 `check_after_get_plan`？
3. `raw_params_` 如果丢了会发生什么？
4. guard 解决的是什么问题？
