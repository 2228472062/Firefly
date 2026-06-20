---
title: 第十章-生成执行计划
published: 2026-06-16 09:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
## 本章概述
本章详细介绍了PostgreSQL查询优化器的最后一步：将最优执行路径（Path）转换为可执行的执行计划（Plan）。主要内容包括转换流程（扫描计划和连接计划的生成）以及执行计划树的清理优化。

## 核心主题
- **路径到计划的转换**：如何将包含冗余信息的Path结构转换为紧凑高效的Plan结构
- **扫描计划生成**：顺序扫描、索引扫描等扫描路径的转换细节
- **连接计划生成**：MergeJoin、HashJoin、NestLoopJoin等连接路径的转换
- **计划树清理**：范围表拉平、Var变量调整等最终优化步骤

---

## 10.1 转换流程

### 10.1.1 扫描计划

#### 公共处理（create_scan_plan函数）
在处理具体扫描路径前，所有扫描节点需要完成以下公共任务：

1. **获取过滤条件**：
   - 普通扫描：从`RelOptInfo->baserestrictinfo`获取
   - 索引扫描：从`IndexOptInfo->indrestrictinfo`获取（可能与baserestrictinfo不同）

2. **处理参数化约束条件**：
   - 如果扫描节点上层是连接操作，可能有下推的参数化约束条件
   - 通过`join_clause_is_movable_to`函数判断是否可下推
   - 示例：Nested Loop中内表的Index Scan使用外表列作为参数

3. **常量约束条件优化**：
   - 将常量约束条件作为gating约束条件
   - 因为常量条件不受执行时变量影响，可能带来优化

4. **投影列优化**：
   - 对于基表扫描，即使只查询部分列，也可能投影所有列以提高效率
   - 但如果有上层节点（如Hash、Sort、Material）要求最小投影，则设置`CP_SMALL_TLIST`标志

#### 顺序扫描计划（create_seqscan_plan）
生成步骤：
1. **调整约束条件执行顺序**：
   - 安全级别相同时，按表达式代价排序
   - 代价越低的约束条件越靠前

2. **提取表达式**：
   - 从`RestrictInfo`结构体中提取表达式
   - 执行器只需要执行表达式，不需要优化信息

3. **参数替换**（如果是参数化路径）：
   - 检查约束条件是否引用外表列
   - 将`Var`或`PlaceHolderVar`替换为`Param`
   - 建立对应的`NestloopParam`结构体

#### 索引扫描计划（create_indexscan_plan）
比顺序扫描复杂，主要处理：

1. **约束条件分类**：
   - **匹配索引的约束条件**：用于索引查找（Index Cond）
   - **不匹配索引的约束条件**：作为过滤条件（Filter）

2. **Var替换**：
   - 将表上的Var替换为索引上的Var
   - 通过`fix_indexqual_references`函数实现
   - 示例：`TEST_A(b)` → `TEST_A_B_IDX(b)`

3. **过滤条件收集**：
   - 从`scan_clauses`中减去`indexquals`
   - 跳过伪常量约束条件
   - 跳过与索引条件基于同一等价类的冗余条件
   - 跳过被索引条件蕴含的条件

4. **蕴含关系判断**：
   - 使用`predicate_implied_by`函数
   - 支持9种蕴含关系（atom、AND-expr、OR-expr之间的组合）
   - 示例：`(a=1) => (a=1 OR c=2)`，因此`c=2`条件可消除

---

### 10.1.2 连接计划

#### 连接类型
- **MergeJoin**：需要排序的等值连接
- **HashJoin**：使用哈希表的等值连接
- **NestLoopJoin**：嵌套循环连接

#### MergeJoin转换要点
1. **子计划递归生成**：
   - 调用`create_plan_recurse`将子路径转为子计划
   - 如果子路径需要排序，设置`CP_SMALL_TLIST`避免扩展投影列

2. **约束条件分类**：
   - **连接条件**：用于连接操作
   - **过滤条件**：连接后的过滤
   - 通过`is_pushed_down`变量区分（`true`表示过滤条件）

3. **MergeJoinable条件**：
   - 用于排序和合并的等值条件
   - 其他条件作为连接过滤条件

