---
title: 第四章-逻辑分解优化
published: 2026-06-15 03:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
## 一、章节概览

本章是逻辑优化的核心章节，承接逻辑重写优化，开启物理优化的准备工作。主要内容是将查询树中的逻辑结构转换为适合物理优化的数据结构，并进行谓词下推、等价类推理、连接顺序优化等关键操作。

**核心主题**：从逻辑层（RangeTblEntry）到物理层（RelOptInfo）的转换，以及约束条件的智能下推与推理。

---

## 二、核心概念详解

### 2.1 RelOptInfo 结构体

**作用**：替代 RangeTblEntry，用于物理优化阶段的代价计算和路径生成。

**三种主要类型**：

| 类型                      | 说明                           |
| ------------------------- | ------------------------------ |
| `RELOPT_BASEREL`          | 基表（普通表、子查询、函数等） |
| `RELOPT_JOINREL`          | 连接操作产生的中间结果         |
| `RELOPT_OTHER_MEMBER_REL` | 继承表子表、UNION子查询等      |

**关键成员变量**：

```c
Relids relids;              // 基表的rtindex或连接表的rtindex集合
double rows;                // 估计的结果集行数
bool consider_startup;      // 是否考虑启动代价（用于LIMIT子句）
bool consider_param_startup; // 参数化路径是否考虑启动代价
bool consider_parallel;     // 是否考虑并行查询

// 路径相关
List *pathlist;             // 所有记录的路径
List *ppilist;              // 参数化路径的参数
List *partial_pathlist;     // 并行路径链表
struct Path *cheapest_startup_path;  // 启动代价最低的路径
struct Path *cheapest_total_path;    // 整体代价最低的路径

// 基表特有变量
Index relid;                // 基表的rtindex
Oid reltablespace;          // 表空间
RTEKind rtekind;            // 表类型（RTE_RELATION/RTE_SUBQUERY/RTE_FUNCTION）
AttrNumber min_attr, max_attr;  // 首列和末列编号
Relids *attr_needed;        // 每列被哪些表引用
int32 *attr_widths;         // 每列的宽度
List *indexlist;            // 索引列表（IndexOptInfo）
BlockNumber pages;          // 页面数
double tuples;              // 元组数
List *baserestrictinfo;     // 基表过滤条件
List *joininfo;             // 连接条件
```

### 2.2 IndexOptInfo 结构体

**作用**：描述索引信息，用于生成索引扫描路径和计算代价。

**关键成员**：

```c
Oid indexoid;           // 索引OID
RelOptInfo *rel;        // 所属基表
BlockNumber pages;      // 索引页面数
double tuples;          // 索引元组数
int ncolumns;           // 索引键值数
int *indexkeys;         // 具体的键值
Oid *opfamily;          // 操作符族
Oid *opcintype;         // 操作符类
bool *reverse_sort;     // 升序/降序
bool *nulls_first;      // NULL位置
bool *canreturn;        // 索引能否直接返回索引项
Oid relam;              // 访问方法（B-tree/Hash等）
List *indexprs;         // 表达式索引中的表达式
List *indpred;          // 局部索引的谓词
bool predOK;            // 谓词是否满足查询需要
bool unique;            // 唯一索引
```

**索引类型分类**：

- **唯一索引/主键索引**：通过 `unique` 标识，`immediate` 控制是否延迟检查
- **局部索引**：带 `WHERE` 条件的索引，存储在 `indpred`，`predOK` 标识是否可用
- **表达式索引**：索引键值包含表达式，存储在 `indexprs`

### 2.3 RestrictInfo 结构体

**作用**：封装约束条件，记录下推过程中的各种状态信息。

**关键成员**：

```c
Expr *clause;               // 约束条件表达式
bool is_pushed_down;        // true=过滤条件，false=不能下推的连接条件
bool outerjoin_delayed;     // 下推过程中是否被外连接阻碍
bool can_join;              // 是否可用于Merge Join/Hash Join（Var=Var形式）
bool pseudoconstant;        // 常量表达式
Relids clause_relids;       // 约束条件引用的表
Relids required_relids;     // 约束条件被应用的最小集合
Relids outer_relids;        // Nonnullable-side表的集合
Relids nullable_relids;     // 下层外连接的Nullable-side表集合
Relids left_relids;         // 左操作数对应的表
Relids right_relids;        // 右操作数对应的表
EquivalenceClass *left_ec;  // 左操作数对应的等价类
EquivalenceClass *right_ec; // 右操作数对应的等价类
```

---

## 三、核心算法流程

### 3.1 创建 RelOptInfo

