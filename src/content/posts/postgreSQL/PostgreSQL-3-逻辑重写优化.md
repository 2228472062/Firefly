---
title: 第三章-逻辑重写优化
published: 2026-06-15 02:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
## 章节概览

本章分析 PostgreSQL 查询优化器中的**逻辑重写优化**阶段。逻辑重写优化是在查询树上进行等价变换，改造后仍然是一棵查询树（与后续的逻辑分解优化不同，后者会将查询树打散重建）。

**核心目标**：减少查询层次、消除外连接操作、减少冗余操作。

---

## 3.1 通用表达式（CTE / WITH 语句）

**定义**：通用表达式对应 WITH 语句，作用类似子查询，但具有"一次求值，多次使用"的特点。

**关键特性**：

- 不做提升优化（因为可能被多次引用）
- 支持递归通用表达式（通过 `UNION ALL` 实现递归）
- 通过给子查询生成子 Plan 实现，复用父 Plan 的优化代码

**示例**：

```sql
-- 普通 CTE
WITH CTE AS (SELECT * FROM STUDENT)
SELECT sname FROM CTE;

-- 递归 CTE
WITH RECURSIVE INC(VAL) AS (
  SELECT 1
  UNION ALL
  SELECT INC.VAL + 1 FROM INC WHERE INC.VAL < 10
)
SELECT * FROM INC;
```

---

## 3.2 子查询提升

### 3.2.0 子查询 vs 子连接

| 区分方式 | 子查询 (SubQuery)          | 子连接 (SubLink)                 |
| -------- | -------------------------- | -------------------------------- |
| 位置     | FROM 关键字后（范围表）    | WHERE/ON 约束条件中或投影中      |
| 存储     | RangeTblEntry 中的子查询树 | SubLink 结构体                   |
| 特点     | 独立执行计划               | 伴随 ANY/ALL/IN/EXISTS/SOME 谓词 |

### 3.2.1 提升子连接

#### SubLink 结构体

```c
typedef struct SubLink {
  SubLinkType subLinkType;  // EXISTS_SUBLINK / ALL_SUBLINK / ANY_SUBLINK
  int subLinkId;            // 编号
  Node *testexpr;           // 操作符表达式
  List *operName;           // 操作符
  Node *subselect;          // 子句产生的查询树
  int location;             // SQL 中的位置
} SubLink;
```

#### 谓词与子连接类型对照表

| 谓词         | 形式              | 提升后结果                   |
| ------------ | ----------------- | ---------------------------- |
| [NOT] IN     | LH [NOT] IN EXPR  | [反]半连接（Anti-）Semi Join |
| ANY/SOME     | LH OP ANY EXPR    | 半连接（Semi Join）          |
| [NOT] EXISTS | [NOT] EXISTS EXPR | [反]半连接（Anti-）Semi Join |

---

#### 3.2.1.1 ANY 类型子连接提升

**提升条件**（5项）：

1. 左操作数必须在 `available_rels` 中
2. 必须是**非相关子连接**（不能引用父查询列属性）
3. `testexpr` 中必须引用上一层的列（否则无法构成连接关系）
4. `testexpr` 中必须有 Var（列属性）
5. `testexpr` 中不能含有易失性函数（VOLATILE）

**提升流程**（5步骤）：

1. 将 `SubLink->subselect` 转换为 SubQuery 类型的 RangeTblEntry
2. 生成新的 RangeTblRef 节点
3. 提取子查询投影列，生成新 Var
4. 用新 Var 替换 `testexpr` 中的 Param 变量（`convert_testexpr` 函数）
5. 生成新的 JoinExpr（Semi Join），LHS 端由上层函数填充

**提升结果**：子连接先变为子查询（在 FROM 后），后续可能再被提升为连接。

**关键函数**：`pull_up_sublinks` → `pull_up_sublinks_qual_recurse` → `convert_ANY_sublink_to_join`

---

#### 3.2.1.2 EXISTS 类型子连接提升

**提升条件**（5项）：

