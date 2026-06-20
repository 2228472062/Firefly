---
title: 第九章-NonSPJ优化
published: 2026-06-16 08:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
## 一、章节概述

本章介绍 PostgreSQL 查询优化器中 **Non-SPJ（非Select-Project-Join）操作** 的优化过程。Non-SPJ优化主要包括：

- 集合操作（UNION、INTERSECT、EXCEPT）
- 聚集操作（GROUP BY、GROUPING SETS、ROLLUP、CUBE）
- Limit子句处理
- Order By子句处理
- 窗口函数处理

**核心思想**：在SPJ优化路径上叠加Non-SPJ路径，分为预处理和路径生成两个阶段。

---

## 二、Non-SPJ优化整体架构

### 2.1 优化流程

```
SQL查询
    ↓
SPJ优化（query_planner函数）
    ↓
Non-SPJ预处理（表达式预处理）
    ↓
Non-SPJ路径生成（叠加物理路径）
    ↓
生成最终执行计划
```

### 2.2 UpperRelationKind枚举

Non-SPJ路径存储在 `RelOptInfo` 的 `upperrels` 数组中，下标含义如下：

```c
typedef enum UpperRelationKind
{
    UPPERREL_SETOP,      // 集合操作的结果路径
    UPPERREL_GROUP_AGG,  // Group By子句和聚集函数对应的路径
    UPPERREL_WINDOW,     // 窗口函数对应的路径
    UPPERREL_DISTINCT,   // 去重操作对应的路径
    UPPERREL_ORDERED,    // Order By子句对应的路径
    UPPERREL_FINAL       // 叠加了Non-SPJ之后的路径
} UpperRelationKind;
```

---

## 三、集合操作处理（9.1节）

### 3.1 集合操作类型

| 操作      | 语义         | ALL变体                 |
| --------- | ------------ | ----------------------- |
| UNION     | 并集（去重） | UNION ALL（不去重）     |
| INTERSECT | 交集（去重） | INTERSECT ALL（不去重） |
| EXCEPT    | 差集（去重） | EXCEPT ALL（不去重）    |

### 3.2 执行计划特点

集合操作的执行计划本质上是通过 **Append** 方式实现的：

```sql
EXPLAIN VERBOSE 
SELECT * FROM TEST_A UNION SELECT * FROM TEST_B ORDER BY a,b,c LIMIT 2;
```

```
Limit (cost=205.00..205.01 rows=2 width=16)
-> Sort (cost=205.00..214.25 rows=3700 width=16)
   -> HashAggregate (cost=131.00..168.00 rows=3700 width=16)
      -> Append (cost=0.00..94.00 rows=3700 width=16)
         -> Seq Scan on public.test_a (cost=0.00..28.50 rows=1850 width=16)
         -> Seq Scan on public.test_b (cost=0.00..28.50 rows=1850 width=16)
```

### 3.3 投影列处理

集合操作的投影列借用 **最左端叶子节点** 的列名：

```c
// 使用最左端的叶子节点
node = topop->larg;
while (node && IsA(node, SetOperationStmt))
    node = ((SetOperationStmt *) node)->larg;
Assert(node && IsA(node, RangeTblRef));
leftmostRTE = root->simple_rte_array[((RangeTblRef *) node)->rtindex];
leftmostQuery = leftmostRTE->subquery;
```

### 3.4 数据类型转换

子节点数据类型不一致时，需要在 **叶子节点** 进行隐式强制转换：

```sql
CREATE TABLE TEST_INT2(a INT2);
CREATE TABLE TEST_INT4(a INT4);
CREATE TABLE TEST_INT8(a INT8);

EXPLAIN SELECT * FROM TEST_INT2 UNION ALL SELECT * FROM TEST_INT4 UNION ALL SELECT * FROM TEST_INT8;
```

```
Append (cost=0.00..253.27 rows=7530 width=8)
-> Result (cost=0.00..198.07 rows=5270 width=8)
   -> Append (cost=0.00..132.20 rows=5270 width=4)
      -> Subquery Scan on "*SELECT* 1" (cost=0.00..71.20 rows=2720 width=4)
         -> Seq Scan on test_int2 (cost=0.00..37.20 rows=2720 width=2)
      -> Seq Scan on test_int4 (cost=0.00..35.50 rows=2550 width=4)
   -> Seq Scan on test_int8 (cost=0.00..32.60 rows=2260 width=8)
```

