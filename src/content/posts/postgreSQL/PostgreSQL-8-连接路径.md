---
title: 第八章-连接路径
published: 2026-06-16 07:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
## 一句话概括

本章详细讲解了PostgreSQL查询优化器中物理连接路径的完整建立过程，包括连接合法性检查、三种经典连接方法（NestloopJoin、HashJoin、MergeJoin）的路径生成与代价计算，以及路径的预筛选与最终筛选机制。

---

## 书籍骨架

### 结构拆解

| 小节 | 主题 | 核心内容 |
|------|------|----------|
| 8.1 | 检查 | 连接路径生成前的三重检查：初步检查、精确检查、合法性检查 |
| 8.2 | 生成新的RelOptInfo | 通过build_join_rel函数创建父RelOptInfo，管理约束条件 |
| 8.3 | 虚表 | 常量false约束条件导致的空结果集优化 |
| 8.4 | Semi Join和唯一化路径 | Semi Join内表的唯一化处理及其代价考量 |
| 8.5 | 建立连接路径 | 三种物理连接方法的路径生成与代价计算 |
| 8.6 | 路径的筛选 | 预筛选（初级代价）与最终筛选（add_path）的比较机制 |

---

## 详细内容笔记

### 8.1 检查

在动态规划方法中，将N-1层的每个RelOptInfo和第1层的每个RelOptInfo尝试做连接，时间复杂度为O(M×N)。为避免无效搜索，通过检查机制进行剪枝。

#### 8.1.1 初步检查（基于单个RelOptInfo的"可能性"判断）

判断条件（满足任一即进入精确检查）：
1. `RelOptInfo->joininfo != NULL` → 有连接条件
2. `RelOptInfo->has_eclass_joins == true` → 等价类中有等值连接条件
3. `has_join_restriction()` 返回true → 有连接顺序限制

`has_join_restriction`的判断依据：
- `lateral_relids` 或 `lateral_referencers` 不为NULL（涉及Lateral）
- `PlaceHolderVar` 必须在下层求值
- `SpecialJoinInfo` 的 `min_lefthand` 和 `min_righthand` 与当前RelOptInfo有交集

#### 8.1.2 精确检查（基于两个RelOptInfo的真实关系）

- **have_relevant_joinclause**：判断两个RelOptInfo是否有连接条件
    - `joininfo` 中的条件引用了另一个RelOptInfo
    - 两个RelOptInfo的relids在同一个等价类中

- **have_join_order_restriction**：判断两个RelOptInfo是否有连接顺序限制
    - Lateral依赖关系
    - PlaceHolderVar
    - SpecialJoinInfo的min_lefthand/min_righthand限制

#### 8.1.3 "合法"连接

分两步检查：

**步骤1：合法的连接关系**
- 遍历`PlannerInfo->join_info_list`中的SpecialJoinInfo
- 通过最小集合（min_lefthand、min_righthand）判断连接顺序
- 支持的连接类型：LeftJoin、FullJoin、SemiJoin、AntiJoin
- SemiJoin可尝试唯一化处理（RHS去重）
- 特殊情况处理：`A leftjoin (B innerjoin C) left join D` 的连接顺序交换

**步骤2：合法的连接顺序**
- 两个RelOptInfo之间不能互相依赖（无环）
- Lateral语义的单向依赖不能与SpecialJoinInfo冲突
- SemiJoin唯一化路径与Lateral的冲突检查
- FullJoin不能用NestloopJoin实现的限制
- direct_lateral_relids必须有直接依赖关系
- NestLoopParam只能传递简单Var，PlaceHolderVar可能有风险
- Lateral变量引用外连接Nullable-side的表的风险排除

---

### 8.2 生成新的RelOptInfo

`build_join_rel`函数的核心逻辑：

1. **查找已存在的RelOptInfo**：
    - 先查`PlannerInfo->join_rel_list`（O(N)遍历）
    - 若超过32个，构建`PlannerInfo->join_rel_hash`哈希表加速查找

2. **约束条件处理**：
    - `build_joinrel_restrictlist`：筛选连接路径的约束条件
    - `build_joinrel_joinlist`：筛选适用于所有路径的约束条件
    - 约束条件保存在`JoinPath->joinrestrictinfo`中（非RelOptInfo中）

