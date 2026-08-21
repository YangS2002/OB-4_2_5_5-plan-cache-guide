# Answer 01：全局地图与概念边界（源码细化版）

## 1. 先抓主问题：Plan Cache 到底缓存什么

OceanBase plan cache 缓存的是**可复用的执行对象**，SQL text 场景下主要是 `ObPhysicalPlan`。

它不是简单的：

```text
SQL 原文 -> ObPhysicalPlan
```

而是：

```text
SQL 原文
  -> 参数化 SQL 模板
  -> 带 session/database/sys var/config/weak-read 等上下文的 key
  -> key 对应的 node
  -> node 内部继续按 schema/参数/约束/plan 状态匹配
  -> ObPhysicalPlan
```

为什么要这么复杂：

- 同一 SQL 文本，在不同 database 下解析对象可能不同。
- 同一 SQL 模板，不同 sys var/config 可能生成不同 plan。
- 同一模板，表结构或统计信息变化后旧 plan 可能不能继续用。
- 同一模板，不同参数类型或特殊常量可能不能复用。

所以 plan cache 的重点不是“缓存 map”，而是“安全复用判定”。

## 2. 源码入口总图

主入口在 `src/sql/ob_sql.cpp`：

| 入口 | 位置 | 作用 |
|---|---|---|
| `ObSql::stmt_query` | `src/sql/ob_sql.cpp:209` | 文本 SQL 请求入口之一 |
| `ObSql::handle_text_query` | `src/sql/ob_sql.cpp:2735` | text SQL 处理主函数，分出 hit/miss |
| `ObSql::pc_get_plan_and_fill_result` | `src/sql/ob_sql.cpp:4154` | 从 cache 获取 plan 并填回 result |
| `ObSql::pc_get_plan` | `src/sql/ob_sql.cpp:4186` | 调用 `ObPlanCache`，处理命中统计和错误码 |
| `ObSql::execute_get_plan` | `src/sql/ob_sql.cpp:4119` | 按 `PC_TEXT_MODE` / `PC_PS_MODE` 分派 |
| `ObSql::handle_physical_plan` | `src/sql/ob_sql.cpp:5072` | miss 后 hard parse 主入口 |
| `ObSql::pc_add_plan` | `src/sql/ob_sql.cpp:4669` | hard parse 后尝试加入 plan cache |

PlanCache 核心在 `src/sql/plan_cache/ob_plan_cache.cpp`：

| 函数 | 位置 | 作用 |
|---|---|---|
| `ObPlanCache::get_plan` | `src/sql/plan_cache/ob_plan_cache.cpp:537` | SQL text 模式 get plan 入口 |
| `ObPlanCache::construct_fast_parser_result` | `src/sql/plan_cache/ob_plan_cache.cpp:672` | 参数化 SQL，生成 key 和 raw params |
| `ObPlanCache::get_plan_cache` | `src/sql/plan_cache/ob_plan_cache.cpp:1084` | 用 key 找 cache object |
| `ObPlanCache::add_plan` | `src/sql/plan_cache/ob_plan_cache.cpp:1022` | SQL text 模式 add plan 入口 |
| `ObPlanCache::add_plan_cache` | `src/sql/plan_cache/ob_plan_cache.cpp:1053` | add plan 到通用 Lib Cache |
| `ObPlanCache::add_cache_obj` | `src/sql/plan_cache/ob_plan_cache.cpp:1104` | 创建/复用 node，挂载 object |

整体：

```text
SQL 到达
  -> handle_text_query
    -> pc_get_plan_and_fill_result
      -> pc_get_plan
        -> execute_get_plan
          -> ObPlanCache::get_plan
            -> construct_fast_parser_result
            -> get_plan_cache
              -> get_cache_obj
                -> get_value(key)
    -> if hit:
         result.from_plan(cached_plan, raw_params)
    -> if miss:
         handle_physical_plan
           -> parser/resolver/optimizer/codegen
           -> pc_add_plan
             -> ObPlanCache::add_plan
```

## 3. Lib Cache：为什么不是一个简单 map

`src/sql/plan_cache/ob_lib_cache_register.h:14` 注册 SQL physical plan cache：

```text
NS_CRSR -> ObPlanCacheKey -> ObPCVSet -> ObPhysicalPlan
```

通用抽象：

```text
ObILibCacheKey   定位一类缓存
ObILibCacheNode  管理这一类 key 下的多个对象和匹配逻辑
ObILibCacheObject 真正缓存的对象
```

SQL 场景对应：

```text
ObPlanCacheKey
  -> ObPCVSet
    -> ObPlanCacheValue
      -> ObPlanSet
        -> ObPhysicalPlan
```

这里最重要的认识：

- `ObPlanCacheKey` 只做第一层粗筛。
- `ObPCVSet` 是 `NS_CRSR` 的 node。
- `ObPlanCacheValue` 继续按上下文/约束分组。
- `ObPlanSet` 里可能有多个具体 plan。
- 最终才返回 `ObPhysicalPlan`。

## 4. Namespace 注册表怎么读

`ob_lib_cache_register.h` 通过 `LIB_CACHE_OBJ_DEF` 注册多种缓存类型。

你只需要看懂它把一个 namespace 映射成三类对象：

```text
namespace -> key class -> node class -> object class
```

常见 namespace：

