---
title: 第六章-扫描路径
published: 2026-06-16 05:00:00
tags: ["数据库","PostgreSQL","数据库原理"]
category: PostgreSQL
image: "./img.png"
---
## 一句话概括

本章详细介绍了PostgreSQL查询优化器中扫描路径的生成机制，包括代价模型、路径结构、以及顺序扫描/索引扫描/位图扫描三种核心扫描方式的实现原理。

---

## 一、代价（Cost）体系

### 1.1 代价基准单位

PostgreSQL采用**相对代价**模型，不追求"绝对真实"的代价，只要路径间可比较即可。

| 基准类型     | 默认值   | GUC参数                | 说明                          |
| ------------ | -------- | ---------------------- | ----------------------------- |
| 顺序IO       | 1.0      | `seq_page_cost`        | 顺序读写一个页面              |
| 随机IO       | 4.0      | `random_page_cost`     | 随机读写（机械硬盘寻道+预读） |
| CPU元组      | 0.01     | `cpu_tuple_cost`       | 处理一条元组                  |
| CPU索引元组  | 0.005    | `cpu_index_tuple_cost` | 处理一条索引元组              |
| CPU表达式    | 0.0025   | `cpu_operator_cost`    | 表达式求值                    |
| 并行元组传输 | 0.1      | `parallel_tuple_cost`  | Worker→Gather传递元组         |
| 并行初始化   | 1000.0   | `parallel_setup_cost`  | IPC初始化代价                 |
| 缓存大小     | 524288页 | `effective_cache_size` | 估计缓存中的页面数            |

**关键点**：

- 顺序IO vs 随机IO 差距4倍，源于机械硬盘寻道时间和磁盘预读机制
- 支持**表空间级别**自定义代价：`CREATE TABLESPACE ... WITH (SEQ_PAGE_COST=2, RANDOM_PAGE_COST=3)`
- SSD场景下4倍差距可能不再适用，需根据硬件调整

### 1.2 启动代价 vs 整体代价

```
Total Cost = Startup Cost + Run Cost
```

- **Startup Cost**：从执行到返回第一条元组的代价
- **Total Cost**：语句执行的全部代价

**为什么区分？** → LIMIT子句场景

- 无LIMIT：SeqScan+Sort总代价869 < IndexScan总代价7373 → 选SeqScan
- 有LIMIT 1：IndexScan启动代价0.29远低于SeqScan+Sort的230 → 选IndexScan
- 带LIMIT时查询优化器采用**TOP-K堆排序**降低排序启动代价

### 1.3 表达式代价计算

通过 `cost_qual_eval` / `cost_qual_eval_node` 递归计算，核心结构：

```c
typedef struct QualCost {
    Cost startup;    // 启动代价部分
    Cost per_tuple;  // 应用到元组上的代价
} QualCost;
```

| 表达式类型                              | 计算方式                                                     |
| --------------------------------------- | ------------------------------------------------------------ |
| FuncExpr/OpExpr/DistinctExpr/NullIfExpr | per_tuple = procost × cpu_operator_cost                      |
| ScalarArrayOpExpr                       | per_tuple = procost × cpu_operator_cost × sizeof(ARRAY) × 0.5 |
| RowCompareExpr                          | 按元素个数累加                                               |
| SubPlan                                 | startup += subplan->startup_cost; per_tuple += subplan->per_call_cost |
| CurrentOfExpr                           | startup += disable_cost（强制选择TID扫描）                   |

---

## 二、路径（Path）结构

### 2.1 Path结构体（基类）

```c
typedef struct Path {
    NodeTag type;
    NodeTag pathtype;        // 路径类型 T_IndexPath / T_NestPath 等
    RelOptInfo *parent;      // 服务于哪个RelOptInfo
    PathTarget *pathtarget;  // 投影信息
    ParamPathInfo *param_info; // 参数化信息
    bool parallel_aware;     // 是否真正应用了并行
    bool parallel_safe;      // 路径是否并行安全
    int parallel_workers;    // 并行度
    double rows;             // 中间结果估计行数
    Cost startup_cost;       // 启动代价
    Cost total_cost;         // 整体代价
    List *pathkeys;          // 排序键（NULL表示无序）
} Path;
```

### 2.2 并行参数

- **parallel_workers**：实际并行度，通过 `compute_parallel_worker` 计算
- 并行度参考值 = min(heap_pages, index_pages) 的 log₃
- 受 `max_parallel_workers_per_gather` 限制

| 参数                              | 说明                         |
| --------------------------------- | ---------------------------- |
| `min_parallel_table_scan_size`    | 启用并行表扫描的最小页面数   |
| `min_parallel_index_scan_size`    | 启用并行索引扫描的最小页面数 |
| `max_parallel_workers_per_gather` | 每个Gather下的最大并行度     |

**parallel_aware vs parallel_safe**：

- `parallel_safe`：路径能否并行执行（受子路径和RelOptInfo影响）
- `parallel_aware`：路径是否真正被分配了并行Worker

### 2.3 参数化路径

**问题**：`A JOIN B ON A.a = B.b` 中，NestLoop时A表取出一条元组后，需要扫描整个B表。