1. 子句中不能包含通用表达式（CTE）
2. 子句必须是"简单"的（无集合操作、聚集操作、HAVING、窗口函数、LIMIT、FOR UPDATE 等）
3. 约束条件中必须包含上层的列属性（Var）——非相关的 EXISTS 子连接不提升
4. 除约束条件外的其他表达式不能引用上层列属性
5. 约束条件中不能含有易失性函数

**"简单"判断**（`simplify_EXISTS_query` 函数）：

```c
if (query->commandType != CMD_SELECT ||
    query->setOperations ||
    query->hasAggs ||
    query->groupingSets ||
    query->hasWindowFuncs ||
    query->hasTargetSRFs ||
    query->hasModifyingCTE ||
    query->havingQual ||
    query->limitOffset ||
    query->limitCount ||
    query->rowMarks)
  return false;
```

**提升流程**：

1. 保存子查询的 `FromExpr->quals` 为 `whereClause`
2. 调整 Var 的 `varno`（rtindex 增加上层 rtable 长度）
3. 调整引用上层表的 Var 的 `varlevelsup`（从 1 改为 0）
4. 将子查询的 rtable 附加到上层父查询的 rtable
5. 创建新的 jointree，用子查询的 jointree 作为 RHS，`whereClause` 作为 quals

**提升结果**：EXISTS 子连接会彻底提升为表与表的直接连接（不同于 ANY 类型先变子查询）。

**关键函数**：`convert_EXISTS_sublink_to_join`

---

#### 连接类型与子连接提升的关系

| 连接类型 | 提升情况                                     |
| -------- | -------------------------------------------- |
| 内连接   | 左操作数可以是 LHS 或 RHS 的列属性，均可提升 |
| 左连接   | 左操作数只能是 RHS 的列属性才能提升          |
| 右连接   | 左操作数只能是 LHS 的列属性才能提升          |
| 全连接   | 子连接不能提升                               |

---

### 3.2.2 提升子查询

#### RTE_SUBQUERY(SIMPLE) 提升

**提升条件**（6项）：
a) 必须是 SELECT 查询语句
b) `setOperations` 必须是 NULL（否则交给 UNION ALL 处理）
c) 不能包含聚集操作、窗口函数、GROUP、HAVING、ORDER BY、DISTINCT、LIMIT、FOR UPDATE、CTE
d) 无范围表时需满足特定条件
e) 不能有 LATERAL 引用更上层的表
f) 不能有易失性函数

**提升流程**：

1. 调整子查询中 Var 的 `varno`（`OffsetVarNodes`）
2. 调整 `varlevelsup`（`IncrementVarSublevelsUp`）
3. 用子查询投影列替换父查询中的 Var（`pullup_replace_vars`）
4. 将子查询的 rtable 附加到父查询的 rtable

**关键函数**：`pull_up_subqueries` → `pull_up_subqueries_recurse` → `pull_up_simple_subquery`

#### RTE_SUBQUERY(UNION) 提升

**提升条件**（`is_simple_union_all` 函数判断）：
a) 必须是 SELECT 查询语句
b) `setOperations` 必须有值，且操作符必须是 UNION ALL
c) 不能有 ORDER BY、LIMIT/OFFSET
d) SetOperationStmt 中的子语句必须包含相同类型的投影列

**提升目的**：将三个层次变成两个层次（子子查询提升为 Append Relation）。

**关键函数**：`pull_up_simple_union_all` → `pull_up_union_leaf_queries`

#### RTE_VALUES 提升

**提升条件**：

1. 不能是外连接下的语句
2. 不能是 UNION ALL 集合操作下的子句
3. RTE_VALUES 中只能有一个"元组"
4. VALUES 指定的值中不能含有易失性函数或返回集合的表达式
5. 必须只能有一个 RTE_VALUES

**提升流程**：

1. 将 VALUES 中的常量组织成 TargetEntry 形式
2. 用新 Var 替换所有引用 VALUES 的变量

**提升结果**：RTE_VALUES 消除，变成空 FROM 的子查询，可能进一步提升。

---

## 3.3 UNION ALL 优化

**目标**：将顶层的 UNION ALL 集合操作转换为 AppendRelInfo 形式。

**原理**：UNION ALL 和继承表效果相似，都使用 AppendRelInfo 结构体封装。