**流程**：

1. `setup_simple_rel_arrays()`：将 Query->rtable 中的 RangeTblEntry 提取到 `simple_rte_array`
2. `add_base_rels_to_query()`：遍历 Query->jointree，为每个 RangeTblRef 创建 RelOptInfo
3. `build_simple_rel()`：实际创建 RelOptInfo，根据 rtekind 确定具体类型

**基表类型层次**：

```
RELOPT_BASEREL
├── RTE_RELATION（普通表）→ get_relation_info() 填充统计信息
├── RTE_SUBQUERY（子查询）→ 递归遍历子查询
├── RTE_VALUES（VALUES子查询）
└── RTE_FUNCTION（函数型基表）
```

**普通表信息获取**：

- 页面数：`RelationGetNumberOfBlocks()`（文件大小/块大小）
- 元组数：通过 `SYS_CLASS` 统计信息估算（元组密度 × 当前页面数）
- TRUNCATE后的表：假设页面"满"，用 `BLCKSZ / tuple_width` 计算

### 3.2 等价类（EquivalenceClass）

**定义**：基于多个等值约束条件（A=B）推理出的等价属性集合。

**示例**：

```sql
-- A=B AND B=C → {A,B,C}构成等价类
-- 推理出 A=C，可生成更多连接路径

-- A=B AND B=5 → {A,B,5}构成等价类
-- 推理出 A=5，可下推到基表过滤
```

**数据结构**：

```c
typedef struct EquivalenceClass {
    List *ec_opfamilies;    // 操作符族
    List *ec_members;       // 等价类成员链表（EquivalenceMember）
    List *ec_sources;       // 产生等价类的约束条件
    List *ec_derives;       // 推理产生的约束条件
    Relids ec_relids;       // 涉及的所有表
    bool ec_has_const;      // 是否含常量
    bool ec_has_volatile;   // 是否含易失性函数
    bool ec_below_outer_join; // 是否在外连接之下
    bool ec_broken;         // 推理失败标记
    struct EquivalenceClass *ec_merged; // 已合并到另一个等价类
} EquivalenceClass;
```

**等价类的价值**：

1. 扩充连接路径：A=B, B=C → 推理出 A=C
2. 下推过滤条件：A=B, B=5 → 推理出 A=5 下推到A表
3. 优化排序：A=B, ORDER BY B → 可按A排序

---

## 四、谓词下推（Predicate Pushdown）

### 4.1 连接条件下推规则

**结论1**：连接条件下推后变成过滤条件，过滤条件下推后仍然是过滤条件。

**结论2**（外连接限制）：

- 连接条件引用了 **Nonnullable-side** 的表 → **不能下推**
- 连接条件只引用了 **Nullable-side** 的表 → **可以下推**

**各连接类型的 Nullable-side/Nonnullable-side**：

| 连接类型       | Nonnullable-side | Nullable-side |
| -------------- | ---------------- | ------------- |
| A Left Join B  | A                | B             |
| A Right Join B | B                | A             |
| A Full Join B  | A、B             | A、B          |
| A Semi Join B  | A、B             | 无            |
| A Anti Join B  | A                | B             |

**示例**：

```sql
-- 左外连接，连接条件引用Nonnullable-side，不能下推
SELECT * FROM STUDENT LEFT JOIN SCORE ON STUDENT.sno = 1;
-- 执行计划：Join Filter: (student.sno = 1) -- 保持为连接条件

-- 右外连接，连接条件引用Nullable-side，可以下推
SELECT * FROM STUDENT RIGHT JOIN SCORE ON STUDENT.sno = 1;
-- 执行计划：Filter: (sno = 1) -- 变成过滤条件
```

### 4.2 过滤条件下推规则

**结论3**：

- 过滤条件只引用 Nonnullable-side 的表 → 可以下推
- 过滤条件引用 Nullable-side 的表且是 **严格** 的 → 导致外连接消除，变内连接后可下推
- 过滤条件引用 Nullable-side 的表且是 **不严格** 的 → **不能下推**

**"严格"的定义**：输入NULL，输出NULL（精确的严格）

**示例**：

```sql
-- 严格过滤条件，导致外连接消除
SELECT * FROM SCORE LEFT JOIN COURSE ON TRUE WHERE COURSE.cno = 1;
-- 执行计划：Hash Join -- 变成内连接

-- 不严格过滤条件，不能下推
SELECT * FROM SCORE LEFT JOIN COURSE ON TRUE WHERE COURSE.cno IS NULL;
-- 执行计划：Filter: (course.cno IS NULL) -- 保持为过滤条件
```