3. **PlaceHolderVar处理**：
    - 不直接加入`reltarget`
    - 通过`bms_nonempty_difference`和`bms_is_subset`双重检查判断是否需要加入

---

### 8.3 虚表

当约束条件为常量false时，连接结果可能为空集。

| 连接类型 | 优化方式 |
|----------|----------|
| Inner Join | 内表、外表或约束条件中有常量false → 虚表 |
| Left Join | 外表为虚表 → 整体为虚表；内表为虚表 → 内表设为虚表 |
| Full Join | 内外表均为虚表 → 虚表；有常量false约束 → 虚表 |
| Semi Join | 与内连接相似 |
| Anti Join | 与左连接相似 |

关键点：
- `restriction_is_constant_false`的`only_pushed_down`参数区分检查范围
- 外连接中，ON FALSE对Nullable-side的表效果等同于虚表
- 使用`syn_righthand`而非`min_righthand`判断Nullable-side

---

### 8.4 Semi Join和唯一化路径

**唯一化的价值**：
- Semi Join内表唯一化后，可转换为Inner Join
- 转换后连接顺序交换更自由，路径建立空间更大

**隐式唯一化条件**：
- 普通表：有Unique索引，约束条件与索引匹配
    - 必须是唯一索引、immediate=true、谓词匹配、键匹配
- 子查询：有DISTINCT、GROUP BY（无Grouping Sets）、聚合函数、UNION等

**显式唯一化方法**：
- Sort去重（semi_can_btree == true）
- Hash去重（semi_can_hash == true）
- 需要考虑唯一化带来的额外代价

**代价因子计算**（`compute_semi_anti_join_factors`）：
- `nselec`：Inner Join下的选择率
- `jselec`：Semi Join下的选择率
- `avgmatch = (X/Y) × n`：外表一条元组平均匹配内表的元组数

---

### 8.5 建立连接路径

#### 8.5.1 sort_inner_and_outer函数（MergeJoin路径）

**路径选择**：
- 外表和内表均选择`cheapest_total_path`
- 并行路径取决于外表的并行度

**PathKeys生成流程**（`select_outer_pathkeys_for_merge`）：
1. 根据MergeJoinable连接条件生成等价类数组并打分
    - 打分标准：等价类成员与多少个其他RelOptInfo相关
2. 匹配`PlannerInfo->query_pathkeys`（Non-SPJ优化阶段的期望）
3. 按分数排序生成初始PathKeys

**不同顺序的约束条件**：
- 启发式规则：每个PathKey都有机会作为第一个，后续顺序不变
- 示例：4个约束条件只需考虑4种PathKeys顺序

**MergeJoin代价计算**：
- **初级代价**：预筛选，计算排序代价
    - 外排序（output > sort_mem）
    - 堆排序（input > sort_mem, output < sort_mem）
    - 快速排序（全部在内存）
- **最终代价**：
    - 计算重复扫描元组数（rescannedtuples）
    - 物化决策（materialize_inner）
    - 表达式代价计算

#### 8.5.2 match_unsorted_outer函数（外表已有序）

**特点**：
- 外表路径已有有序的PathKeys
- 只对内表显式排序
- 可利用B树索引扫描路径（节省排序时间）

**生成的路径类型**：
- MergeJoin路径（内表排序）
- NestloopJoin路径（内表考虑最低代价、参数化、物化三种情况）
- 并行NestloopJoin和MergeJoin路径

**NestloopJoin代价计算**：
- 时间复杂度：O(m×n)
- 第一次扫描 vs 后续扫描的代价差异（`cost_rescan`）
- SemiJoin/AntiJoin/UniquePath的特殊处理：
    - `outer_match_frac`：外表能匹配的元组比例
    - `match_count`：匹配一条需要扫描的内表元组数

#### 8.5.3 hash_inner_and_outer函数（HashJoin路径）

**路径选择**：
- 外表：最低启动代价路径 + 最低总代价路径
- 内表：最低总代价路径（哈希表重建使启动代价无意义）

