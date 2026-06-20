---
title: 第五章-统计信息与选择率
published: 2026-06-16 04:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
### 5.1 统计信息

#### 5.1.1 PG_STATISTIC系统表

**统计信息的作用**：

- 物理优化需要计算各种物理路径的代价
- 代价估算严重依赖于统计信息
- 统计信息是否能准确描述表中的数据分布是决定代价准确性的重要条件

**单列统计信息类型（7种）**：

| 类型                                  | 说明                                                         |
| ------------------------------------- | ------------------------------------------------------------ |
| STATISTIC_KIND_MCV                    | 高频值（常见值），在一个列里出现最频繁的值，按频率排序，生成一一对应的频率数组 |
| STATISTIC_KIND_HISTOGRAM              | 直方图，使用等频直方图描述数据分布，高频值不出现，保证数据分布相对平坦 |
| STATISTIC_KIND_CORRELATION            | 相关系数，记录未排序数据分布和排序后数据分布的相关性，用于索引扫描代价估计 |
| STATISTIC_KIND_MCELEM                 | 类型高频值，用于数组类型或特殊类型，由ts_typanalyze函数生成  |
| STATISTIC_KIND_DECHIST                | 数组类型直方图，由array_typanalyze函数生成                   |
| STATISTIC_KIND_RANGE_LENGTH_HISTOGRAM | Range类型基于长度的直方图，由range_typanalyze函数生成        |
| STATISTIC_KIND_BOUNDS_HISTOGRAM       | Range类型基于边界的直方图，由range_typanalyze函数生成        |

**多列统计信息类型（2种）**：

| 类型                   | 说明                                                  |
| ---------------------- | ----------------------------------------------------- |
| STATS_EXT_NDISTINCT    | 记录基于多列的消重之后的数据量，类似单列的stadistinct |
| STATS_EXT_DEPENDENCIES | 计算各个列之间的函数依赖度，得到准确的统计信息        |

**选择率示例**：

```sql
-- 高选择率，采用顺序扫描
EXPLAIN SELECT * FROM STUDENT WHERE sno > 2;
-- 结果：Seq Scan on student (cost=0.00..224.00 rows=9999 width=10)

-- 低选择率，采用索引扫描
EXPLAIN SELECT * FROM STUDENT WHERE sno < 2;
-- 结果：Index Scan using student_pkey on student (cost=0.29..8.30 rows=1 width=10)
```

**顺序读vs随机读**：

```c
#define DEFAULT_SEQ_PAGE_COST 1.0    // 顺序读的单页代价
#define DEFAULT_RANDOM_PAGE_COST 4.0 // 随机读的单页代价
```

**PG_STATISTIC系统表结构**：

```sql
-- 基本字段
starelid    -- 对应表的OID（来自PG_CLASS）
staattnum   -- 列的编号（来自PG_ATTRIBUTES）
stanullfrac -- NULL值率，NULL值所占比例
stawidth    -- 列的平均宽度
stadistinct -- 消重之后的数据个数或比例
```

**stadistinct取值**：

- = 0：代表未知或者未计算的情况

- > 0：代表消除重复值之后的个数

- < 0：绝对值是去重之后的个数占总个数的比例（通常使用）

**示例分析**：

```sql
-- STUDENT表统计信息
starelid | staattnum | stanullfrac | stawidth | stadistinct
---------+-----------+-------------+----------+-------------
24582    | 1         | 0           | 4        | -1          -- sno列
24582    | 2         | 0.142857    | 3        | -0.571429   -- sname列
24582    | 3         | 0.142857    | 4        | -0.285714   -- ssex列
```

**槽位系统**：

- 每个属性最多支持5种统计方法（STATISTIC_NUM_SLOTS = 5）
- 每个槽位包含：stakind、staop、stanumbers、stavalues
- 不同列使用不同数量的槽位

**槽位示例**：

