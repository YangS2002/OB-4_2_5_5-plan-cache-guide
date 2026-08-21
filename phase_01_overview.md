# Phase 01：全局地图与概念边界

## 本阶段重点

先回答一个问题：OceanBase 为什么需要 plan cache，它在 SQL 执行链路中处于什么位置？

本阶段只建立地图，不深挖实现。你要知道后续该沿哪几条线读：

1. SQL 请求进来后，什么时候尝试 get plan。
2. miss 后，什么时候 hard parse 并 add plan。
3. plan cache 底层不是单个 map，而是 Lib Cache 的 key/node/object 模型。
4. SQL、PS、PL 都在不同程度上复用这套框架。
5. 虚拟表和 flush 是理解 plan cache 行为的验证入口。

## 不必深挖

本阶段先不要追：

- 参数化细节。
- plan match 的所有条件。
- ref count 每个加减位置。
- PL package/function 的复杂失效逻辑。
- 所有虚拟表字段来源。

这些后面阶段再看。

## 核心问题

读完本阶段，只要求能回答：

1. plan cache 要解决什么问题？
2. SQL text 模式 get plan 的主入口在哪里？
3. miss 后 hard parse 的主入口在哪里？
4. add plan 的主入口在哪里？
5. `NS_CRSR`、`ObPlanCacheKey`、`ObPCVSet`、`ObPhysicalPlan` 之间是什么关系？
6. 为什么 PS/PL 也会出现在 plan cache 讨论里？

## 关键入口

只先看这些：

- `src/sql/ob_sql.cpp`
  - `ObSql::handle_text_query`
  - `ObSql::pc_get_plan_and_fill_result`
  - `ObSql::handle_physical_plan`
  - `ObSql::pc_add_plan`

- `src/sql/plan_cache/ob_plan_cache.h/.cpp`
  - `ObPlanCache::get_plan`
  - `ObPlanCache::add_plan`

- `src/sql/plan_cache/ob_lib_cache_register.h`
  - `LIB_CACHE_OBJ_DEF`
  - `NS_CRSR`

## 理解模型

### 1. 主链路

```text
SQL 到达
  -> 判断是否尝试 plan cache
  -> get plan
       hit  -> 使用 cached ObPhysicalPlan
       miss -> hard parse 生成 ObPhysicalPlan
               -> 如果适合复用，则 add plan
```

### 2. Lib Cache 模型

```text
Key -> Node -> Cache Object
```

SQL plan cache 具体化后：

```text
ObPlanCacheKey -> ObPCVSet -> ObPhysicalPlan
```

这张图先记住即可。后面再解释为什么 key 后面还有 node、value、plan set。

## 必须理解的点

### 1. Plan cache 不是“SQL 文本到 plan 的简单 map”

因为同一条可参数化 SQL 可能对应多个上下文：

- 不同 database。
- 不同 sys var。
- 不同 schema version。
- 不同参数类型。
- 不同弱读/强读。

所以它不是：

```text
sql string -> plan
```

而更接近：

```text
parameterized sql + context -> candidate plans -> match one plan
```

### 2. `ObPlanCache` 是 tenant 级服务对象

重点认识：它管理 get/add/evict/flush/memory/ref，不只是查询接口。

### 3. Lib Cache 是通用抽象

`NS_CRSR` 是 SQL physical plan。PL、TableAPI、SQLSTAT 也可以注册不同 namespace，复用 key/node/object 创建、引用、淘汰框架。

## 验证方式

执行：

```sql
alter system flush plan cache;
create table if not exists t_pc(a int primary key, b int);
select * from t_pc where a = 1;
select * from t_pc where a = 2;
select tenant_id, access_count, hit_count, hit_rate
from oceanbase.GV$OB_PLAN_CACHE_STAT;
```

观察：

- 第一次通常 miss。
- 第二次可能 hit。
- `GV$OB_PLAN_CACHE_STAT` 中 access/hit 变化。

## 阶段产出

1. 一张总链路图。
2. 一张 `ObPlanCacheKey -> ObPCVSet -> ObPhysicalPlan` 图。
3. 一句话总结：plan cache 是“按参数化 SQL 和上下文查找可复用物理计划”的机制。