**函数**：`flatten_simple_union_all`

---

## 3.4 展开继承表

**继承表机制**：

- 父表在 `PG_CLASS` 中的 `relhassubclass` 值为 true
- 父表和子表的继承关系记录在 `PG_INHERITS` 系统表中

**展开流程**（`expand_inherited_tables` 函数）：

1. 遍历 `Query->rtable`，检查 `has_subclass`
2. 调用 `find_all_inheritors` 查找所有子表
3. 为每个子表生成新的 RangeTblEntry，附加到 `Query->rtable`
4. 生成 AppendRelInfo，附加到 `PlannerInfo->append_rel_list`
5. 如果父表有 rowMarks（FOR UPDATE），给子表也生成

---

## 3.5 预处理表达式

### 3.5.1 连接 Var 的溯源（flatten_join_alias_vars）

**问题**：连接操作的投影基于连接结果生成 Var，需要替换为实际表的 Var。

**示例**：`sc.cname`（varno=4）→ `COURSE.cname`（varno=2）

**实现**：通过 `joinaliasvars` 中的 Var 替换引用连接结果的 Var。

---

### 3.5.2 常量化简（eval_const_expressions）

**优化点**：

1. **参数常量化**：参数表达式全部为常量时预先求值
2. **函数常量化**：`simplify_function` 函数，所有参数为常量时获取执行结果
3. **约束条件常量化**

**示例**：

```sql
-- 优化前
SELECT MAX(degree + (1+2)) FROM SCORE;
-- 优化后
SELECT MAX(degree + 3) FROM SCORE;
```

---

### 3.5.3 谓词规范

#### 3.5.3.1 谓词规约

**OR 操作**：`NULL OR FALSE OR sno = 1` → `sno = 1`（忽略 NULL 和 FALSE）
**AND 操作**：`NULL AND FALSE AND sno = 1` → `FALSE`（整个约束条件为 FALSE）

**函数**：`find_duplicate_ors`

#### 3.5.3.2 谓词拉平

将树状的 OR/AND 约束条件拉平为扁平结构。

**示例**：

```sql
-- 优化前（树状）
sno=1 OR (sno=2 OR (sno=3 OR sno=4))
-- 优化后（扁平）
(sno = 1) OR (sno = 2) OR (sno = 3) OR (sno = 4)
```

**函数**：`pull_ors` / `pull_ands`

#### 3.5.3.3 提取公共项

将 `(A AND B) OR (A AND C)` 转换为 `A AND (B OR C)`，使 A 可以下推到基表。

**算法**：

1. 找到最短的 AND 子句
2. 以最短项为依据，检查所有 AND 子句是否都包含公共项
3. 提取公共项，生成新的 OR-of-ANDs 形式

**函数**：`process_duplicate_ors`

**特殊规约**：

- `(A AND B) OR A` → `A`
- `(A AND B AND C) OR (A AND B)` → `A AND B`

---

### 3.5.4 子连接处理（make_subplan）

**相关 vs 非相关子执行计划**：

- **非相关**：生成 Param，父计划执行时才对子计划求值，值固定可复用
- **相关**：直接返回子执行计划，通过 Param 通信

**执行计划标记**：

- 非相关：`InitPlan 1 (returns $0)`
- 相关：`SubPlan 1`

---

## 3.6 处理 HAVING 子句

**优化**：将 HAVING 中可转换为过滤条件的约束条件下推到 WHERE。

**示例**：

```sql
-- 原始
SELECT SUM(degree), sno, cno FROM SCORE
WHERE sno > 0 GROUP BY sno, cno
HAVING SUM(degree) > 100 AND cno > 0;
-- 优化后
-- cno > 0 下推到 WHERE，SUM(degree) > 100 保留在 HAVING
```

---

## 3.7 Group By 键值消除

**原理**：如果 Group By 包含表的全部主键，主键已能唯一标识分组，可消除其他字段。

**消除条件**：Group By 字段包含表的所有主键键值。

**函数**：`remove_useless_groupby_columns`

**示例**：

```sql
-- 优化前
SELECT * FROM STUDENT GROUP BY sno, sname, ssex;
-- 优化后（sno 是主键）
SELECT * FROM STUDENT GROUP BY sno;
```

