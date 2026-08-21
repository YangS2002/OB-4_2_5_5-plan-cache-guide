# Answer 04：Key 构造、参数化、Plan Match（源码细化版）

## 1. 本阶段核心问题

本阶段回答：OceanBase 如何判断“这条 SQL 能不能复用已有 plan”。

核心结论：

```text
Plan cache 复用不是 SQL 文本相等，
而是模板相同 + 上下文兼容 + 参数/schema/约束安全匹配 + plan 未过期。
```

主流程：

```text
raw SQL
  -> fast parser 参数化
  -> ObPlanCacheKey 粗筛
  -> ObPCVSet / ObPlanCacheValue / ObPlanSet 精筛
  -> ObPhysicalPlan
```

## 2. `ObPlanCacheKey`：第一层粗筛

位置：`src/sql/plan_cache/ob_plan_cache_struct.h:52`

`ObPlanCacheKey` 决定某个 SQL 是否能进入同一个 cache node。

关键字段：

| 字段 | 含义 | 为什么影响复用 |
|---|---|---|
| `name_` | 参数化 SQL 模板 | SQL 结构是否相同 |
| `key_id_` | PS stmt id / schema object id | PS/PL 定位对象 |
| `db_id_` | database id | 同名表可能解析到不同对象 |
| `sessid_` | session id | 临时表、session 隔离 |
| `mode_` | text/PS/PL | 不同执行模式不能混用 |
| `sys_vars_str_` | 系统变量串 | 影响解析/优化结果 |
| `config_str_` | 配置串 | 影响计划生成 |
| `is_weak_read_` | 弱读标记 | 强读/弱读执行语义不同 |
| `namespace_` | Lib Cache namespace | SQL/PL/TableAPI 分离 |
| `sys_var_config_hash_val_` | sys/config hash | 提升 hash 区分度 |

`hash()` 位于 `src/sql/plan_cache/ob_plan_cache_struct.h:117`。

`is_equal()` 位于 `src/sql/plan_cache/ob_plan_cache_struct.h:131`。

理解：

```text
同一 SQL 文本，如果 db_id/sys var/weak read/mode 不同，也可能不是同一个 key。
```

## 3. Key 构造：`construct_plan_cache_key`

位置：`src/sql/plan_cache/ob_plan_cache.cpp:2242`

它从 session 和调用上下文中填 key。

关注字段来源：

| key 字段 | 来源 |
|---|---|
| `db_id_` | session 当前 database |
| `namespace_` | 调用方传入，如 `NS_CRSR` |
| `sys_vars_str_` | session 中影响 plan 的 sys vars |
| `config_str_` | session/config 中影响 plan 的配置 |
| `sys_var_config_hash_val_` | sys vars + config hash |
| `is_weak_read_` | 协议弱读/SQL 上下文 |
| `enable_mysql_compatible_dates_` | session 兼容设置 |

`name_` 通常不在这里最终定型，而是在 fast parser 阶段由参数化 SQL 填入。

重点认识：

```text
get 阶段和 add 阶段必须构造出一致 key。
```

如果 get 用一个 key，add 用另一个 key，就会出现“明明 add 了，下一次还是 miss”。

## 4. 参数化：`construct_fast_parser_result`

位置：`src/sql/plan_cache/ob_plan_cache.cpp:672`

它是 text SQL 复用的关键。

流程：

```text
读取 session sql_mode / charset
创建基础 ObPlanCacheKey
判断 exact mode
  -> exact mode: key.name_ = raw_sql
  -> normal mode: 调 ObSqlParameterization::fast_parser
fast parser 输出：
  -> 参数化模板 SQL
  -> raw_params
处理 insert batch opt 等特殊优化
```

## 5. Fast Parser：模板和参数的分离

位置：`src/sql/plan_cache/ob_sql_parameterization.cpp:1751`

它把：

```sql
select * from t where a = 1 and b = 'x';
```

变成：

```text
pc_key_.name_ = select * from t where a = ? and b = ?
raw_params_   = [1, 'x']
```

职责分离：

| 数据 | 用途 |
|---|---|
| `pc_key_.name_` | 查找 cache 的模板 key |
| `raw_params_` | 当前 SQL 执行时真实参数 |

重点：

```text
模板用于复用，参数用于执行。
```

这就是为什么 `a=1` 和 `a=2` 能用同一个 plan，但执行结果不同。

## 6. exact mode：为什么会降低复用

exact mode 下，key name 可能直接是原 SQL。

对比：

```text
normal mode:
  a=1 -> a=?
  a=2 -> a=?
  可以同 key

exact mode:
  a=1 -> a=1
  a=2 -> a=2
  key 不同
```

认识：exact mode 更保守，复用率更低，但行为更精确。

## 7. 为什么 key 命中后还要 match

key 命中只说明：

```text
模板和一些上下文维度相同
```

但还不能保证 plan 一定可用。

