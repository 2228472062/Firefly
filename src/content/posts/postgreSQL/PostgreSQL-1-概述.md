---
title: 第一章-概述
published: 2026-06-15 00:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
## 一、章节概述

本章是查询优化器的基础章节，介绍了SQL查询语句在经过词法分析、语法分析和语义分析后形成的**查询树（Query Tree）**。查询树对应一个关系代数表达式，是查询优化器的输入，贯穿整个查询优化过程。

---

## 二、核心数据结构详解

### 2.1 Node 结构体（所有节点的基础）

**作用**：PostgreSQL中所有数据结构的基础，采用统一的"类继承"方式实现。

```c
typedef struct Node {
    NodeTag type;  // 枚举类型，标识节点具体类型
} Node;
```

**关键点**：

- 所有结构体的第一个成员变量都是 `NodeTag`
- 通过 `Node*` 指针可以统一表示任何结构体
- 通过查看 `NodeTag` 枚举值来区分实际类型

**示例**：

- `List` 结构体：`type` 可能是 `T_List`、`T_IntList`、`T_OidList`
- `Query` 结构体：`type` 值为 `T_Query`

---

### 2.2 Var 结构体（列属性表示）

**作用**：表示查询中涉及的表的列属性（投影列、约束条件中的列）。

| 成员变量      | 含义                                    | 来源                       |
| ------------- | --------------------------------------- | -------------------------- |
| `varno`       | 列属性所在的表在 `rtable` 中的编号      | `Query->rtable` 的 rtindex |
| `varattno`    | 列属性是表中的第几列                    | `PG_ATTRIBUTES` 系统表     |
| `vartype`     | 列属性的类型（如 INT=23, VARCHAR=1043） | `PG_ATTRIBUTES` 系统表     |
| `vartypmod`   | 列属性的精度（如 `VARCHAR(10)` → 14）   | `PG_ATTRIBUTES` 系统表     |
| `varlevelsup` | 列属性对应表所在的层次（相对值）        | 用于子查询场景             |
| `varnoold`    | `varno` 的初值（等价变换前）            | 记录原始值                 |
| `varoattno`   | `varattno` 的初值（等价变换前）         | 记录原始值                 |
| `location`    | 列属性出现在 SQL 语句中的位置           | 语法分析阶段               |

**子查询示例**：

```sql
SELECT st.sname FROM STUDENT st 
WHERE st.sno = ANY(SELECT sno FROM SCORE WHERE st.sno = sno);
```

| 列属性              | varno | varattno | vartype        | varlevelsup | 说明                |
| ------------------- | ----- | -------- | -------------- | ----------- | ------------------- |
| `st.sno` (子查询中) | 1     | 1        | 23 (INT)       | 1           | 父查询表的列，上1层 |
| `SCORE.sno`         | 1     | 1        | 23 (INT)       | 0           | 当前层次表的列      |
| `st.sname`          | 1     | 2        | 1043 (VARCHAR) | 0           | 当前层次表的列      |

---

### 2.3 RangeTblEntry 结构体（范围表）

**作用**：描述查询中出现的表（FROM子句），包括普通表、子查询、连接表等。

**RTEKind 枚举类型**：

| 类型           | 含义                  | 对应的 FROM 子句部分      |
| -------------- | --------------------- | ------------------------- |
| `RTE_RELATION` | 普通堆表              | `FROM STUDENT`            |
| `RTE_SUBQUERY` | 子查询                | `(SELECT * FROM TEACHER)` |
| `RTE_JOIN`     | JOIN 产生的表         | `STUDENT LEFT JOIN SCORE` |
| `RTE_FUNCTION` | 表函数返回的表        | `GENERATE_SERIES(1,10)`   |
| `RTE_VALUES`   | VALUES 产生的表       | `VALUES(1,1)`             |
| `RTE_CTE`      | WITH 语句附带的公共表 | CTE 子句                  |

**结构体关键成员**：

```c
typedef struct RangeTblEntry {
    NodeTag type;
    RTEKind rtekind;          // 范围表类型
    // For RTE_RELATION
    Oid relid;                // 表的 OID，来自 PG_CLASS
    char relkind;             // 表的类型，来自 PG_CLASS
    // For RTE_SUBQUERY
    Query *subquery;          // 子查询的查询树
    // For RTE_JOIN
    JoinType jointype;        // JOIN 的类型
    List *joinaliasvars;      // JOIN 的表的所有列的集合
} RangeTblEntry;
```

**示例**：

```sql
SELECT * FROM STUDENT LEFT JOIN SCORE ON TRUE, 
       (SELECT * FROM TEACHER) AS t, 
       COURSE, 
       (VALUES(1,1)) AS NUM(x, y), 
       GENERATE_SERIES(1,10) AS GS(z);
```