```sql
-- 第一个槽位
starelid | staattnum | stakind1 | staop1 | stanumbers1          | stavalues1
---------+-----------+----------+--------+----------------------+------------------
24582    | 1         | 2        | 97     |                      | {1,2,3,4,5,6,7}  -- sno直方图
24582    | 2         | 1        | 98     | {0.285714,0.285714}  | {ls,zs}           -- sname MCV
24582    | 3         | 1        | 96     | {0.571429,0.285714}  | {1,2}             -- ssex MCV
```

#### 5.1.2 PG_STATISTIC_EXT系统表

**创建多列统计信息**：

```sql
CREATE STATISTICS STATEXT_STUDENT ON sno, sname, ssex FROM STUDENT;
```

**系统表结构**：

| 属性            | 类型            | 说明                                        |
| --------------- | --------------- | ------------------------------------------- |
| stxrelid        | Oid             | 统计信息属于哪个表                          |
| stxname         | NameData        | 统计信息的名字                              |
| stxnamespace    | Oid             | 统计信息的namespace                         |
| stxowner        | Oid             | 统计信息的创建者                            |
| stxkeys         | int2vector      | 统计哪些列                                  |
| stxkind         | char            | 统计信息类型（d=NDISTINCT, f=DEPENDENCIES） |
| stxndistinct    | pg_ndistinct    | NDISTINCT类型的统计信息                     |
| stxdependencies | pg_dependencies | DEPENDENCIES类型的统计信息                  |

**示例**：

```sql
CREATE STATISTICS STATEXT_STUDENT ON sno, sname, ssex FROM STUDENT;
ANALYZE STUDENT(sno, sname, ssex);

-- 查看统计信息
SELECT * FROM pg_statistic_ext WHERE stxname='statext_student';
-- 结果包含NDISTINCT和DEPENDENCIES信息
```

#### 5.1.3 单列统计信息生成

**生成步骤**：

1. 数据采样（acquire_sample_rows函数）
2. 数据统计（compute_scalar_stats函数等）

**采样算法**：

- **第一阶段**：S算法（选择抽样技术）对页面进行随机采样
- **第二阶段**：Z(Vitter)算法（蓄水池抽样）对元组进行采样

**S算法（简化版）**：

```
从N个记录中随机选取n个记录（0 < n <= N）
S1. [初始化] K = N - t, k = n - m
S2. [生成随机数] 0 <= V <= 1
S3. [生成跳过条件] p = 1 - k/K
S4. [检测] 如果 V < p，跳转至S6
S5. [纳入样本] m++, t++, 如果 m < n 跳转至S2
S6. [跳过] t++, K--, p *= 1 - k/K, 跳转至S4
```

**Vitter算法改进**：

1. 随机数生成算法改进
2. 通过随机变量跳过记录，避免O(N)复杂度
3. 使用临界点（22.0 × n）在X算法和Z算法之间切换

**采样框架**：

```c
acquire_sample_rows {
    BlockSampler_Init;           // S算法赋初值
    reservoir_init_selection_state; // Z算法赋初值
    while(BlockSampler_HasMore) {
        block = BlockSampler_Next; // S算法核心
        foreach(tuple in block) {
            if(蓄水池未满) {
                记录TUPLE到蓄水池;
            }
            if(rowstoskip < 0) { // Z算法中的随机变量T
                rowstoskip = reservoir_get_next_S;
                if(rowstoskip <= 0) {
                    随机替换蓄水池中的元组;
                }
            }
        }
    }
}
```

**元组密度估算（EMA方法）**：

```c
old_density = old_rel_tuples / old_rel_pages;
new_density = scanned_tuples / scanned_pages;
multiplier = (double) scanned_pages / (double) total_pages;
updated_density = old_density + (new_density - old_density) * multiplier;
reltuples = floor(updated_density * total_pages + 0.5);
```

**采样相关变量**：

| 变量           | 说明                             |
| -------------- | -------------------------------- |
| old_rel_pages  | 目前采样的表中的总页面数         |
| old_rel_tuples | 目前采样的表中的总元组数         |
| scanned_pages  | 第一阶段采样获得的页面数         |
| scanned_tuples | 第二阶段采样获得的有效元组数     |
| total_pages    | 第一阶段采样时获得的实际总页面数 |

**统计方法选择**：