### 3.5 路径生成函数

集合操作路径生成分为3种情况：

```c
// 递归调用
generate_recursion_path    // RecursiveUnion
generate_union_path        // UNION
generate_nonunion_path     // EXCEPT & INTERSECT

// 核心函数
recurse_set_operations     // 递归处理集合操作
```

---

## 四、Non-SPJ预处理（9.2.1节）

### 4.1 Group By子句预处理

#### 4.1.1 GROUPING SETS、ROLLUP、CUBE扩展

`expand_grouping_sets` 函数负责扩展复杂分组：

```sql
-- 示例：GROUP BY a, CUBE(b, c, d)
-- CUBE(b,c,d) 语义：{b} ∪ {c} ∪ {d} ∪ {b,c} ∪ {b,d} ∪ {c,d} ∪ {b,c,d} ∪ {}
-- 与{a}的卡氏积：8种情况
```

扩展后的等价Group Keys：

```c
// 排序后（从短到长）
{a} ∪ {a,b} ∪ {a,c} ∪ {a,d} ∪ {a,b,c} ∪ {a,b,d} ∪ {a,c,d} ∪ {a,b,c,d}
```

#### 4.1.2 ROLLUP分组优化

`extract_rollup_sets` 函数使用 **Hopcroft-Karp算法**（二分图最大匹配）将Group Keys划分为多组ROLLUP：

```sql
-- GROUP BY a, CUBE(b,c,d) 可以划分为3组ROLLUP
ROLLUP(a,b,c,d) = {a,b,c,d} ∪ {a,b,c} ∪ {a,b} ∪ {a}
ROLLUP(a,c,d) = {a,c,d} ∪ {a,c}
ROLLUP(a,b,d) = {a,d,b} ∪ {a,d}
```

#### 4.1.3 排序借用优化

`reorder_grouping_sets` 函数判断是否可以借用Order By子句的排序：

```sql
-- 示例1：没有借用Order By的排序结果
EXPLAIN SELECT a,b,c FROM TEST_A GROUP BY GROUPING SETS((a,b,c),(c)) ORDER BY c,b,a;
-- 需要额外的Sort节点

-- 示例2：借用了Order By的排序结果
EXPLAIN SELECT a,b,c FROM TEST_A GROUP BY GROUPING SETS((a,b,c),(c)) ORDER BY c,a,b;
-- 直接使用GroupAggregate，无需额外排序
```

### 4.2 聚集函数预处理

`get_agg_clause_costs` 函数计算聚集函数的代价：

```c
// 遍历投影列中的聚集函数
get_agg_clause_costs(root, (Node *) tlist, AGGSPLIT_SIMPLE, &agg_costs);

// 遍历Having子句中的聚集函数
get_agg_clause_costs(root, parse->havingQual, AGGSPLIT_SIMPLE, &agg_costs);
```

**关键判断**：

```c
// 如果聚集函数包含DISTINCT或WITHIN GROUP，不能创建并行聚集路径
if (aggref->aggorder != NIL || aggref->aggdistinct != NIL)
{
    costs->numOrderedAggs++;
    costs->hasNonPartial = true;
}

// 并行聚集必须需要combine函数
if (!OidIsValid(aggcombinefn))
    costs->hasNonPartial = true;

// INTERNAL类型的中间类型必须有序列化和反序列化函数
if (aggtranstype == INTERNALOID &&
    (!OidIsValid(aggserialfn) || !OidIsValid(aggdeserialfn)))
    costs->hasNonSerial = true;
```

### 4.3 MIN/MAX优化预处理

利用B树索引获取最大值或最小值：

```sql
-- 示例
CREATE INDEX TEST_A_A_IDX ON TEST_A(a);
CREATE INDEX TEST_A_B_IDX ON TEST_A(b);

EXPLAIN SELECT MAX(a), MIN(b) FROM TEST_A WHERE c > 1;
```