4. **显式排序**：
   - 根据`PathKeys`添加Sort节点
   - 虽然Sort在计划生成时创建，但代价已在路径计算时考虑

---

## 10.2 执行计划树清理

### 主要工作
将子计划的范围表拉平到父计划的范围表，统一管理。

### 10.2.1 范围表拉平
- **目的**：将子计划的范围表合并到父计划
- **结果**：所有计划共享同一个范围表（`finalrtable`）

### 10.2.2 Var变量调整
1. **扫描计划调整**：
   - 遍历计划树中的Var变量
   - 调整`varno`值（增加偏移量`rtoffset`）
   - 通过`fix_scan_list`函数实现

2. **连接计划调整**：
   - 使用`INNER_VAR`和`OUTER_VAR`标识变量来源
   - 通过`search_indexed_tlist_for_var`函数查找

### 10.2.3 并行聚集函数修正
- **问题**：Partial聚集和Finalize聚集使用相同Var
- **解决**：通过`convert_combining_aggrefs`函数修正
- **步骤**：
  1. 生成`child_agg`（Partial聚集）
  2. 生成`parent_agg`（Finalize聚集）
  3. 将`child_agg`作为`parent_agg`的参数

### 10.2.4 参数改进
- **PARAM_MULTIEXPR**：转换为多个`PARAM_EXEC`参数
- 通过`generate_subquery_params`函数实现

### 10.2.5 函数依赖关系记录
- 通过`record_plan_function_dependency`记录用户自定义函数与执行计划的依赖
- 提供给缓存模块，用于失效管理

### 10.2.6 SubqueryScan消除
**消除条件**：
1. SubqueryScan没有约束条件
2. SubqueryScan的投影链表与子计划相同

**示例**：
- 原始计划：`NestloopJoin → Material → SubqueryScan → Limit → SeqScan`
- 优化后：`NestloopJoin → Material → Limit → SeqScan`

---

## 关键技术点总结

### 1. Path与Plan的对应关系
- 每个Path节点对应一个Plan节点
- Path包含代价计算等冗余信息，Plan只包含执行必要信息

### 2. 约束条件处理
- **扫描条件**：过滤条件 + 参数化约束条件
- **连接条件**：连接条件 + 过滤条件
- **索引条件**：匹配索引的条件（Index Cond） + 不匹配的条件（Filter）

### 3. 投影列优化
- **CP_SMALL_TLIST标志**：避免对无用数据排序
- 适用于Hash、Sort、Material等节点

### 4. 范围表管理
- 子计划范围表拉平到父计划
- 统一管理，简化执行器处理

### 5. 变量引用调整
- 扫描计划：调整varno偏移量
- 连接计划：使用INNER_VAR/OUTER_VAR标识来源

---

## 实际应用示例

### 示例1：索引扫描条件分类
```sql
CREATE TABLE TEST_A(a INT, b INT, c INT, d INT);
CREATE INDEX TEST_A_A_IDX ON TEST_A(a);
EXPLAIN SELECT * FROM TEST_A WHERE a=1 AND b > 2;
```
**执行计划**：
- `Index Cond: (a = 1)` -- 匹配索引
- `Filter: (b > 2)` -- 不匹配索引

### 示例2：连接条件区分
```sql
EXPLAIN SELECT * FROM TEST_A LEFT JOIN TEST_B ON TEST_A.a = TEST_B.a AND TEST_A.b = 1;
```
**执行计划**：
- `Merge Cond: (test_a.a = test_b.a)` -- MergeJoinable条件
- `Join Filter: (test_a.b = 1)` -- 普通连接条件

### 示例3：SubqueryScan消除
```sql
EXPLAIN SELECT * FROM TEST_A LEFT JOIN (SELECT * FROM TEST_B LIMIT 10) b ON TRUE;
```
**优化结果**：消除SubqueryScan节点，直接连接Limit和SeqScan

---

## 总结

本章深入讲解了PostgreSQL查询优化器的最后一步：执行计划生成。核心内容包括：

1. **转换流程**：将Path节点转换为Plan节点，处理扫描和连接的不同需求
2. **约束条件处理**：分类处理扫描条件、连接条件、索引条件
3. **计划树清理**：范围表拉平、Var变量调整、并行聚集修正等
4. **优化技术**：SubqueryScan消除、常量条件优化等