```c
if (OidIsValid(eqopr) && OidIsValid(ltopr)) {
    stats->compute_stats = compute_scalar_stats;  // 有等值和不等值操作符
} else if (OidIsValid(eqopr)) {
    stats->compute_stats = compute_distinct_stats; // 只有等值操作符
} else {
    stats->compute_stats = compute_trivial_stats;  // 其他情况
}
```

**compute_scalar_stats函数处理流程**：

以sname列为例：

```
初始状态：
samplerows = 7    -- 样本容量
null_cnt = 1      -- NULL值个数
nonnull_cnt = 6   -- 非NULL值个数
toowide_cnt = 0   -- 超长值的个数
total_width = 18  -- 非NULL值的长度总和

排序后：
ndistinct = 4     -- 去掉重复值和NULL值之后的个数
stanullfrac = 1/7 -- NULL值比例
stdwidth = 3      -- 平均宽度
stadistinct = 4/7 -- 去重比例
```

**MCV（高频值）确定**：

- 情况1：所有值都可以做MCV值（满足条件时）
- 情况2：部分值做MCV值（不满足条件时）

**直方图生成**：

- 使用等频直方图
- 桶数计算：num_hist = ndistinct - num_mcv
- 边界值确定：通过排序后的值均匀分布

**相关系数计算**：

```c
// 排序前编号：0, 1, 2, 3, 4, 5
// 排序后编号：1, 5, 2, 3, 0, 4
// 相关系数公式：ρ = (n * ∑xd - ∑x * ∑d) / (√(n * ∑x² - (∑x)²) * √(n * ∑d² - (∑d)²))
```

#### 5.1.4 多列统计信息生成

**STATS_EXT_NDISTINCT类型**：

- 记录多列组合的去重数据量
- 例如：{sno, sname}、{sno, ssex}、{sname, ssex}、{sno, sname, ssex}的去重数据量

**生成流程**：

```c
for (k = 2; k <= numattrs; k++) {
    generator = generator_init(numattrs, k);
    while ((combination = generator_next(generator))) {
        item->ndistinct = ndistinct_for_combination(...);
    }
}
```

**STATS_EXT_DEPENDENCIES类型**：

- 计算列之间的函数依赖度
- 函数依赖度定义：两个属性之间满足函数依赖的值占总体数量的比例

**函数依赖度计算**：

```c
for (i = 1; i <= numrows; i++) {
    if (当前值和上一个值不同) {
        if (n_violations == 0) {
            n_supporting_rows += group_size;
        }
        n_violations = 0;
        group_size = 1;
    } else {
        if (第k个值相同) {
            group_size++;
        } else {
            n_violations++;
        }
    }
}
// 函数依赖度 = n_supporting_rows / numrows
```

**数据结构**：

```c
// MVDependencies：记录整个多列统计信息的函数依赖关系
typedef struct MVDependencies {
    uint32 magic;
    uint32 type;
    uint32 ndeps;
    MVDependency *deps[FLEXIBLE_ARRAY_MEMBER];
} MVDependencies;

// MVDependency：记录每个单独的函数依赖关系
typedef struct MVDependency {
    double degree;           // 函数依赖度
    AttrNumber nattributes;  // 涉及的属性数量
    AttrNumber attributes[FLEXIBLE_ARRAY_MEMBER];
} MVDependency;
```

### 5.2 选择率

#### 5.2.1 使用函数依赖计算选择率

**使用条件**：

- 约束条件中只涉及一个表
- 该表上有基于函数依赖的统计信息
- 约束条件形式：Var = Const或RelabelType = Const

**统计信息选择原则**：

1. 约束条件的属性和函数依赖关系统计信息中的键值取交集
2. 交集中的键值多的最适用
3. 如果交集中键值数相同，则键值数少的获胜

**示例**：

```sql
-- 约束条件：A = 1 AND B = 1
-- 统计信息键值：{A, B}, {A, B, C}, {B, C, D}
-- 最适合使用：{A, B}（交集=2，键值数=2）
```

**选择率计算公式**：

```
P(A = Const, B = Const) = P(A = Const) * (d + (1-d) * P(B = Const))
其中d是{A -> B}的函数依赖度
```

