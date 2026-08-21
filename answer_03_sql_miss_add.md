# Answer 03：Cache Miss、Hard Parse、Add Plan 主链路（源码细化版）

## 1. 本阶段核心问题

本阶段回答：plan cache 没命中后，OceanBase 如何兜底执行，并把新 plan 放回 cache。

核心结论：

```text
miss 是正常路径，不是错误。
miss 后 hard parse 得到当前 SQL 可执行 plan。
add plan 只是为了后续复用。
add 失败通常不影响当前 SQL 执行。
```

## 2. Miss/Add 主调用链

源码锚点：

| 步骤 | 源码位置 | 作用 |
|---|---|---|
| get plan | `src/sql/plan_cache/ob_plan_cache.cpp:537` | 尝试从 cache 拿 plan |
| get plan 封装 | `src/sql/ob_sql.cpp:4186` | `pc_get_plan` 处理错误与统计 |
| hard parse | `src/sql/ob_sql.cpp:5072` | `handle_physical_plan` |
| 是否 add | `src/sql/ob_sql.cpp:5002` | `need_add_plan` |
| add 封装 | `src/sql/ob_sql.cpp:4669` | `pc_add_plan` |
| PlanCache add | `src/sql/plan_cache/ob_plan_cache.cpp:1022` | `ObPlanCache::add_plan` |
| add 到 Lib Cache | `src/sql/plan_cache/ob_plan_cache.cpp:1053` | `add_plan_cache` |
| node/object 插入 | `src/sql/plan_cache/ob_plan_cache.cpp:1104` | `add_cache_obj` |

主流程：

```text
ObPlanCache::get_plan
  -> OB_SQL_PC_NOT_EXIST / OB_PC_LOCK_CONFLICT
    -> pc_get_plan 记录 get_plan_err，但不直接失败
      -> handle_text_query 判断 result 不是 from plan cache
        -> handle_physical_plan
          -> parser / resolver / optimizer / codegen
          -> need_add_plan
          -> pc_add_plan
            -> result.to_plan
            -> ObPlanCache::add_plan
              -> construct_plan_cache_key
              -> add_plan_cache
                -> add_cache_obj
```

## 3. miss 为什么不报错

`pc_get_plan` 的设计思想：

```text
plan cache 是优化，不是唯一执行路径。
```

所以：

```text
cache miss
  -> get_plan_err = OB_SQL_PC_NOT_EXIST
  -> ret 仍可恢复为 OB_SUCCESS
  -> 上层继续 hard parse
```

只有某些必须中断或重试的错误才会上抛，例如：

- `OB_EAGAIN`
- `OB_REACH_MAX_CONCURRENT_NUM`
- `OB_ERR_PROXY_REROUTE`
- `OB_NEED_SWITCH_CONSUMER_GROUP`

重点不是背全错误码，而是理解分类：

```text
可兜底错误 -> hard parse
不可兜底错误 -> 上抛
```

## 4. miss 来源有哪些

| 来源 | 解释 |
|---|---|
| 首次执行 | cache 没有对应 key/node/object |
| flush/evict 后执行 | 旧 plan 已不可见 |
| schema 变化 | `check_after_get_plan` 判断旧 plan 不安全 |
| 统计信息过期 | plan 可能需要重算 |
| sys var/context 变化 | key 变了 |
| 参数/约束不匹配 | key 命中后精筛失败 |
| lock conflict | 并发写锁导致本次不使用 cache |

理解：miss 不只是“map 查不到”，还包括“查到了但不安全”。

## 5. `handle_physical_plan`：hard parse 主入口

位置：`src/sql/ob_sql.cpp:5072`

它做的是常规 SQL 编译链路。plan cache 只关心其中两段：

```text
前半段：生成 ObPhysicalPlan
后半段：判断是否把 ObPhysicalPlan 加入 cache
```

可以这样读：

```text
handle_physical_plan
  -> parser/resolver/optimizer/codegen  得到当前 plan
  -> need_add_plan                      判断是否适合复用
  -> pc_add_plan                        尝试放入 plan cache
```

不需要在本阶段深挖 optimizer 如何选 join order、cost 等。

## 6. `need_add_plan`：哪些 plan 不应该 cache

位置：`src/sql/ob_sql.cpp:5002`

它回答：当前生成的 plan 是否适合给未来 SQL 复用。

典型否决原因：

| 原因 | 为什么不 cache |
|---|---|
| session 关闭 plan cache | 用户/会话明确禁用 |
| hint `use_plan_cache(none)` | SQL 级别禁用 |
| `should_add_plan_ == false` | 上下文不允许 |
| link table | 依赖外部/远端对象，复用风险高 |
| 特殊语句类型 | 不适合复用 |
| batch/multi stmt 特殊路径 | 参数化和执行语义复杂 |
| plan 太大 | 缓存收益不够，内存成本高 |
| 内存水位超限 | 保护 tenant 内存 |

你要理解的是：

```text
能执行 ≠ 适合缓存
```