---

## 3.8 外连接消除

### 3.8.1 外连接转内连接

**消除条件**：

1. 上层有"严格"（strict）的约束条件
2. 约束条件中引用了 Nullable-side 的表

**"严格"定义**：

- 函数/操作符/表达式：输入 NULL → 输出 NULL（或 FALSE）
- `IS NOT NULL` 是严格的
- `IS NULL` 是不严格的

**判断方式**：

- 函数：`PG_PROC` 系统表的 `proisstrict` 列
- 操作符：转为函数后同上
- NullTest：`IS NOT NULL` 严格，`IS NULL` 不严格

### 3.8.2 左连接转反连接（Anti Join）

**转换条件**：

1. 当前层的连接条件必须是严格的
2. 上层的约束条件和当前层的连接条件都引用了 Nullable-side 表的同一列
3. 上层的约束条件强制 Nullable-side 产生的结果必须是 NULL

**示例**：

```sql
-- 原始
SELECT * FROM STUDENT LEFT JOIN SCORE ON STUDENT.sno = SCORE.sno
WHERE SCORE.sno IS NULL;
-- 优化后
SELECT * FROM STUDENT ANTI JOIN SCORE ON STUDENT.sno = SCORE.sno;
```

### 3.8.3 右连接转左连接

所有右外连接都转换为左外连接，简化后续逻辑优化处理。

### 3.8.4 实现函数

**两步流程**：

1. **预检**（`reduce_outer_joins_pass1`）：递归遍历查询树，记录外连接层次结构
2. **消除**（`reduce_outer_joins_pass2`）：根据约束条件判断并消除外连接

**关键结构体**：

```c
typedef struct reduce_outer_joins_state {
  Relids relids;                    // 当前层次及下层引用的表
  bool contains_outer;              // 下层是否有外连接
  List *sub_states;                 // 树状子状态
} reduce_outer_joins_state;
```

**参数传递规则**（处理 JoinExpr 时）：

- 外连接在左子树 + 内/半连接 → 传递参数变量 + 当前层变量
- 外连接在左子树 + 左/反半连接 → 仅传递参数变量
- 外连接在右子树 + 非全连接 → 传递参数变量 + 当前层变量
- 全连接 → 不传递变量

---

## 3.9 grouping_planner 的说明

**职责**：

1. 对 SPJ（Selection-Projection-Join）操作进行优化（`query_planner` 函数）
2. 在 SPJ 最优路径基础上叠加 Non-SPJ 操作路径（Group By、LIMIT、ORDER BY）

**两个重要点**：

1. **Limit 影响启动代价**：`tuple_fraction` 变量在代价计算中起重要作用
2. **PathKeys 提示排序**：根据 Order By/Group By 生成 PathKeys，优先选择符合顺序的路径

**PathKeys 优先级**：

1. Group By 产生的 PathKeys（第 1 顺位）
2. 窗口函数产生的 PathKeys（第 2 顺位）
3. Order By 或 DISTINCT 产生的 PathKeys（第 3 顺位，长的优先）

---

## 3.10 小结

**逻辑重写优化的核心工作**：

1. 子查询/子连接的提升
2. 外连接的消除
3. 谓词规范（规约、拉平、提取公共项）
4. 常量化简
5. Group By 键值消除
6. 继承表展开

**优化后效果**：

- 查询树中不再有右外连接
- 查询层次减少
- 冗余操作消除
- 为后续逻辑分解优化（谓词下推、连接顺序交换等）奠定基础

---

## 关键数据结构速查

| 结构体                        | 用途                                                      |
| ----------------------------- | --------------------------------------------------------- |
| `SubLink`                     | 子连接信息（类型、操作符、子查询树）                      |
| `AppendRelInfo`               | UNION ALL / 继承表的层次提升信息                          |
| `pullup_replace_vars_context` | 子查询提升时替换 Var 的上下文                             |
| `reduce_outer_joins_state`    | 外连接消除的预检状态                                      |
| `Var`                         | 列属性（varno=表编号, varattno=列编号, varlevelsup=层次） |