### 4.3 连接顺序交换

**等式1.1**：(A leftjoin B on Pab) innerjoin C on Pac = (A innerjoin C on Pac) leftjoin B on Pab

**等式1.2**：(A leftjoin B on Pab) leftjoin C on Pac = (A leftjoin C on Pac) leftjoin B on Pab

**等式1.3**：(A leftjoin B on Pab) leftjoin C on Pbc = A leftjoin (B leftjoin C on Pbc) on Pab

- 条件：Pbc must be strict

**半连接等式2**：(A semijoin B ON Pab) join C ON Pac = (A join C ON Pac) semijoin B ON Pab

**反半连接等式3**：(A antijoin B ON Pab) join C ON Pac = (A join C ON Pac) antijoin B ON Pab

**大原则**：

- SemiJoin ≈ InnerJoin + 内表唯一化
- AntiJoin ≈ LeftJoin（都含有"补NULL"逻辑）

---

## 五、关键函数分析

### 5.1 deconstruct_recurse 函数

**作用**：递归遍历 Query->jointree，为 RelOptInfo 分发约束条件，建立逻辑连接关系。

**关键变量**：

- `qualscope`：当前连接层次下所有表的rtindex集合
- `inner_join_rels`：内连接涉及的表的rtindex集合
- `nullable_rels` / `nonnullable_rels`：Nullable-side / Nonnullable-side 表的集合
- `below_outer_join`：是否在外连接的Nullable-side之下
- `postponed_qual_list`：需要推迟分发的约束条件（Lateral相关）

**处理三种节点**：

1. **RangeTblRef**：设置 qualscope，inner_join_rels=NULL
2. **FromExpr**：遍历 fromlist，合并 sub_qualscope，处理 child_postponed_quals
3. **JoinExpr**：递归处理 larg/rarg，根据连接类型设置各变量

### 5.2 make_outerjoininfo 函数

**作用**：生成 SpecialJoinInfo 结构体，记录外连接/半连接/反半连接的连接关系和顺序限制。

**SpecialJoinInfo 结构体**：

```c
typedef struct SpecialJoinInfo {
    Relids min_lefthand;    // LHS出现的最小集
    Relids min_righthand;   // RHS出现的最小集
    Relids syn_lefthand;    // 语法中LHS的表集合
    Relids syn_righthand;   // 语法中RHS的表集合
    JoinType jointype;      // 连接类型
    bool lhs_strict;        // 约束条件是否严格
    bool delay_upper_joins; // 阻止与上层交换顺序
    bool semi_can_btree;    // 半连接是否可用B-tree
    bool semi_can_hash;     // 半连接是否可用Hash
    List *semi_operators;   // 半连接操作符
    List *semi_rhs_exprs;   // 半连接右操作数
} SpecialJoinInfo;
```

**最小集生成流程**：

1. 赋予初值：`min_lefthand = clause_relids ∩ left_rels`
2. 遍历下层 SpecialJoinInfo 限制连接顺序
3. 查看全局 PlaceHolderVar 链表，加入涉及 Nullable-side 的表

### 5.3 distribute_qual_to_rels 函数

**作用**：参照谓词下推规则，对约束条件进行下推。

**三部分检查**：

1. **推理产生的约束条件**（is_deduced=true）：一定是过滤条件或能下推的连接条件
2. **连接条件引用 Nonnullable-side**：不能下推，is_pushed_down=false
3. **过滤条件或能下推的连接条件**：is_pushed_down=true，检查外连接阻碍

**关键局部变量**：

- `is_pushed_down`：true=过滤条件，false=不能下推的连接条件
- `outerjoin_delayed`：下推是否被外连接阻碍
- `maybe_equivalence`：是否可用于生成等价类
- `maybe_outer_join`：外连接情况下的等价类处理

### 5.4 check_outerjoin_delay 函数

**作用**：

1. 检查约束条件下推是否被外连接阻碍
2. 设置 `delay_upper_joins` 标记（处理特殊左连接情况）

**阻碍条件**：约束条件引用了 Nullable-side 的表，但未包含整个外连接的表集合

### 5.5 生成等价类流程

**MergeJoinable 条件**：

- 非常量表达式
- 操作符表达式（OpExpr）
- 两个参数
- 操作符 MergeJoinable（`oprcanmerge == true`）
- 不含易失性函数

**process_equivalence 四种情况**：

1. 两端表达式在同一等价类 → 补充 RestrictInfo 信息
2. 两端表达式在不同等价类 → 合并等价类
3. 一端找到，另一端未找到 → 加入找到的等价类
4. 两端都未找到 → 创建新的等价类