```
Result (cost=0.66..0.67 rows=1 width=8)
InitPlan 1 (returns $0)
-> Limit (cost=0.29..0.32 rows=1 width=4)
   -> Index Scan Backward using test_a_a_idx on test_a
      Index Cond: (a IS NOT NULL)
      Filter: (c > 1)
InitPlan 2 (returns $1)
-> Limit (cost=0.29..0.34 rows=1 width=4)
   -> Index Scan using test_a_b_idx on test_a test_a_1
      Index Cond: (b IS NOT NULL)
      Filter: (c > 1)
```

**适用条件**：

1. 只能包含MIN/MAX聚集函数，不能有其他聚集函数
2. 多表连接时不适用
3. 查询中不能包含Group By子句
4. 语句中不能包含CTE表达式
5. MIN/MAX对应的列上必须有B树索引

---

## 五、Non-SPJ路径生成（9.2.2节）

### 5.1 投影处理

#### 5.1.1 易失性表达式处理

如果投影中有Order By子句，易失性表达式（如 `nextval('SEQ')`）可能因为路径不同而产生不同结果：

```sql
CREATE TEMP SEQUENCE SEQ;
SELECT a, nextval('SEQ') FROM TEST_A ORDER BY a LIMIT 10;
```

**问题**：

- 路径1（显式排序）：对所有行执行nextval
- 路径2（借用IndexScan有序特性）：只执行10次nextval

**解决方案**：将易失性表达式转移到高层节点执行：

```sql
EXPLAIN VERBOSE SELECT a, nextval('SEQ') FROM TEST_A ORDER BY a LIMIT 10;
```

```
Limit (cost=371.10..371.25 rows=10 width=12)
-> Result (cost=371.10..521.10 rows=10000 width=12)
   -> Sort (cost=371.10..396.10 rows=10000 width=4)
      -> Seq Scan on public.test_a (cost=0.00..155.00 rows=10000 width=4)
```

#### 5.1.2 投影列构建

`make_sort_input_target` 函数处理3种优化情况：

1. 易失性表达式需要在高层节点计算
2. SRF表达式需要在高层节点计算（如 `generate_series`）
3. 代价较高的表达式（> 10 × cpu_operator_cost）在有Limit子句时需要在高层节点计算

### 5.2 聚集与分组

#### 5.2.1 分组路径类型

分组路径的实现主要有两种手段：

**基于排序的分组**：

- 需要满足条件：每个Group Keys都可以满足基于排序的操作符
- 优势：可以借用Order By的排序结果

**基于Hash的分组**：

- 需要满足条件：
    - 查询中必须包含Group By子句
    - 聚集函数中不能包含排序的情况
    - 聚集函数中不能包含DISTINCT

#### 5.2.2 并行聚集

并行聚集分两个阶段实现：

```sql
EXPLAIN VERBOSE SELECT SUM(a) FROM TEST_A;
```

```
Finalize Aggregate (cost=7793.55..7793.56 rows=1 width=8)
-> Gather (cost=7793.33..7793.54 rows=2 width=8)
   Workers Planned: 2
   -> Partial Aggregate (cost=6793.33..6793.34 rows=1 width=8)
      -> Parallel Seq Scan on public.test_a
```

**并行聚集条件**：

1. 路径中的表达式需要是并行安全的
2. 聚集的子路径也需要是并行路径
3. 路径中或者有聚集函数、或者有Group By子句
4. Group By子句中不能包含Grouping Sets、ROLLUP、CUBE

### 5.3 窗口函数处理

窗口函数在 `UPPERREL_WINDOW` 层处理，生成窗口函数路径。

---

## 六、关键函数详解

### 6.1 grouping_planner函数

Non-SPJ优化的核心函数，以 `query_planner` 函数为分水岭：

```c
// 在query_planner函数之前：对Non-SPJ操作中的各种表达式进行预处理
// 在query_planner函数中：SPJ优化路径的生成
// 在query_planner函数之后：开始在SPJ优化路径上叠加新的Non-SPJ路径
```

### 6.2 generate_union_path函数