**递归计算**：

```c
while (true) {
    dependency = find_strongest_dependency(stat, dependencies, clauses_attnums);
    s2 = clause_selectivity(root, clause, ...);
    s1 *= (dependency->degree + (1 - dependency->degree) * s2);
}
```

#### 5.2.2 子约束条件的选择率

**默认选择率**：

- clause_selectivity函数默认选择率：0.5
- 缓存机制：RestrictInfo->norm_selec和outer_selec

**选择率计算函数表**：

| 子约束条件类型 | 计算方法                                                   |
| -------------- | ---------------------------------------------------------- |
| Var            | 使用boolvarsel函数，或默认0.5                              |
| Const          | NULL或0→选择率0，其他→选择率1                              |
| Param          | 先常量替换，否则默认0.5                                    |
| NOT            | P(B) = 1 – P(A)                                            |
| AND            | P(AB) = P(A) × P(B)                                        |
| OR             | P(A+B) = P(A) + P(B) – P(AB)                               |
| NullTest       | 有统计信息：IS NULL=stanullfrac，IS NOT NULL=1-stanullfrac |
| BooleanTest    | 根据统计信息中的高频值数组计算                             |
| CurrentOfExpr  | 选择率 = 1/表的元组数量                                    |
| RelabelType    | 递归调用clause_selectivity                                 |
| CoerceToDomain | 递归调用clause_selectivity                                 |

#### 5.2.3 基于范围的约束条件的选择率修正

**问题**：约束条件`A > 5 AND A > 7 AND A < 9 AND A < 8`中的子条件不独立

**解决方法**：使用RangeQueryClause链表

**数据结构**：

```c
typedef struct RangeQueryClause {
    struct RangeQueryClause *next;
    Node *var;              // 属性（列）
    bool have_lobound;      // 是否有下界
    bool have_hibound;      // 是否有上界
    Selectivity lobound;    // 下界选择率
    Selectivity hibound;    // 上界选择率
} RangeQueryClause;
```

**修正逻辑**：

```c
if (上界和下界同时存在) {
    s2 = hibound + lobound - 1.0;  // P(A > x) + P(A < y) = 1 - P(A > x AND A < y)
    s2 += nulltestsel(IS_NULL);     // 考虑NULL值
    s1 *= s2;
} else {
    if (只有下界) s1 *= lobound;
    if (只有上界) s1 *= hibound;
}
```

### 5.3 OpExpr的选择率

**连接条件vs过滤条件**：

- varRelid有值：过滤条件
- sjinfo == NULL：过滤条件
- 其他：根据引用表数量判断

**操作符选择率函数对照表**：

| 操作符 | oprrest函数 | oprjoin函数     |
| ------ | ----------- | --------------- |
| =      | eqsel       | eqjoinsel       |
| <>     | neqsel      | neqjoinsel      |
| <=, <  | scalarltsel | Scalarltjoinsel |
| >=, >  | scalargtsel | scalargtjoinsel |
| ~      | regexeqsel  | regexeqjoinsel  |
| ~~     | likesel     | likejoinsel     |

#### 5.3.1 eqsel函数

**处理流程**：

```
eqsel/neqsel
    ↓
eqsel_internal(negate)
    ↓
const? → var_eq_const
nonconst? → var_eq_non_const
```

**var_eq_const函数逻辑**：

1. 如果列有唯一性：选择率 = 1/元组数量
2. 如果常量在高频值数组中：直接使用高频值频率
3. 否则：计算剩余值的平均选择率

**选择率计算**：

```c
// 从高频值数组获取
for (i = 0; i < sslot.nvalues; i++) {
    if (匹配成功) {
        selec = sslot.numbers[i];
        break;
    }
}

// 未匹配时
sumcommon = sum(所有高频值比例);
selec = 1.0 - sumcommon - nullfrac;  // 去除NULL和高频值
otherdistinct = get_variable_numdistinct() - sslot.nnumbers;
selec /= otherdistinct;  // 平均分配

// 限制：不能比高频值中最低的比例高
if (selec > sslot.numbers[sslot.nnumbers - 1])
    selec = sslot.numbers[sslot.nnumbers - 1];
```

