# Answer 02：SQL Text 模式 Cache Hit 主链路（源码细化版）

## 1. 本阶段核心问题

本阶段回答：一次 SQL text 查询如何命中 plan cache。

一句话：

```text
SQL 被参数化成模板 key，
用 key 找到 Lib Cache 中的候选 plan，
通过有效性检查后，
ResultSet 绑定 cached plan + 本次 raw params，
跳过 hard parse。
```

例子：

```sql
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

可能共享：

```text
pc_key_.name_ = select * from t_pc where a = ?
raw_params_   = [1] 或 [2]
```

## 2. Hit 主调用链

关键源码：

| 步骤 | 源码位置 | 说明 |
|---|---|---|
| 文本 SQL 入口 | `src/sql/ob_sql.cpp:209` | `ObSql::stmt_query` |
| text SQL 主处理 | `src/sql/ob_sql.cpp:2735` | `ObSql::handle_text_query` |
| get plan + fill result | `src/sql/ob_sql.cpp:4154` | `ObSql::pc_get_plan_and_fill_result` |
| plan cache 调用封装 | `src/sql/ob_sql.cpp:4186` | `ObSql::pc_get_plan` |
| text/PS 分派 | `src/sql/ob_sql.cpp:4119` | `ObSql::execute_get_plan` |
| text get plan | `src/sql/plan_cache/ob_plan_cache.cpp:537` | `ObPlanCache::get_plan` |
| fast parser | `src/sql/plan_cache/ob_plan_cache.cpp:672` | `construct_fast_parser_result` |
| get cache obj | `src/sql/plan_cache/ob_plan_cache.cpp:1084` | `get_plan_cache` |
| key 查 node | `src/sql/plan_cache/ob_plan_cache.cpp:1295` | `get_value` |

时序：

```text
ObSql::handle_text_query
  -> pc_get_plan_and_fill_result
    -> pc_get_plan
      -> execute_get_plan
        -> ObPlanCache::get_plan
          -> construct_fast_parser_result
          -> get_plan_cache
            -> get_cache_obj
              -> get_value(key)
              -> ObPCVSet / ObPlanCacheValue / ObPlanSet match
          -> check_after_get_plan
    -> result.set_is_from_plan_cache(true)
    -> result.from_plan(plan, raw_params)
```

## 3. `handle_text_query`：hit/miss 的分叉点

位置：`src/sql/ob_sql.cpp:2735`

重点看三件事。

### 3.1 SQL 进入 plan cache 前先 trim

`handle_text_query` 会处理 trim 后的 SQL。这个动作的意义：

```text
'select * from t'
'  select * from t  '
```

不应因为首尾空格不同导致模板不同。

### 3.2 构造 `ObPlanCacheCtx`

这里会构造 text 模式的 plan cache 上下文，大致作用：

```text
ObPlanCacheCtx
  -> 记录 raw_sql
  -> 记录 PC_TEXT_MODE
  -> 关联 ObSqlCtx / ObExecContext
  -> 后续保存 fp_result_（key + raw_params）
```

### 3.3 调用 `pc_get_plan_and_fill_result`

如果 session 启用 plan cache，就尝试走 get plan。

结果有两种：

```text
result.is_from_plan_cache = true
  -> hit，后续跳过 hard parse

result.is_from_plan_cache = false
  -> miss，进入 handle_physical_plan
```

理解：`handle_text_query` 不是纯执行入口，而是 plan cache hit/miss 分叉点。

## 4. `pc_get_plan_and_fill_result`：拿 plan 并填 result

位置：`src/sql/ob_sql.cpp:4154`

该函数关注两个对象：

```text
ObCacheObjGuard guard
ObResultSet result
```

核心动作：

```text
pc_get_plan(pc_ctx, guard, get_plan_err, need_disconnect)
plan = guard.get_cache_obj()
result.set_is_from_plan_cache(true)
result.from_plan(*plan, pc_ctx.fp_result_.raw_params_)
```

### 4.1 `guard` 的意义

`guard` 不是普通指针包装。它保护 cached plan 生命周期：

```text
plan 被当前 SQL 使用期间
  -> ref count 保持
  -> flush/evict 不能直接物理释放