生成UNION操作的路径：

```c
// 核心逻辑
void generate_union_path(...)
{
    // 1. 递归处理子查询
    recurse_set_operations(...);
    
    // 2. 创建Append路径
    // 3. 处理去重（如果有ALL关键字则跳过）
    // 4. 生成最终路径
}
```

### 6.3 create_minmax_path函数

创建MIN/MAX优化的路径：

```c
// 基于如下SQL语句生成子Plan
// SELECT col FROM tab
// WHERE col IS NOT NULL AND existing-quals
// ORDER BY col ASC/DESC
// LIMIT 1;
```

---

## 七、执行计划示例分析

### 7.1 完整的Non-SPJ执行计划

```sql
EXPLAIN SELECT a,b,c,sum(d) as d 
FROM TEST_A 
GROUP BY a,b,c 
ORDER BY d 
LIMIT 10;
```

```
Limit (cost=1260.48..1260.51 rows=10 width=20)
-> Sort (cost=1260.48..1285.48 rows=10000 width=20)
   Sort Key: (sum(d))
   -> GroupAggregate (cost=819.39..1044.39 rows=10000 width=20)
      Group Key: a, b, c
      -> Sort (cost=819.39..844.39 rows=10000 width=16)
         Sort Key: a, b, c
         -> Seq Scan on test_a (cost=0.00..155.00 rows=10000 width=16)
```

**分析**：

- SPJ部分：SeqScan路径
- Non-SPJ叠加：GroupAggregate（基于排序的分组）→ Sort（Order By）→ Limit

### 7.2 集合操作执行计划

```sql
EXPLAIN VERBOSE 
SELECT * FROM TEST_A UNION SELECT * FROM TEST_B ORDER BY a,b,c LIMIT 2;
```

```
Limit (cost=205.00..205.01 rows=2 width=16)
-> Sort (cost=205.00..214.25 rows=3700 width=16)
   -> HashAggregate (cost=131.00..168.00 rows=3700 width=16)
      -> Append (cost=0.00..94.00 rows=3700 width=16)
         -> Seq Scan on public.test_a (cost=0.00..28.50 rows=1850 width=16)
         -> Seq Scan on public.test_b (cost=0.00..28.50 rows=1850 width=16)
```

**分析**：

- 集合操作通过Append实现
- HashAggregate用于去重（UNION不带ALL）
- Sort用于Order By
- Limit用于限制返回行数

---

## 八、关键要点总结

### 8.1 核心概念

1. **Non-SPJ优化**：非Select-Project-Join优化，包括集合操作、聚集操作、Limit、Order By等
2. **预处理阶段**：对各种Non-SPJ操作的表达式进行预处理
3. **路径生成阶段**：将Non-SPJ操作对应的物理路径叠加到SPJ优化产生的连接树上

### 8.2 重要设计

1. **投影列借用**：集合操作借用最左端叶子节点的列名
2. **数据类型转换**：在叶子节点进行，尽早转换
3. **排序借用**：Group By尝试借用Order By的排序结果
4. **易失性表达式转移**：将易失性表达式转移到高层节点执行

### 8.3 性能优化

1. **MIN/MAX优化**：利用B树索引快速获取极值
2. **并行聚集**：分两个阶段（Partial + Finalize）实现并行
3. **ROLLUP分组优化**：使用Hopcroft-Karp算法划分Group Keys

### 8.4 与其他章节的关系

- **第8章（SPJ优化）**：Non-SPJ优化的基础
- **第10章（执行计划生成）**：Non-SPJ路径最终转换为执行计划

---

## 九、学习建议

1. **动手实践**：使用EXPLAIN分析各种Non-SPJ查询的执行计划
2. **对比分析**：对比有无Group By、Order By、Limit时的执行计划差异
3. **源码阅读**：重点阅读 `grouping_planner`、`generate_union_path` 等函数
4. **性能调优**：理解哪些写法能被优化器更好地优化

---

## 十、遗留问题

1. Hopcroft-Karp算法在ROLLUP分组中的具体实现细节
2. 并行聚集的代价计算模型
3. 窗口函数与Group By的交互优化

---