#### 5.3.2 scalargtsel函数

**处理流程**：

1. 保证Var在操作符左侧
2. 调用scalarineqsel函数
3. 结合MCV和直方图计算

**直方图计算**：

```c
// 二分法找Const所在的桶
while (lobound < hibound) {
    int probe = (lobound + hibound) / 2;
    // get_actual_variable_range：修正边界值
}

// 计算桶内比例
if (桶边界相等) binfrac = 0.5;
else if (val <= low) binfrac = 0.0;
else if (val >= high) binfrac = 1.0;
else binfrac = (val - low) / (high - low);

// 总选择率
histfrac = (满足条件的桶数 + binfrac) / 总桶数;
selec = (1.0 - NULL比例 - 高频值比例) × histfrac + mcv_selec;
```

#### 5.3.3 eqjoinsel函数

**连接选择率计算的4个部分**：

1. **NULL值率部分**：一个表的某列NULL值比例
2. **高频值数组中互相匹配部分**：两个表的高频值数组取交集
3. **高频值数组中未匹配部分**：去掉互相匹配的值
4. **其他值部分**：去掉NULL值和高频值后的值

**选择率公式**：

```
selec = S(x2)×S(y2) + S(x3)×S(y4) + S(x4)×S(y3) + S(x4)×S(y4)
       ─────────────────────────────────────────────────────────
                          N × N
```

其中：

- S(x2), S(y2)：高频值数组中互相匹配的部分
- S(x3), S(y3)：高频值数组中未匹配的部分
- S(x4), S(y4)：其他值的部分

**匹配算法**：

```c
for (i = 0; i < sslot1.nvalues; i++) {
    for (j = 0; j < sslot2.nvalues; j++) {
        if (值相等) {
            matchprodfreq += sslot1.numbers[i] * sslot2.numbers[j];
            hasmatch1[i] = hasmatch2[j] = true;
            nmatches++;
            break;
        }
    }
}
```

### 5.4 小结

**核心要点**：

1. 统计信息是选择率计算的基础
2. PostgreSQL 10.0之前只有单列统计信息，10.0增加了多列统计信息
3. 选择率估算依赖统计信息的准确性
4. 假设数据分布平坦可能导致估算偏差
5. 无法使用统计信息时采用默认值

**局限性**：

- 统计信息基于样本，样本显著性影响结果
- 数据更新后不会立即更新统计信息
- 基于概率的方法不适用于所有情况
- 默认值可能导致代价估算偏差

---

## 关键概念速查

| 概念     | 定义                                   | 用途                   |
| -------- | -------------------------------------- | ---------------------- |
| 选择率   | 约束条件过滤后的元组数占总元组数的比例 | 查询代价估算           |
| MCV      | 高频值数组，记录最频繁出现的值及其频率 | 等值条件选择率计算     |
| 直方图   | 等频直方图，描述数据分布               | 范围条件选择率计算     |
| 相关系数 | 未排序和排序后数据分布的相关性         | 索引扫描代价估计       |
| 函数依赖 | 列之间的依赖关系                       | 多列约束条件选择率计算 |
| EMA      | 指数平均数指标                         | 元组密度估算           |
| S算法    | 页面采样算法                           | 统计信息生成第一阶段   |
| Z算法    | 蓄水池采样算法                         | 统计信息生成第二阶段   |

---

## 常用SQL查询

```sql
-- 查看表的统计信息
SELECT * FROM pg_statistic WHERE starelid = '表名'::regclass;

-- 查看表的基本信息
SELECT relname, relpages, reltuples FROM pg_class WHERE relname = '表名';

-- 创建多列统计信息
CREATE STATISTICS stats_name ON col1, col2, col3 FROM table_name;

-- 更新统计信息
ANALYZE table_name;

-- 查看多列统计信息
SELECT * FROM pg_statistic_ext WHERE stxname = 'stats_name';

-- 查看查询计划
EXPLAIN ANALYZE SELECT * FROM table_name WHERE 条件;
```