```

所以 hit 后拿到 plan，不是裸用 `ObPhysicalPlan*`。

### 4.2 `result.from_plan` 的意义

cached plan 是模板化的执行计划。执行本次 SQL 还需要本次参数：

```text
cached plan + raw_params -> 当前 ResultSet 可执行状态
```

所以 `raw_params_` 是 hit 路径不可缺少的一半。

## 5. `pc_get_plan`：命中统计和错误语义

位置：`src/sql/ob_sql.cpp:4186`

这个函数做四件事：

1. 从 session 拿 tenant 级 `ObPlanCache`。
2. 调 `execute_get_plan`。
3. 成功时更新 hit/access 统计。
4. 设置 `pc_ctx.sql_ctx_.plan_cache_hit_ = true`。

重点理解：

```text
plan cache miss 不等于 SQL 错误
```

如果 `get_plan` 返回 `OB_SQL_PC_NOT_EXIST` 或一些 cache 内部错误，通常会记录到 `get_plan_err`，然后让上层走 hard parse。

命中时：

```text
plan_cache->inc_hit_and_access_cnt()
pc_ctx.sql_ctx_.plan_cache_hit_ = true
```

未命中时通常：

```text
plan_cache->inc_access_cnt()
后续 hard parse
```

## 6. `execute_get_plan`：text 和 PS/PL 分派

位置：`src/sql/ob_sql.cpp:4119`

核心逻辑：

```text
if pc_ctx.mode_ == PC_PS_MODE or PC_PL_MODE:
    plan_cache.get_ps_plan(...)
else:
    plan_cache.get_plan(...)
```

Text SQL 走：

```text
guard.init(CLI_QUERY_HANDLE)
plan_cache.get_plan(allocator, pc_ctx, guard)
```

这告诉你：

- text query 的引用来源是 `CLI_QUERY_HANDLE`。
- PS/PL 会走不同 get 接口，但仍和 plan cache 结构有关。

## 7. `ObPlanCache::get_plan`：text cache 查询核心

位置：`src/sql/plan_cache/ob_plan_cache.cpp:537`

源码里的注释已经给出三步：

```text
1. fast parser 获取 param sql 及 raw params
2. 根据 param sql 获得 pcv set
3. 检查权限信息
```

拆开看：

### 7.1 参数化

```text
construct_fast_parser_result(allocator, pc_ctx, pc_ctx.raw_sql_, pc_ctx.fp_result_)
```

产出：

```text
pc_ctx.fp_result_.pc_key_      用于查 cache
pc_ctx.fp_result_.raw_params_  本次真实参数
```

### 7.2 查 cache

```text
get_plan_cache(pc_ctx, guard)
```

内部会：

```text
pc_ctx.key_ = &pc_ctx.fp_result_.pc_key_
get_cache_obj(ctx, key, guard)
```

### 7.3 校验返回对象

`get_plan` 会确认拿到的对象：

```text
guard.cache_obj_ != null
namespace == NS_CRSR
cast to ObPhysicalPlan
```

如果类型不对，说明 Lib Cache 返回了不该返回的对象，是异常。

## 8. `construct_fast_parser_result`：key 和参数如何产生

位置：`src/sql/plan_cache/ob_plan_cache.cpp:672`

它做这些事：

1. 从 session 获取 SQL mode、charset 等 parser 环境。
2. 先构造基础 plan cache key。
3. 如果 exact mode 开启，则 `pc_key_.name_ = raw_sql`。
4. 否则调用 `ObSqlParameterization::fast_parser`。
5. fast parser 生成参数化 SQL 和 raw params。
6. insert values batch 优化可能重建 raw params。

重点：

```text
pc_key_.name_ 和 raw_params_ 是一起生成的。
```

它们分别服务：

```text
pc_key_.name_ -> 找 plan
raw_params_   -> 执行 plan
```

## 9. `get_plan_cache` / `get_cache_obj`：从 key 到 plan

`get_plan_cache` 位置：`src/sql/plan_cache/ob_plan_cache.cpp:1084`

核心：

```text
pc_ctx.key_ = &(pc_ctx.fp_result_.pc_key_)
get_cache_obj(ctx, pc_ctx.key_, guard)
check_after_get_plan(...)
```

`get_cache_obj` 大致做：

```text
get_value(key, cache_node)
  -> 从 cache_key_node_map_ 找 node
