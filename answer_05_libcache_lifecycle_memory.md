# Answer 05：Lib Cache、生命周期、内存淘汰与 Flush（源码细化版）

## 1. 本阶段核心问题

本阶段回答：cached plan 被命中后，如何安全存在、被引用、被 flush、被淘汰。

核心结论：

```text
flush/evict 不能直接释放正在执行的 plan。
OceanBase 通过 Lib Cache 分层、guard、ref count、逻辑删除、延迟释放保证安全。
```

## 2. Lib Cache 三层模型

通用模型：

```text
Key -> Node -> Cache Object
```

SQL plan cache：

```text
ObPlanCacheKey -> ObPCVSet -> ObPhysicalPlan
```

PL cache：

```text
ObPLObjectKey -> ObPLObjectSet -> ObPLFunction / ObPLPackage
```

源码接口：

| 接口 | 位置 | 作用 |
|---|---|---|
| `ObILibCacheKey` | `src/sql/plan_cache/ob_i_lib_cache_key.h` | key 抽象，提供 hash/equal/deep_copy |
| `ObILibCacheNode` | `src/sql/plan_cache/ob_i_lib_cache_node.h/.cpp` | node 抽象，维护对象列表、锁、统计、引用 |
| `ObILibCacheObject` | `src/sql/plan_cache/ob_i_lib_cache_object.h/.cpp` | object 抽象，维护 ref count、状态、tenant、namespace |

## 3. 为什么要三层

如果只有：

```text
key -> object
```

就很难处理：

- 同一 SQL 模板下多个上下文。
- 同一上下文下多个 candidate plan。
- node 级统计。
- node 级锁。
- object 级生命周期。

三层后职责更清楚：

```text
Key    决定第一层查找
Node   管理同 key 下的候选对象和匹配逻辑
Object 真正被执行/复用
```

## 4. Namespace 注册机制

位置：`src/sql/plan_cache/ob_lib_cache_register.h`

核心宏：

```text
LIB_CACHE_OBJ_DEF(ns, ns_name, ck_class, cn_class, co_class, label)
```

它生成：

```text
ObLibCacheNameSpace 枚举
CK_ALLOC / CN_ALLOC / CO_ALLOC 工厂数组
namespace name / label
```

重点理解：

```text
一个 namespace 注册一套 key/node/object 类型。
```

例子：

| Namespace | Key | Node | Object |
|---|---|---|---|
| `NS_CRSR` | `ObPlanCacheKey` | `ObPCVSet` | `ObPhysicalPlan` |
| `NS_PRCR` | `ObPLObjectKey` | `ObPLObjectSet` | `ObPLFunction` |
| `NS_SFC` | `ObPLObjectKey` | `ObPLObjectSet` | `ObPLFunction` |
| `NS_ANON` | `ObPLObjectKey` | `ObPLObjectSet` | `ObPLFunction` |
| `NS_PKG` | `ObPLObjectKey` | `ObPLObjectSet` | `ObPLPackage` |

这解释了 SQL/PL/TableAPI 复用同一套框架。

## 5. `ObPlanCache` 内部主要成员

位置：`src/sql/plan_cache/ob_plan_cache.h`

重点认识这些：

```text
cache_key_node_map_  key -> node
co_mgr_              cache object manager
cn_factory_          node factory
ck_creator_          key creator
pc_stat_             plan cache 统计
mem_used_            plan cache 自维护内存
```

读 `ObPlanCache` 不要把所有成员背下来，只要知道：

```text
它既负责查找，也负责对象生命周期、内存、flush、evict、统计。
```

## 6. Cache Object 状态

位置：`src/sql/plan_cache/ob_i_lib_cache_object.h`

状态大致有：

```text
ACTIVE       可用
ERASED       已擦除
MARK_ERASED  标记擦除
```

重要字段：

| 字段 | 作用 |
|---|---|
| `ref_count_` | 当前被多少地方引用 |
| `log_del_time_` | 逻辑删除时间 |
| `added_to_lc_` | 是否加入 Lib Cache |
| `obj_status_` | 当前状态 |

核心认识：状态和 ref count 一起决定对象能不能真正释放。

## 7. Object Manager 双 map

位置：`src/sql/plan_cache/ob_lib_cache_object_manager.h/.cpp`

两个 map：

```text
cache_obj_map_       活跃可见对象
alloc_cache_obj_map_ 所有已分配、尚未物理销毁的对象
```

为什么需要两个：

- 一个对象可能已经从 cache 可见结构中删除。
- 但它仍可能被正在执行的 SQL 引用。
- 这时不能丢失它，否则没法管理和释放。

所以：

```text
cache_obj_map_ 控制可见性
alloc_cache_obj_map_ 控制生命周期追踪
```

