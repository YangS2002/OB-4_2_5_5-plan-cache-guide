# Phase 05：Lib Cache、生命周期、内存淘汰与 Flush

## 本阶段重点

理解 cached plan 如何安全存在、被引用、被 flush、被淘汰。

核心模型：

```text
cache object 被多个线程引用
flush/evict 只能先标记/移除可见性
真正释放要等引用归零
```

本阶段重点是设计认识，不是背每个 ref count 位置。

## 不必深挖

先不要追：

- 每个 handle 的所有调用点。
- 每个锁的所有加解锁路径。
- 淘汰算法所有排序细节。
- MemoryContext 分配细节。
- 所有 flush 命令参数。

只理解为什么不会释放正在执行的 plan。

## 核心问题

读完只要求回答：

1. Lib Cache 为什么抽象成 key/node/object？
2. `ObCacheObjGuard` 解决什么问题？
3. flush 后为什么正在执行的 SQL 不应该崩？
4. 逻辑删除和物理释放有什么区别？
5. ref count 大致在哪里起作用？
6. 内存水位如何决定是否 add/evict？
7. plan flush、PL flush、PS flush、lib cache flush 有什么不同？

## 关键入口

只重点看：

1. Lib Cache 注册/创建
   - `src/sql/plan_cache/ob_lib_cache_register.h/.cpp`
   - `src/sql/plan_cache/ob_lib_cache_key_creator.*`
   - `src/sql/plan_cache/ob_lib_cache_node_factory.*`
   - `src/sql/plan_cache/ob_cache_object_factory.*`

2. Lib Cache 三类对象
   - `src/sql/plan_cache/ob_i_lib_cache_key.h`
   - `src/sql/plan_cache/ob_i_lib_cache_node.h/.cpp`
   - `src/sql/plan_cache/ob_i_lib_cache_object.h/.cpp`

3. 引用与 guard
   - `src/sql/plan_cache/ob_pc_ref_handle.h/.cpp`
   - `ObCacheObjGuard`

4. 内存/淘汰/flush
   - `src/sql/plan_cache/ob_plan_cache.h/.cpp`
   - `src/observer/ob_rpc_processor_simple.cpp`

## 理解模型

### 1. key/node/object 分层

```text
Key：怎么找到一类缓存
Node：这一类缓存下如何组织/匹配对象
Object：真正被复用的对象，如 ObPhysicalPlan
```

SQL plan cache：

```text
ObPlanCacheKey -> ObPCVSet -> ObPhysicalPlan
```

PL cache：

```text
ObPLObjectKey -> ObPLObjectSet -> ObPLFunction/ObPLPackage
```

这就是为什么 SQL/PL 可以复用同一套 Lib Cache 框架。

### 2. 生命周期模型

```text
plan 加入 cache
  -> query hit，guard 持有引用
  -> flush/evict 到来
       先从可见结构移除或标记删除
       不立即释放仍被引用对象
  -> query 结束，guard 析构释放引用
  -> ref count 为 0
       真正 destroy
```

### 3. 内存水位模型

```text
低水位以下：正常
高水位以上：触发淘汰
超过限制：拒绝 add plan
```

理解重点：内存压力影响“是否继续缓存”，不应直接影响当前 SQL 已有执行计划。

## 必须理解的点

### 1. flush 不等于立刻 free

flush 让 plan 不再被新查询命中，但旧查询可能还在执行。必须靠 ref count/guard 延迟释放。

### 2. `ObCacheObjGuard` 是生命周期边界

只要查询正在使用 cached plan，guard 就应保证对象不被释放。

### 3. Lib Cache 提供统一生命周期管理

SQL、PL、TableAPI 不必各自实现一套对象管理。它们注册自己的 key/node/object 类型即可。

### 4. 淘汰和 add 是解耦的

内存压力大时可以拒绝 add，也可以后台淘汰。当前 SQL 不一定失败。

## 必须会画的图

### 生命周期图

```text
visible in cache
  -> hit and ref++
  -> flush/evict mark deleted
  -> invisible to new query
  -> old query done ref--
  -> ref == 0 destroy
```

### Flush 路由图

```text
ALTER SYSTEM FLUSH ...
  -> RPC processor
    -> PLAN      -> flush SQL plan
    -> PS        -> flush prepared stmt cache
    -> PL        -> flush PL object cache
    -> LIB_CACHE -> by namespace flush
```

## 验证方式

### 实验 1：执行中 flush

1. 执行一个长 SQL。
2. 另一个连接执行：

```sql
alter system flush plan cache;
```

预期：长 SQL 不应该因为 plan 被 flush 而崩溃。

理解：guard/ref count 保护正在执行的 plan。

### 实验 2：flush 后重新 hard parse

```sql
select * from t_pc where a = 1;
alter system flush plan cache;
select * from t_pc where a = 2;
```

预期：flush 后第二次重新 miss/add。

### 实验 3：观察内存/统计

```sql
select * from oceanbase.GV$OB_PLAN_CACHE_STAT;
```

观察：

- mem used。
- mem hold。
- access count。
- hit count。
- cache object 数量。

## 阶段产出

1. 一张 key/node/object 通用模型图。
2. 一张 flush + ref count 生命周期图。
3. 一张 plan/PS/PL/lib cache flush 区别表。
4. 一句话总结：plan cache 生命周期管理的核心是“可见性删除”和“对象物理释放”分离。

## 自查问题

1. flush 后为什么不能直接 free object？
2. guard 如果缺失，会有什么风险？
3. 为什么 Lib Cache 要抽象 namespace？
4. 内存满时为什么可以只是不 add plan？
5. plan flush 和 lib cache flush 有什么范围差别？