cache_node->get_cache_obj(...)
  -> node 内部匹配 PCV/PlanSet
guard.cache_obj_ = plan
```

理解重点：

```text
key 只找到 node，不一定直接就是 plan。
node 里还有进一步匹配。
```

## 10. `check_after_get_plan`：hit 后为什么还要检查

位置：`src/sql/plan_cache/ob_plan_cache.cpp:459`

命中后还可能因为以下原因退化成 miss：

| 原因 | 解释 |
|---|---|
| schema 版本过期 | 表结构变了，旧 plan 不安全 |
| 统计信息过期 | CBO 选择可能不再合理 |
| UDR/规则版本变化 | 改写规则或依赖对象变了 |
| node invalid | cache node 被标记失效 |
| 锁冲突 | 并发修改中，本次不安全使用 |

这体现核心原则：

```text
宁可 miss 重新 hard parse，也不要错误复用。
```

## 11. `plan_cache_hit_` vs `is_from_plan_cache_`

| 字段 | 位置 | 用途 |
|---|---|---|
| `plan_cache_hit_` | `ObSqlCtx` | SQL 执行上下文、audit、统计 |
| `is_from_plan_cache_` | `ObResultSet` | 当前结果集是否来自 cached plan |

一般 hit 时都表现为 true，但它们服务不同层。

## 12. 一次命中的状态变化

| 阶段 | 字段/对象 | 状态 |
|---|---|---|
| fast parser | `pc_ctx.fp_result_.pc_key_` | 生成模板 key |
| fast parser | `pc_ctx.fp_result_.raw_params_` | 保存本次参数 |
| get cache | `pc_ctx.key_` | 指向 key |
| get object | `guard.cache_obj_` | 持有 plan |
| pc_get_plan | `plan_cache_hit_` | 设为 true |
| fill result | `result.is_from_plan_cache_` | 设为 true |
| fill result | `result.from_plan` | 装载 plan 与 params |

## 13. 验证方式

### 基础 hit

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

观察：

```sql
select tenant_id, access_count, hit_count, hit_rate
from oceanbase.GV$OB_PLAN_CACHE_STAT;
```

预期：第二次执行后 hit count 增长。

### 参数复用

```sql
select * from t_pc where a = 1;
select * from t_pc where a = 100;
```

理解重点：

```text
模板相同，raw params 不同。
```

### 禁用 plan cache

```sql
set ob_enable_plan_cache = 0;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

预期：不命中，hit count 不增长。

## 14. 本阶段参考答案

1. hit 主链路是 `handle_text_query -> pc_get_plan_and_fill_result -> pc_get_plan -> execute_get_plan -> ObPlanCache::get_plan`。
2. `ObPlanCacheCtx` 保存 raw SQL、mode、fast parser 结果、key、执行上下文。
3. `pc_key_.name_` 是参数化模板，`raw_params_` 是本次真实参数。
4. hit 后由 `ObCacheObjGuard` 持有 plan 引用。
5. `ObResultSet::from_plan` 把 cached plan 和 raw params 绑定到当前执行。
6. `check_after_get_plan` 防止 schema/统计/规则变化导致错误复用。