## 7. `pc_add_plan`：把当前 plan 交给 plan cache

位置：`src/sql/ob_sql.cpp:4669`

重点动作：

```text
检查 hint / add 条件
result.to_plan
  -> 从 ResultSet 取出当前物理计划
  -> 填充 plan stat，如 sql_id、format_sql_id、constructed sql
按 mode 调用：
  -> text: ObPlanCache::add_plan
  -> ps/pl: add_ps_plan 等
```

关键点：

```text
pc_add_plan 是桥梁：从 SQL 编译执行框架进入 ObPlanCache。
```

## 8. `pc_add_plan` 的错误处理哲学

源码中很多 add 失败会被转成 `OB_SUCCESS`。

原因：

```text
当前 SQL 已经有可执行 plan。
cache add 失败只是后续不能复用。
```

典型处理：

| 错误/情况 | 源码语义 | 对用户 SQL |
|---|---|---|
| `OB_SQL_PC_PLAN_DUPLICATE` | 并发已加入 | 不失败 |
| `OB_REACH_MEMORY_LIMIT` | cache 内存满 | 不失败 |
| `OB_SQL_PC_PLAN_SIZE_LIMIT` | plan 太大 | 不失败 |
| not supported | 不支持 cache | 不失败 |
| lock conflict | 并发冲突 | 多数可退化 |

只有少数错误不能吞，例如影响当前执行正确性或调度语义的错误。

## 9. `ObPlanCache::add_plan`：PlanCache 层 add 入口

位置：`src/sql/plan_cache/ob_plan_cache.cpp:1022`

它做这些检查：

```text
plan 非空？
是否达到 plan cache memory limit？
单个 plan 是否过大？
构造 plan cache key
add_plan_cache
成功后 inc_mem_used
```

重点认识：

1. add 前先看内存。
2. add 阶段要构造和 get 阶段一致的 key。
3. 内存限制保护的是 cache，不是当前执行。

## 10. `add_plan_cache`：进入通用 add 逻辑

位置：`src/sql/plan_cache/ob_plan_cache.cpp:1053`

核心：

```text
pc_ctx.key_ = &pc_ctx.fp_result_.pc_key_
如果 regenerating_expired_plan_：先 remove_cache_node 旧 key
add_cache_obj(ctx, key, plan)
如果 OB_OLD_SCHEMA_VERSION 且允许 retry：重试
```

理解：

- 过期 plan 重新生成时，需要先移除旧 node。
- schema 旧版本 add 失败可能重试，不是无限死循环。

## 11. `add_cache_obj`：创建 node 或挂入已有 node

位置：`src/sql/plan_cache/ob_plan_cache.cpp:1104`

两条路径：

```text
key 对应 node 不存在
  -> create_node_and_add_cache_obj
  -> deep copy key
  -> set_refactored(cache_key_node_map_)

key 对应 node 已存在
  -> cache_node->add_cache_obj
  -> update_node_stat
  -> add_stat_for_cache_obj
```

并发竞争：

```text
线程 A 和 B 同时首次 add 同 key
  A set_refactored 成功
  B 遇到 OB_HASH_EXIST
  B 释放自己创建的 node
  B 重新 add 到已有 node
```

认识：并发 duplicate 是正常情况。

## 12. Add Plan 与 Lib Cache 结构关系

add 完后结构变成：

```text
cache_key_node_map_
  key(ObPlanCacheKey)
    -> node(ObPCVSet)
       -> value(ObPlanCacheValue)
          -> plan_set(ObPlanSet)
             -> plan(ObPhysicalPlan)
```

注意：不是把 plan 直接挂到 hash map value 上，而是挂到 node 内部结构里。

## 13. 验证实验

### 实验 1：首次执行 add

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

验证：

- 第一次 miss。
- 第一次走 `handle_physical_plan`。
- 第一次后走 `pc_add_plan`。
- 第二次有机会 hit。

### 实验 2：hint 禁 cache

```sql
select /*+ use_plan_cache(none) */ * from t_pc where a = 1;
select /*+ use_plan_cache(none) */ * from t_pc where a = 2;
```

验证：

- SQL 正常执行。
- 不 add plan。
- 第二次仍可能 hard parse。

### 实验 3：并发 add

多个连接同时执行：

```sql
select * from t_pc where a = 1;
```

观察：

- 多个线程可能都 hard parse。
- 一个 add 成功。
- 其他遇到 duplicate/hash exist。
- 用户 SQL 不失败。

## 14. 本阶段参考答案

1. miss 后不报错，因为 hard parse 是兜底路径。
2. `handle_physical_plan` 负责生成当前 SQL 的 physical plan。
3. `need_add_plan` 判断这个 plan 是否适合后续复用。
4. `pc_add_plan` 把 SQL 层的 plan 交给 `ObPlanCache`。
5. `ObPlanCache::add_plan` 做内存检查、key 构造和 Lib Cache 插入。
6. add 失败通常只影响后续复用，不影响当前 SQL。
7. 并发 duplicate 是正常竞争，不是功能错误。