这是本阶段重点。

## 8. Guard 与 Ref Count

`ObCacheObjGuard` 是 RAII 保护对象。

模型：

```text
get cached plan
  -> guard 持有 object
  -> object ref_count++
query 结束
  -> guard 析构
  -> object ref_count--
```

没有 guard 的风险：

```text
query 正在执行 plan
flush/evict 释放 plan
query 访问悬挂指针
```

所以 guard 是 plan cache 线程安全和生命周期安全的关键。

## 9. Ref Handle：引用的身份

文件：`src/sql/plan_cache/ob_pc_ref_handle.h`

`CacheRefHandleID` 给引用计数加来源身份。

常见：

| Handle | 场景 |
|---|---|
| `CLI_QUERY_HANDLE` | text SQL 查询 |
| `PS_EXEC_HANDLE` | PS 执行 |
| `PLAN_GEN_HANDLE` | plan 生成 |
| `GV_SQL_HANDLE` | 虚拟表遍历 |
| PL/package 相关 handle | PL cache 引用 |

作用：

```text
不仅知道 ref_count 不为 0，
还知道是谁持有引用。
```

这对泄漏诊断重要。

## 10. Flush/Evict 的核心区别

### flush

人为或命令触发，目标是清掉某类 cache：

```text
alter system flush plan cache
alter system flush pl cache
alter system flush ps cache
```

### evict

系统因内存压力或定时任务自动淘汰：

```text
mem_hold > high watermark
  -> 选择 victim node/object
  -> evict
```

共同点：

```text
都不能直接释放仍被引用对象。
```

## 11. 逻辑删除 vs 物理释放

核心流程：

```text
flush/evict
  -> 从 cache 可见结构移除
  -> 标记 log_del_time
  -> 新查询不能再命中
  -> 老查询如果还持有 guard，可以继续使用
  -> guard 释放后 ref_count 归零
  -> before_cache_evicted
  -> destroy object
```

这就是为什么 flush 后正在执行的 SQL 不应该崩溃。

## 12. 内存水位模型

`ObPlanCache` 维护内存阈值：

```text
mem_limit  最大限制
mem_high   高水位，触发淘汰
mem_low    低水位，淘汰目标
```

理解图：

```text
mem_hold < low          正常
low <= mem_hold < high  有压力但不一定淘汰
mem_hold >= high        触发淘汰
mem_hold >= limit       拒绝 add plan
```

关键认识：

```text
内存超限通常让 plan 不进入 cache，
不代表当前 SQL 不能执行。
```

## 13. Flush 类型区别

入口：`src/observer/ob_rpc_processor_simple.cpp`

| 类型 | 范围 | 认识 |
|---|---|---|
| plan cache | SQL physical plan | 主要清 `NS_CRSR` |
| PS cache | prepared statement 相关 cache | metadata 和 ps 相关对象 |
| PL cache | PL compiled object | function/package/anonymous |
| lib cache | 按 namespace 清理 | 更通用、更底层 |

## 14. 为什么虚拟表遍历也要引用保护

虚拟表查询 plan cache 时，也是在遍历 cached object。

风险：

```text
虚拟表正在读 plan 信息
另一个线程 flush plan cache
plan 被释放
```

所以虚拟表遍历时也要 ref cached object，常见 handle 如 `GV_SQL_HANDLE`。

## 15. 验证实验

### 执行中 flush

1. 连接 A 执行长 SQL。
2. 连接 B 执行：

```sql
alter system flush plan cache;
```

预期：连接 A 不崩溃。

证明：flush 与物理释放分离。

### flush 后重新生成

```sql
select * from t_pc where a = 1;
alter system flush plan cache;
select * from t_pc where a = 2;
```

预期：flush 后第二条重新 miss/add。

### 观察统计

```sql
select * from oceanbase.GV$OB_PLAN_CACHE_STAT;
```

关注：

- `MEM_USED`
- `MEM_HOLD`
- `ACCESS_COUNT`
- `HIT_COUNT`
- ref handle 相关列

## 16. 本阶段参考答案

1. Lib Cache 抽象成 key/node/object，是为了把定位、匹配组织、真实对象生命周期分开。
2. `ObCacheObjGuard` 保证查询使用 cached object 时对象不被释放。
3. flush 后正在执行 SQL 不崩，是因为对象先逻辑删除，物理释放等 ref count 归零。
4. `cache_obj_map_` 管可见对象，`alloc_cache_obj_map_` 管所有未销毁对象。
5. 内存水位影响是否 add/evict，不应直接决定当前 SQL 成败。
6. plan/PS/PL/lib cache flush 清理范围不同，但生命周期保护思想相同。