| RangeTblEntry 类型 | 对应部分                        |
| ------------------ | ------------------------------- |
| RTE_RELATION       | COURSE                          |
| RTE_JOIN           | STUDENT LEFT JOIN SCORE ON TRUE |
| RTE_SUBQUERY       | (SELECT * FROM TEACHER)         |
| RTE_FUNCTION       | GENERATE_SERIES(1,10)           |
| RTE_VALUES         | (VALUES(1,2))                   |

---

### 2.4 RangeTblRef 结构体（范围表引用）

**作用**：在 `jointree` 中引用 `rtable` 中的 `RangeTblEntry`，避免数据冗余。

```c
typedef struct RangeTblRef {
    NodeTag type;
    int rtindex;  // RangeTblEntry 在 Query->rtable 中的位置
} RangeTblRef;
```

**设计原则**：

- 每个范围表在 `Query->rtable` 中有且仅有一个
- 其他地方使用 `RangeTblRef` 引用，不重复存储

---

### 2.5 JoinExpr 结构体（显式连接）

**作用**：表示显式指定的连接关系（如 `A LEFT JOIN B ON Pab`）。

```c
typedef struct JoinExpr {
    NodeTag type;
    JoinType jointype;      // 连接类型（LeftJoin, InnerJoin 等）
    bool isNatural;         // 是否是自然连接
    Node *larg;             // 左侧表（LHS）
    Node *rarg;             // 右侧表（RHS）
    List *usingClause;      // USING 子句的约束条件
    Node *quals;            // ON 子句的约束条件
    Alias *alias;           // 连接操作的投影列
    int rtindex;            // 对应的 RangeTblRef->rtindex
} JoinExpr;
```

**示例分析**：

1. `SELECT * FROM STUDENT INNER JOIN SCORE ON STUDENT.sno = SCORE.sno;`
    - 使用 `JoinExpr` 表示
    - `quals` 保存 `STUDENT.sno = SCORE.sno`

2. `SELECT * FROM STUDENT LEFT JOIN SCORE ON TRUE LEFT JOIN COURSE ON SCORE.cno = COURSE.cno;`
    - 需要**两个** `JoinExpr` 表示
    - 形成树状结构

---

### 2.6 FromExpr 结构体（隐式连接）

**作用**：表示表之间的隐式连接关系（默认为 Inner Join）。

```c
typedef struct FromExpr {
    NodeTag type;
    List *fromlist;    // FromExpr 中包含的表
    Node *quals;       // 表之间的约束条件
} FromExpr;
```

**示例**：

```sql
SELECT * FROM STUDENT, SCORE, COURSE WHERE STUDENT.sno = SCORE.sno;
```

- `fromlist`：3 个 `RangeTblRef` 节点
- `quals`：`STUDENT.sno = SCORE.sno`

---

### 2.7 Query 结构体（查询树核心）

**作用**：查询优化模块的输入参数，表示完整的查询树。

**关键成员变量**：

| 成员变量         | 含义                                              |
| ---------------- | ------------------------------------------------- |
| `commandType`    | 语句类型：SELECT, INSERT, UPDATE, DELETE, UTILITY |
| `rtable`         | 语句涉及的表清单（List of RangeTblEntry）         |
| `jointree`       | 表的连接关系及约束关系（FromExpr 类型）           |
| `targetList`     | 需要投影的列（SELECT 子句中的列）                 |
| `hasAggs`        | 是否有聚集操作                                    |
| `hasWindowFuncs` | 是否有窗口函数                                    |
| `hasSubLinks`    | 是否包含 SubLink 或 SubQuery                      |
| `groupClause`    | GROUP BY 子句                                     |
| `havingQual`     | HAVING 子句                                       |
| `sortClause`     | ORDER BY 子句                                     |
| `limitOffset`    | OFFSET 子句                                       |
| `limitCount`     | LIMIT 子句                                        |
| `setOperations`  | UNION/INTERSECT/EXCEPT 操作                       |

---

## 三、查询树示例分析

### 示例 1：简单查询

```sql
SELECT * FROM STUDENT WHERE SNO = 1;
```

| 成员变量     | 内容                                      |
| ------------ | ----------------------------------------- |
| `rtable`     | 1 个 `RangeTblEntry`（STUDENT表）         |
| `jointree`   | FromExpr: fromlist 有 1 个 RangeTblRef    |
|              | FromExpr: quals 有 1 个 OPExpr（SNO = 1） |
| `targetlist` | 3 个 TargetEntry（sno, sname, ssex）      |