| Namespace | Key | Node | Object | 说明 |
|---|---|---|---|---|
| `NS_CRSR` | `ObPlanCacheKey` | `ObPCVSet` | `ObPhysicalPlan` | SQL physical plan |
| `NS_PRCR` | `pl::ObPLObjectKey` | `pl::ObPLObjectSet` | `pl::ObPLFunction` | procedure |
| `NS_SFC` | `pl::ObPLObjectKey` | `pl::ObPLObjectSet` | `pl::ObPLFunction` | function |
| `NS_ANON` | `pl::ObPLObjectKey` | `pl::ObPLObjectSet` | `pl::ObPLFunction` | anonymous block |
| `NS_PKG` | `pl::ObPLObjectKey` | `pl::ObPLObjectSet` | `pl::ObPLPackage` | package |
| `NS_TABLEAPI` | table key | table node | table obj | TableAPI cache |
| `NS_SQLSTAT` | SQL stat key | SQL stat node | SQL stat obj | SQL stat cache |

这解释了为什么读 plan cache 时会遇到 PL、TableAPI、SQLSTAT：它们复用同一套 Lib Cache 框架。

## 5. `ObPlanCacheKey` 的源码认识

位置：`src/sql/plan_cache/ob_plan_cache_struct.h:52`

关键字段：

| 字段 | 源码意义 | 阅读重点 |
|---|---|---|
| `name_` | 参数化 SQL 模板 | 最重要，类似 `select ... where a=?` |
| `key_id_` | PS stmt id 或 schema object id | PS/PL 使用更多 |
| `db_id_` | 当前 database id | 切库可能导致 miss |
| `sessid_` | session id | 临时表等场景隔离 |
| `mode_` | `PC_TEXT_MODE` / `PC_PS_MODE` / `PC_PL_MODE` | 不同模式不共享 |
| `sys_vars_str_` | 影响计划的 session sys vars | sys var 变化可导致 miss |
| `config_str_` | 影响计划的 config | config 变化可导致 miss |
| `is_weak_read_` | 弱读标记 | 强弱读分离 |
| `namespace_` | `NS_CRSR` 等 | SQL/PL 分离 |
| `sys_var_config_hash_val_` | sys vars/config hash | 提升 hash 区分度 |

`hash()` 位于 `src/sql/plan_cache/ob_plan_cache_struct.h:117`。

`is_equal()` 位于 `src/sql/plan_cache/ob_plan_cache_struct.h:131`。

重点：`is_equal()` 是能否落入同一个 key 的最终判定，不只是比较 SQL 字符串。

## 6. `ObPlanCacheCtx` 的角色

位置：`src/sql/plan_cache/ob_plan_cache_struct.h:361`

它是一次 SQL get/add plan 的上下文容器。

它把这些信息串起来：

```text
raw_sql_              原始 SQL
mode_                 text/ps/pl
sql_ctx_              SQL 执行上下文
exec_ctx_             执行上下文
fp_result_            fast parser 结果
key_                  当前 Lib Cache 查找 key
not_param_info_       不能参数化的常量信息
neg_param_index_      负数参数信息
```

理解方式：

```text
ObSqlCtx 是 SQL 执行上下文
ObPlanCacheCtx 是 plan cache 操作上下文
ObFastParserResult 是 key/params 生成结果
```

## 7. 从整体看 hit/miss/add

### Hit

```text
handle_text_query
  -> pc_get_plan_and_fill_result
    -> ObPlanCache::get_plan
      -> fast parser 生成 key
      -> Lib Cache 查 object
      -> check_after_get_plan
    -> result.from_plan
```

### Miss

```text
ObPlanCache::get_plan 返回 OB_SQL_PC_NOT_EXIST
  -> handle_physical_plan
    -> parser/resolver/optimizer/codegen
```

### Add

```text
hard parse 得到 ObPhysicalPlan
  -> need_add_plan 判断
  -> pc_add_plan
  -> ObPlanCache::add_plan
  -> add_cache_obj
```

## 8. 你需要形成的“全局答案”

Plan cache 是：

```text
通过参数化 SQL 模板和执行上下文，查找可安全复用的物理计划；
miss 后 hard parse 兜底；
hard parse 后再尝试把 plan 放回 cache；
Lib Cache 负责 key/node/object、引用计数、flush/evict 等通用能力。
```

不是：

```text
把 SQL 原文作为 key 直接映射到 plan。
```

## 9. 验证方式

```sql
alter system flush plan cache;
create table if not exists t_pc(a int primary key, b int);
select * from t_pc where a = 1;
select * from t_pc where a = 2;
select tenant_id, access_count, hit_count, hit_rate
from oceanbase.GV$OB_PLAN_CACHE_STAT;
```

判断：

- 第一次 SQL 通常 miss。
- 第二次同模板 SQL 有机会 hit。
- `access_count` 增长说明访问了 plan cache。
- `hit_count` 增长说明命中了 plan cache。

## 10. 本阶段参考答案

1. Plan cache 解决重复 hard parse 成本问题。
2. SQL text get plan 主入口是 `ObPlanCache::get_plan`。
3. miss 后 hard parse 主入口是 `ObSql::handle_physical_plan`。
4. add plan 主入口是 `ObPlanCache::add_plan`。
5. `NS_CRSR` 对应 SQL physical plan，映射为 `ObPlanCacheKey -> ObPCVSet -> ObPhysicalPlan`。
6. PS/PL 出现，是因为它们复用 Lib Cache 的 namespace/key/node/object 框架。