---

## 六、优化技术

### 6.1 PlaceHolderVar

**作用**：在子查询提升时，对不严格的表达式进行封装，保证逻辑等价。

**使用场景**：

- 外连接 Nullable-side 的不严格表达式
- AppendRel 中的表达式
- Lateral 子查询引用上层表的属性

**结构体**：

```c
typedef struct PlaceHolderVar {
    Expr *phexpr;       // 被封装的Var或表达式
    Relids phrels;      // 子查询的表
    Index phid;         // 唯一ID
    Index phlevelsup;   // 层次关系
} PlaceHolderVar;
```

### 6.2 消除无用连接项

**条件**：

1. 必须是左连接，且内表是基表
2. 其他位置不出现内表的任何列
3. 连接条件中内表的列具有唯一性

**唯一性判断**（rel_is_distinct_for）：

- 主键索引或UNIQUE索引
- DISTINCT子句
- GROUP BY子句
- 空GROUP BY（返回一行）
- 聚集函数（无GROUP BY）
- UNION/EXCEPT/INTERSECT（无ALL）

### 6.3 Semi Join 消除

**条件**：内表保证唯一性时，Semi Join可转换为Inner Join

**判断流程**：

```
reduce_unique_semijoins → is_innerrel_unique_for → rel_is_distinct_for
```

### 6.4 提取新的约束条件

**场景**：`(sno=1 AND cname='math') OR (sno=2 AND cname='physics')`

**提取结果**：

- `sno = 1 OR sno = 2` → 下推到STUDENT表
- `cname = 'math' OR cname = 'physics'` → 下推到COURSE表

**提取条件**：

- 每个子句引用同一基表
- 符合谓词下推规则
- 不含易失性函数
- 非常量约束条件

**选择率修正**：

```
修正后选择率 = 原选择率 / 新提取条件选择率
```

---

## 七、Lateral 语法支持

### 7.1 语义分析

- 在 `gram.y` 中添加 `LATERAL` 语法支持
- 设置 `ParseState->p_lateral_active` 标记
- 允许子查询引用上层表的属性

### 7.2 收集 Lateral 变量

**find_lateral_references 函数**：

- 遍历 RelOptInfo 数组
- 查找 Lateral 列属性
- 保存到 `RelOptInfo->lateral_vars`

### 7.3 收集 Lateral 信息

**create_lateral_join_info 函数**：

- `direct_lateral_relids`：直接观察到的依赖关系
- `lateral_relids`：传递闭包推理出的依赖关系
- `lateral_vars`：Lateral 变量列表
- `lateral_referencers`：被依赖关系（反向）

---

## 八、执行计划示例解析

### 示例1：等价类推理

```sql
SELECT * FROM STUDENT,SCORE,COURSE 
WHERE COURSE.cno=STUDENT.sno AND STUDENT.sno=SCORE.sno;
```

**推理**：COURSE.cno=STUDENT.sno AND STUDENT.sno=SCORE.sno → COURSE.cno=SCORE.sno

**执行计划**：

```
Hash Join (cost=30.35..68.53 rows=71 width=69)
  Hash Cond: (score.sno = course.cno)  -- 推理出的新条件
```

### 示例2：连接条件下推

```sql
SELECT * FROM STUDENT LEFT JOIN SCORE ON STUDENT.sno = 1;
```

**分析**：STUDENT.sno=1 引用 Nonnullable-side，不能下推

**执行计划**：

```
Nested Loop Left Join
  Join Filter: (student.sno = 1)  -- 保持为连接条件
```

### 示例3：无用连接消除

```sql
SELECT SCORE.sno FROM SCORE LEFT JOIN STUDENT ON SCORE.sno = STUDENT.sno;
```

**分析**：STUDENT表无其他引用，且STUDENT.sno有主键唯一性

**执行计划**：

```
Seq Scan on score  -- STUDENT被消除
```

---

## 九、本章小结

1. **核心转换**：RangeTblEntry → RelOptInfo，从逻辑层到物理层
2. **等价类推理**：基于等值条件生成隐含约束条件，扩充连接路径
3. **谓词下推**：根据连接类型（Left/Semi/Anti）决定约束条件是否可下推
4. **连接顺序交换**：遵循等价公式，受外连接/半连接限制
5. **优化技术**：PlaceHolderVar、无用连接消除、Semi Join消除、OR条件提取

**难点**：外连接/半连接/反半连接引入后，经典关系代数理论不再完全适用，需要深入理解连接语义才能掌握谓词下推和顺序交换的规则。

---