### 示例 2：两表连接

```sql
SELECT st.sname, sc.degree 
FROM STUDENT st, SCORE sc 
WHERE st.sno = sc.sno;
```

| 成员变量     | 内容                                              |
| ------------ | ------------------------------------------------- |
| `rtable`     | 2 个 `RangeTblEntry`（STUDENT, SCORE）            |
| `jointree`   | FromExpr: fromlist 有 2 个 RangeTblRef            |
|              | FromExpr: quals 有 1 个 OPExpr（st.sno = sc.sno） |
| `targetlist` | 2 个 TargetEntry（sname, degree）                 |

### 示例 3：显式 INNER JOIN

```sql
SELECT st.sname, sc.degree 
FROM STUDENT st INNER JOIN SCORE sc ON st.sno = sc.sno;
```

| 成员变量     | 内容                                             |
| ------------ | ------------------------------------------------ |
| `rtable`     | 3 个 `RangeTblEntry`（STUDENT, SCORE, 连接关系） |
| `jointree`   | FromExpr: fromlist 有 1 个 JoinExpr              |
|              | JoinExpr: larg = STUDENT, rarg = SCORE           |
|              | JoinExpr: quals = st.sno = sc.sno                |
|              | FromExpr: quals = NULL                           |
| `targetlist` | 2 个 TargetEntry（sname, degree）                |

---

## 四、查询树的展示与调试

### GUC 参数

| 参数名                  | 默认值 | 描述                   |
| ----------------------- | ------ | ---------------------- |
| `debug_pretty_print`    | On     | 以结构化方式展示查询树 |
| `debug_print_parse`     | Off    | 打印查询树             |
| `debug_print_rewritten` | Off    | 打印重写后的查询树     |
| `debug_print_plan`      | Off    | 打印执行计划           |

### 调试技巧

- 在逻辑优化前后使用 `elog_node_display` 函数打印查询树
- 对比重写前后的查询树，分析优化效果

---

## 五、查询树的遍历

**遍历函数**：

- `query_tree_walker`：遍历查询树，可修改值，但不增删节点
- `query_tree_mutator`：遍历查询树，可增删节点

**应用场景**：

- 子查询提升时调整 `varlevelsup` 值
- 查找、替换、编辑特定结构体

---

## 六、执行计划展示（EXPLAIN）

### EXPLAIN 语法

```sql
EXPLAIN [ ( option [, ...] ) ] statement
EXPLAIN [ ANALYZE ] [ VERBOSE ] statement
```

### 选项说明

| 选项      | 说明                               |
| --------- | ---------------------------------- |
| `VERBOSE` | 打印更详细的信息                   |
| `ANALYZE` | 打印实际执行代价（会真正执行语句） |
| `BUFFERS` | 与 ANALYZE 配合，打印缓冲区命中率  |
| `COSTS`   | 是否打印代价（默认开启）           |
| `TIMING`  | 与 ANALYZE 配合，是否统计时间      |

### 示例

```sql
EXPLAIN SELECT * FROM STUDENT, (SELECT * FROM SCORE) as sc;
```

```
Nested Loop (cost=0.00..2.15 rows=7 width=31)
  -> Seq Scan on score (cost=0.00..1.01 rows=1 width=12)
  -> Seq Scan on student (cost=0.00..1.07 rows=7 width=19)
```

**代价说明**：

- `cost=0.00..2.15`：0.00 是启动代价，2.15 是整体代价
- 执行计划是一棵**非完全二叉树**

---

## 七、关键要点总结

1. **统一节点模型**：所有结构体基于 `Node` 扩展，通过 `NodeTag` 识别类型

2. **查询树核心组件**：

    - `rtable`：存储所有范围表
    - `jointree`：存储表之间的连接关系
    - `targetlist`：存储投影列

3. **范围表引用**：使用 `RangeTblRef` 避免数据冗余

4. **连接表示**：

    - `JoinExpr`：显式连接（INNER JOIN, LEFT JOIN 等）
    - `FromExpr`：隐式连接（逗号分隔，默认 INNER JOIN）

5. **查询优化流程**：

   ```
   SQL → 词法分析 → 语法分析 → 语义分析 → 查询树 → 逻辑优化 → 物理优化 → 执行计划
   ```

---

## 八、学习建议

1. **动手实践**：使用 `debug_print_parse` 等参数查看实际查询树
2. **对比分析**：修改查询语句，观察查询树的变化
3. **源码阅读**：结合 `query_tree_walker` 和 `query_tree_mutator` 理解遍历机制
4. **联系实际**：通过 EXPLAIN 分析执行计划，理解优化器决策

---