**解决方案**：将 `A.a = B.b` 转换为 `B.b = {Param of A.a}`，利用B.b上的索引。

```
外表获取一条元组 → 提取参数 → 用参数扫描内表索引
```

**关键结构**：

```c
typedef struct NestLoopParam {
    NodeTag type;
    int paramno;      // PARAM_EXEC参数编号
    Var *paramval;    // 外表的Var
} NestLoopParam;
```

**限制**：

- 只有Nestloop Join支持参数传递（外表"驱动"内表）
- HashJoin/MergeJoin不具备驱动关系，不支持

**示例**：

```sql
SELECT * FROM TEST_A, TEST_B 
WHERE TEST_A.a = TEST_B.a AND TEST_A.a > 9000;
-- 执行计划：NestLoop + SeqScan(A) + IndexScan(B, a = test_a.a)
```

### 2.4 PathKey（排序键）

**作用**：避免重复排序，支持排序复用。

- `PlannerInfo->query_pathkeys`：Non-SPJ→SPJ的"期望顺序"（自上而下）
- `Path->pathkeys`：当前路径的输出顺序（自下而上）

```c
typedef struct PathKey {
    NodeTag type;
    EquivalenceClass *pk_eclass;  // 等价类
    Oid pk_opfamily;              // 操作符族
    int pk_strategy;              // 升序/降序
    bool pk_nulls_first;          // NULL值优先级
} PathKey;
```

**关键优化**：

- B树索引天然有序 → IndexScan可省ORDER BY
- MergeJoin需要两端排序 → 利用已有PathKey可省排序
- 等价类中成员等价：`ORDER BY A.a` 等价于 `ORDER BY B.a`（当 `A.a = B.a`）

---

## 三、make_one_rel函数

### 3.1 路径生成两阶段

```
make_one_rel
├── set_base_rel_pathlists()   -- 生成扫描路径（本章重点）
└── make_rel_from_joinlist()   -- 生成连接路径
```

### 3.2 普通表扫描路径入口

```c
set_plain_rel_pathlist(rel)
├── create_seqscan_path()          -- 顺序扫描
├── create_plain_partial_paths()   -- 并行顺序扫描
├── create_index_paths()           -- 索引扫描
└── create_tidscan_paths()         -- TID扫描
```

---

## 四、顺序扫描（SeqScan）

### 4.1 原理

- 全表扫描，算法复杂度 O(N)
- 直接使用Path结构体（无需单独结构体）
- 适用场景：选择率高、表较小、无合适索引

### 4.2 代价计算

```
disk_run_cost = seq_page_cost × 页面数
cpu_run_cost = (cpu_tuple_cost + 表达式代价) × 元组数
total_cost = startup_cost + cpu_run_cost + disk_run_cost
```

**并行优化**：

- CPU代价分摊：`cpu_run_cost /= parallel_divisor`
- `parallel_divisor = 1.0 - (0.3 × parallel_workers)`（Worker越多促进越小）

### 4.3 参数化顺序扫描

仅当Lateral变量出现在投影中时才生成，如：

```sql
SELECT * FROM TEST_A LEFT JOIN LATERAL (
    SELECT a, TEST_A.a FROM TEST_B
) bb ON TRUE;
```

---

## 五、索引扫描（IndexScan）

### 5.1 索引类型

| 索引类型 | 说明                     |
| -------- | ------------------------ |
| B-tree   | 默认，支持等值/范围/排序 |
| Hash     | 仅等值，O(1)复杂度       |
| GiST     | 几何/全文搜索            |
| GIN      | 倒排索引                 |
| SP-GiST  | 空间分区                 |
| BRIN     | 块范围索引               |

### 5.2 索引扫描路径类型

#### (1) 普通索引扫描（Index Scan）

```sql
SELECT * FROM TEST_A WHERE a = 9000;
-- Index Scan using test_a_abc_idx on test_a
```

- 两步：B树查找索引项 → 随机读堆表获取元组
- 随机IO代价高（random_page_cost=4）

#### (2) 快速索引扫描（Index Only Scan）

```sql
SELECT a,b,c FROM TEST_A WHERE a = 9000;
-- Index Only Scan using test_a_abc_idx on test_a
```

- 投影列是索引列子集时可用
- 省去堆表访问，直接从索引取值

#### (3) 位图索引扫描（Bitmap Index Scan）

```sql
SELECT a,b,c FROM TEST_A WHERE a > 9000;
-- Bitmap Heap Scan → Bitmap Index Scan
```

- 先扫描索引生成位图（记录堆表元组地址）
- 位图排序后顺序读堆表 → 随机读转顺序读
- 适用场景：选择率中等

### 5.3 索引键匹配约束条件

**三类匹配**：

1. `baserestrictinfo` → 普通索引扫描
2. `joininfo` → 参数化路径
3. 等价类隐含约束 → 参数化路径

**支持的表达式**：

