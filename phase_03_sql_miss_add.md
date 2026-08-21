# Phase 03：Cache Miss、Hard Parse、Add Plan 主链路

## 本阶段重点

理解第一次执行 miss 后，OceanBase 如何生成 plan，并决定是否放进 cache。

核心模型：

```text
get plan 失败 -> hard parse -> 生成 physical plan -> 判断是否值得 cache -> add plan
```

重点不是 optimizer 怎么生成 plan，而是 plan cache 如何衔接 hard parse。

## 不必深挖

先不要深入：

- parser/resolver/optimizer/codegen 的具体实现。
- CBO 代价模型。
- SPM/outline 全部分支。
- 所有 add 失败错误码。
- 并发 add 的每个锁细节。

本阶段只关注 plan cache 边界。

## 核心问题

读完只要求回答：

1. miss 后为什么不报错，而是 hard parse？
2. hard parse 入口在哪里？
3. 生成的 plan 在哪里决定是否加入 cache？
4. 哪些典型场景不 add plan？
5. add plan 最终如何进入 `ObPlanCache`？
6. add 失败为什么通常不影响当前 SQL？

## 关键入口

只重点看：

1. `src/sql/ob_sql.cpp`
   - `ObSql::handle_physical_plan`
   - `ObSql::need_add_plan`
   - `ObSql::pc_add_plan`

2. `src/sql/plan_cache/ob_plan_cache.cpp`
   - `ObPlanCache::add_plan`
   - `ObPlanCache::add_plan_cache`
   - `ObPlanCache::add_cache_obj`

## 理解模型

### Miss/Add 主链路

```text
ObPlanCache::get_plan
  -> OB_SQL_PC_NOT_EXIST
    -> ObSql::handle_physical_plan
      -> parser / resolver / optimizer / codegen
      -> need_add_plan
      -> pc_add_plan
        -> result.to_plan
        -> ObPlanCache::add_plan
          -> construct_plan_cache_key
          -> add_plan_cache
            -> add_cache_obj
```

### add plan 要理解的边界

```text
当前 SQL 执行需要 plan
plan cache 只是让后续 SQL 复用这个 plan
```

所以 add plan 失败，大多数情况下不该影响当前 SQL 执行。当前 SQL 已经 hard parse 得到 plan，可以继续执行。

## 必须理解的点

### 1. miss 是正常路径

第一次执行、新 schema、新参数上下文、新 sys var，都可能 miss。miss 不是错误，而是触发 hard parse。

### 2. 不是所有 plan 都值得 cache

常见不 add 场景：

- session 关闭 plan cache。
- hint 禁用 plan cache。
- plan 太大。
- 内存压力大。
- 语句类型不适合 cache。
- link table / 特殊执行路径。
- schema 状态不稳定。

### 3. add plan 需要重新构造一致的 key

get 阶段用 key 查，add 阶段也必须用一致 key 放进去。否则第二次执行找不到。

### 4. 并发 duplicate 是正常现象

多个线程同时首次执行同一 SQL，可能都 miss 并 hard parse。只有一个 add 成功，其它看到 duplicate 不应报用户错误。

## Add 错误只记这几类

| 类型 | 含义 | 认识 |
|---|---|---|
| memory limit | plan cache 内存满 | 不 cache，不影响当前执行 |
| plan too large | 单 plan 太大 | 不值得 cache |
| duplicate | 并发已有 plan | 正常竞争 |
| lock conflict | node 锁冲突 | 可退化执行 |
| old schema | schema 版本问题 | 可能 retry |

## 验证方式

### 实验 1：首次 miss + add

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
```

观察：

- 进入 `handle_physical_plan`。
- 进入 `pc_add_plan`。
- 后续同类 SQL hit。

### 实验 2：hint 禁用 cache

```sql
select /*+ use_plan_cache(none) */ * from t_pc where a = 1;
select /*+ use_plan_cache(none) */ * from t_pc where a = 2;
```

理解：

```text
hard parse 仍会成功，但不会为后续复用 add plan
```

### 实验 3：并发首次执行

多个连接同时执行：

```sql
select * from t_pc where a = 1;
```

观察：

- 可能多个线程 hard parse。
- 只有部分 add 成功。
- 用户 SQL 不应失败。

## 阶段产出

1. 一张 miss/add 时序图。
2. 一张“不 add plan 场景”表。
3. 一句话总结：miss 后 hard parse 是正常兜底，add plan 是为未来复用服务，不应绑死当前执行成功与否。

## 自查问题

1. miss 和真正错误有什么区别？
2. 为什么 add 失败多数情况下不能影响用户 SQL？
3. 为什么并发 add duplicate 是正常的？
4. `need_add_plan` 如果判断过宽会有什么风险？
5. get/add key 不一致会导致什么问题？