**并行HashJoin的两种模式**：
1. **parallel-oblivious**：每个工作进程自建哈希表，parallel_aware=false
2. **parallel-aware**：共享哈希表，parallel_aware=true

**代价计算**（三部分）：
1. 内表总代价 + 外表启动代价 → 启动代价
2. 创建哈希表代价（内表） + 匹配检测代价（外表）
3. 多batch情况：内外表数据写入/读取代价

---

### 8.6 路径的筛选

#### 预筛选（add_path_precheck）

- 使用初级代价与`RelOptInfo->pathlist`中所有路径比较
- 条件：总代价差 + 启动代价差 + PathKeys差 + 参数生产者相同
- 并行路径预筛选：`add_partial_path_precheck`

#### 最终筛选（add_path）

5个判断因素：
1. **代价**（总代价、启动代价）：`compare_path_costs_fuzzily`，STD_FUZZ_FACTOR=1.01
2. **PathKeys**：`compare_pathkeys`，逐个PathKey比较
3. **参数化路径**：`PATH_REQ_OUTER`比较
4. **结果集行数**：行数少的更优
5. **并行安全性**：parallel_safe高的更优

筛选结果：
- `COSTS_BETTER1`：新路径优于旧路径
- `COSTS_BETTER2`：旧路径优于新路径
- `COSTS_EQUAL`：代价相似，比较其他因素
- `COSTS_DIFFERENT`：互相不占优势

---

## 核心概念

| 概念 | 定义 | 说明 |
|------|------|------|
| 连接路径 | 物理连接路径，实现逻辑连接操作 | NestloopJoin、HashJoin、MergeJoin三种 |
| 初步检查 | 基于单个RelOptInfo的可能性判断 | 决定进入精确检查还是无条件连接 |
| 精确检查 | 基于两个RelOptInfo的真实关系判断 | have_relevant_joinclause + have_join_order_restriction |
| 合法性检查 | 确认连接关系和连接顺序的合法性 | SpecialJoinInfo + Lateral语义 |
| 虚表 | 约束条件为常量false的RelOptInfo | 连接结果为空集的优化 |
| 唯一化路径 | Semi Join内表去重后的路径 | 可将Semi Join转为Inner Join |
| PathKeys | 路径结果的排序提示 | 用于MergeJoin的排序匹配和上层操作优化 |
| 预筛选 | 基于初级代价的路径创建前筛选 | 节省查询优化时间 |
| 最终筛选 | 基于最终代价的路径选择与淘汰 | add_path中的多因素综合比较 |

---

## 关键函数调用链

```
join_search_one_level
├── make_rels_by_clause_joins（有连接条件）
├── make_rels_by_clauseless_joins（无连接条件）
└── make_join_rel（生成父RelOptInfo）
    └── populate_joinrel_with_paths
        └── add_paths_to_joinrel
            ├── sort_inner_and_outer（MergeJoin，内外表都排序）
            │   ├── try_mergejoin_path
            │   └── try_partial_mergejoin_path
            ├── match_unsorted_outer（MergeJoin，外表已有序）
            │   ├── try_nestloop_path
            │   ├── generate_mergejoin_paths
            │   └── consider_parallel_nestloop
            └── hash_inner_and_outer（HashJoin）
                ├── try_hashjoin_path
                └── try_partial_hashjoin_path
```

---

## 与前后章节的关系

- **前置知识**：第4章（连接顺序交换等式）、第7章（动态规划/遗传算法）
- **本章核心**：从`join_search_one_level`和`merge_clump`函数开始，进入物理连接路径建立阶段
- **后续影响**：生成的连接路径将用于执行计划的最终选择

---

## 个人思考

1. **合法性检查的复杂性**：作者坦言有些情况PostgreSQL开发人员也无法给出示例，说明连接合法性判断的复杂程度
2. **代价模型的分阶段设计**：初级代价用于预筛选，最终代价用于精确选择，这种两阶段设计在优化时间和执行质量间取得平衡
3. **虚表优化的实用性**：看似简单的常量false判断，在外连接场景下需要区分可下推的过滤条件和连接条件
4. **Semi Join唯一化的权衡**：唯一化增加代价但带来更大的优化空间，是典型的收支考量