| 表达式            | 匹配方式                              |
| ----------------- | ------------------------------------- |
| OpExpr            | indexkey op const / const op indexkey |
| BooleanTest       | 直接匹配/NOT递归/BooleanTest参数      |
| RowCompareExpr    | 仅B树，只匹配第一个操作数             |
| ScalarArrayOpExpr | 仅indexkey op const，仅ANY            |
| NullTest          | 需索引支持amsearchnulls               |

**特殊处理**：

- `LIKE 'abc'` → 提取 `a = 'abc'` 约束
- `a > ANY(ARRAY[1,2,3])` → 支持
- `a > ALL(ARRAY[1,2,3])` → 不支持

### 5.4 索引代价计算

#### 索引自身代价（btcostestimate）

```
indexBoundQuals = 第一个非等值约束之前的所有约束
numIndexTuples = 基于indexBoundQuals选择率估算
indexTotalCost = IO代价 + CPU代价
```

**IO代价模型**（基于Mackert-Lohman模型）：

```
有效缓存页面数 b = (effective_cache_size / total_pages) × T
```

**索引相关系数**：

- 单键索引：统计信息中的相关系数 ρ
- 多键索引：第一键相关系数 × 0.75

#### 堆表扫描代价（cost_index）

```
io_cost = max_IO_cost + ρ² × (min_IO_cost - max_IO_cost)
```

- `max_IO_cost`：完全不相关（ρ=0），全随机读
- `min_IO_cost`：完全正相关（ρ=1），顺序读

### 5.5 索引PathKey记录条件

需同时满足：

1. 非位图扫描路径
2. 无ScalarArrayOpExpr或在第一键
3. 索引操作符族保证有序（sortopfamily != NULL）
4. 有序性质"有用"（有MergeJoin或ORDER BY期望）

---

## 六、位图扫描（Bitmap Scan）

### 6.1 核心思想

将索引扫描的**随机堆表读取**转换为**顺序堆表读取**。

### 6.2 路径结构

```
BitmapHeapPath（顶层）
├── BitmapAndPath（AND操作）
│   ├── IndexPath（叶子）
│   └── IndexPath（叶子）
└── BitmapOrPath（OR操作）
    ├── IndexPath（叶子）
    └── BitmapOrPath（嵌套）
```

**结构体**：

```c
typedef struct BitmapHeapPath {
    Path path;
    Path *bitmapqual;  // IndexPath / BitmapAndPath / BitmapOrPath
} BitmapHeapPath;

typedef struct BitmapAndPath {
    Path path;
    List *bitmapquals;        // IndexPaths和BitmapOrPaths
    Selectivity bitmapselectivity;
} BitmapAndPath;

typedef struct BitmapOrPath {
    Path path;
    List *bitmapquals;        // IndexPaths和BitmapAndPaths
    Selectivity bitmapselectivity;
} BitmapOrPath;
```

### 6.3 BitmapOrPath生成

支持OR类型约束条件：

```sql
SELECT * FROM TEST_A WHERE A > 1 OR A > 2;
-- BitmapOr → Bitmap Index Scan(A>1) + Bitmap Index Scan(A>2)
```

### 6.4 BitmapAndPath生成

三阶段组合算法：

1. **筛选**：相同约束条件集合的路径，保留代价低的
2. **排序**：按代价升序，代价相同则选择率低优先
3. **组合**：O(N²)组合，组合后代价升高则淘汰

### 6.5 位图扫描代价

| 路径类型          | 代价计算                                         |
| ----------------- | ------------------------------------------------ |
| IndexPath（叶子） | 仅索引自身代价（indextotalcost）                 |
| BitmapOrPath      | 子节点代价和 + 100×cpu_operator_cost（并集操作） |
| BitmapAndPath     | 子节点代价和                                     |
| BitmapHeapPath    | 启动代价=子节点总代价；IO代价使用插值公式        |

**BitmapHeap IO代价公式**：

```
io_cost = random_page_cost - (random_page_cost - seq_page_cost) × √(pages_fetched / T)
```

- 有序但非连续 → 不完全等于seq_page_cost
- 选择率低时跳过预读范围 → 接近random_page_cost

---

## 七、核心要点总结

### 7.1 扫描路径选择决策树

```
选择率高 → SeqScan
选择率低 → IndexScan（B树等值/范围）
选择率中等 → BitmapScan
有索引且投影列在索引中 → IndexOnlyScan
有多个索引条件组合 → BitmapAnd/BitmapOr
```

### 7.2 关键优化手段

1. **参数化路径**：NestLoop中利用外表值驱动内表索引
2. **PathKey复用**：避免重复排序，支持MergeJoin优化
3. **位图扫描**：随机读转顺序读，支持多索引组合
4. **并行扫描**：分摊CPU代价，受限于GUC参数
5. **启动代价区分**：支持LIMIT场景的路径选择

### 7.3 常见GUC参数速查

```sql
-- 代价调整
SET seq_page_cost = 1.0;
SET random_page_cost = 4.0;
SET cpu_tuple_cost = 0.01;
SET cpu_operator_cost = 0.0025;

-- 并行控制
SET min_parallel_table_scan_size = 8MB;
SET min_parallel_index_scan_size = 512kB;
SET max_parallel_workers_per_gather = 2;

-- 缓存估计
SET effective_cache_size = 4GB;
```

---