还要检查：

- schema version 是否仍有效。
- 参数类型是否兼容。
- not-param 常量是否一致。
- constraint 是否匹配。
- table location 是否变化。
- statistics 是否过期。
- plan 状态是否有效。

因此后面有：

```text
ObPCVSet -> ObPlanCacheValue -> ObPlanSet
```

## 8. `ObPCVSet`：一个 key 下的 node

文件：`src/sql/plan_cache/ob_pcv_set.h/.cpp`

角色：

```text
ObPCVSet 是 NS_CRSR 的 node。
一个 ObPlanCacheKey 对应一个 ObPCVSet。
```

它主要做：

1. 管理这个 key 下的多个 `ObPlanCacheValue`。
2. 在 get 时遍历 value，找可匹配的 PCV。
3. 在 add 时把新 plan 放进对应 value/plan set。
4. 维护 node 级统计和锁。

理解：

```text
Key 找到 PCVSet，但 PCVSet 里可能还有多种上下文组合。
```

## 9. `ObPlanCacheValue`：同 key 下的上下文分组

文件：`src/sql/plan_cache/ob_plan_cache_value.h/.cpp`

角色：

```text
同一个 SQL 模板 key 下，按更细的执行上下文分组。
```

它关注：

- 不参与参数化的常量。
- schema 依赖。
- sys schema / tenant schema version。
- 参数约束。
- plan set。

为什么需要它：

```text
同一模板 SQL，不同上下文可能需要不同 plan。
```

## 10. `ObPlanSet`：同 PCV 下选择具体 plan

文件：`src/sql/plan_cache/ob_plan_set.h/.cpp`

角色：

```text
同一 PlanCacheValue 下可能有多个 ObPhysicalPlan。
ObPlanSet 负责按参数类型、约束、plan 状态选择一个。
```

它解决：

- 同模板下不同参数类型。
- local / distributed plan。
- constraint 匹配。
- plan candidate 选择。

## 11. `ObPlanMatchHelper`：匹配辅助

文件：`src/sql/plan_cache/ob_plan_match_helper.h/.cpp`

不用背所有函数，只记住它服务这些判断：

```text
参数类型是否匹配
schema/table 信息是否匹配
constraint 是否匹配
特殊常量是否匹配
```

## 12. 分层判断树

```text
raw SQL
  -> fast parser
     -> 得到 template + raw_params

ObPlanCacheKey equal?
  no -> miss
  yes -> ObPCVSet

ObPCVSet 中有匹配的 ObPlanCacheValue?
  no -> miss / add new value
  yes -> ObPlanSet

ObPlanSet 中有可用 ObPhysicalPlan?
  no -> miss / add new plan
  yes -> hit
```

## 13. key 相同仍 miss 的典型案例

| 场景 | 为什么 |
|---|---|
| 表结构变了 | schema version 不匹配 |
| 参数类型变了 | plan 可能依赖类型选择索引/表达式 |
| 某些常量不能参数化 | not-param 必须精确匹配 |
| 统计信息过期 | plan 选择可能要重算 |
| table location 变化 | 分布式计划位置不同 |
| weak read/strong read 变了 | 通常 key 层已隔离 |

## 14. 复用安全原则

```text
宁可 miss，也不能错用 plan。
```

因为：

```text
miss -> 多一次优化成本
错用 -> 可能错误执行，或性能严重退化
```

这就是多层 match 的设计动机。

## 15. 验证实验

### 文本不同但模板相同

```sql
alter system flush plan cache;
select * from t_pc where a = 1;
select * from t_pc where a = 2;
```

理解：模板相同，有机会复用。

### 同 SQL，不同 database

```sql
create database if not exists db1;
create database if not exists db2;
use db1;
select * from t_pc where a = 1;
use db2;
select * from t_pc where a = 1;
```

理解：`db_id_` 不同，不能盲目复用。

### schema 变化

```sql
select * from t_pc where a = 1;
alter table t_pc add column c int;
select * from t_pc where a = 1;
```

理解：旧 plan 可能失效。

### sys var 变化

```sql
set sql_mode = '';
select * from t_pc where a = 1;
set sql_mode = 'ANSI_QUOTES';
select * from t_pc where a = 1;
```

理解：影响计划的上下文变化可能 miss。

## 16. 本阶段参考答案

1. `ObPlanCacheKey` 由模板 SQL、db id、mode、sys vars、config、weak read、namespace 等维度组成。
2. SQL 文本不同可能同 key，因为常量被参数化。
3. key 相同仍可能 miss，因为 schema、参数类型、约束、plan 状态还要精筛。
4. `pc_key_.name_` 负责查找，`raw_params_` 负责执行。
5. `ObPCVSet` 是 key 下的 node，`ObPlanCacheValue` 是上下文分组，`ObPlanSet` 负责选具体 plan。
6. 设计目标是安全复用，而不是最大化复用。
