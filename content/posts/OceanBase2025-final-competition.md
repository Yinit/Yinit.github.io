---
title: "OceanBase2025 决赛回顾"
date: 2026-04-05T06:58:02Z
lastMod: 2026-04-05T06:58:02Z
draft: false # 是否为草稿
author: ["tkk"]

categories: ["数据库"]

tags: ["OceanBase", "向量检索", "全文索引"]

keywords: ["OceanBase", "VectorDB", "全文索引", "数据库竞赛"]

description: "OceanBase2025 大赛 内核赛道部分回顾" # 文章描述，与搜索优化相关
summary: "OceanBase2025 大赛 内核赛道部分回顾" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: true
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

## 1. 项目背景与比赛概述

### 1.1 比赛任务描述

**赛题**：在 OceanBase-seekdb（基于 `2025-final-competition` 分支）上，优化带标量过滤条件的全文索引检索性能。

核心查询SQL：

```sql
SELECT `docid_col`, MATCH(`fulltext_col`) AGAINST(`text`) as _score
FROM `items1`
WHERE MATCH(fulltext_col) AGAINST(`text`)
  AND base_id IN ('base_id_1', 'base_id_2', 'base_id_3')
  AND id < 1000
ORDER BY _score DESC
LIMIT 10
```

**表结构**：

```sql
CREATE TABLE `items1` (
  `id` BIGINT AUTO_INCREMENT,
  `base_id` VARCHAR(255) NOT NULL,
  `docid_col` VARCHAR(255) NOT NULL,
  `fulltext_col` LONGTEXT
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COLLATE=utf8mb4_unicode_ci;

-- 普通索引
CREATE INDEX idx_base_id ON `items1`(`base_id`);
CREATE INDEX idx_docid ON `items1`(`docid_col`);

-- 全文索引
CREATE /*+ parallel(90) */ FULLTEXT INDEX ft_fulltext ON `items1`(`fulltext_col`);
```

**数据集**：MLDR英文数据集（约20万条英文长文档）

**评分标准**：

- 三轮查询的平均QPS（召回率需>0.95）
- 测试流程：1次预热 + 3次正式测试
- 时间限制：整个测试过程30分钟内完成

### 1.2 测试环境

- 单机模式，8C 16G
- 测试脚本与seekdb在同一台机器
- 所有查询的标量条件固定：`id < 1000` 且 `base_id IN (固定集合)`

### 1.3 关键约束

- 不能修改编译选项或构建脚本
- 不能绕过SQL执行引擎直接返回数据
- 只允许使用现有缓存机制（如kv cache等）
- 不能使用额外的缓存机制

### 1.4 团队分工

| 成员     | 主要工作                                                     |
| -------- | ------------------------------------------------------------ |
| 我       | IDF缓存、倒排链结果缓存、BMW算子优化、Bitmap过滤方案设计     |
| 同学A    | TopN下推、代价模型修改、BM25向量化、自增列自动建索引、IndexMerge尝试 |
| 同学B    | RAG部分（AI应用赛题）                                        |

### 1.5 优化成效时间线

```
初始状态:  ~5分   (无任何优化，基本等同于全表扫描)
TopN下推:  ~100分  (强制走全文本索引算子 + 开启TopN谓词下推)
IDF缓存:   ~500分  (初版全局hashmap缓存，后迁移到ObKVCache)
结果缓存:  ~1300分 (缓存倒排链扫描结果，避免重复IO)
最终状态:  未能提交有效成绩（Bitmap方案未完成，导致违规内容被取消）
```

---

## 2. seekdb架构深度解析

### 2.1 全文检索整体架构

seekdb的全文检索是OceanBase全文索引能力的轻量化版本，核心思路是：

```
用户SQL查询
    │
    ▼
[SQL层] 查询解析 → 逻辑计划 → 优化器 → 物理计划
    │
    ▼
[DAS层] 分布式访问服务 (Data Access Service)
    │  - 管理倒排索引扫描
    │  - 管理正排索引聚合
    │  - 多token合并
    ▼
[存储层] Token迭代器 → 合并算法(TAAT/DaaT/BMW) → TopK结果
    │
    ▼
[索引层] SSTable倒排索引 + 正排索引 (LSM-Tree存储)
```

### 2.2 全文索引的存储结构

seekdb全文索引包含以下辅助表（以主表`items1`为例）：

**倒排索引表** (`__idx_xxx_inv_idx`)：

- 记录每个token在哪些文档中出现，以及出现的频率
- 键：`(token_text, doc_id)`
- 值：`token_count`（token在文档中出现的次数）

**正排索引表** (`__idx_xxx_fwd_idx`)：

- 记录每个文档的长度信息（用于BM25计算）
- 键：`doc_id`
- 值：`doc_length`（文档中的token总数）

**文档ID映射表** (`__idx_xxx_doc_id_rowkey`)：

- 映射内部doc_id到主表rowkey
- 用于通过全文检索结果回查主表数据

### 2.3 查询执行路径详解

#### 2.3.1 SQL优化器层

关键文件：

- src/sql/optimizer/ob_log_plan.cpp — TopN下推逻辑
- src/sql/optimizer/ob_opt_est_cost_model.cpp — 代价估计

优化器在看到 `MATCH...AGAINST` + `ORDER BY _score DESC LIMIT N` 时，会考虑：

1. **走普通表扫描路径**：忽略全文索引，直接扫主表
2. **走全文索引路径**：使用全文索引获取Top-K候选集，再回表

关键代码路径（`try_push_topn_into_text_retrieval_scan`）：

```cpp
// ob_log_plan.cpp（简化）
// 判断是否可以将TopN下推到全文检索扫描
int ObLogPlan::try_push_topn_into_text_retrieval_scan(ObLogTopk *topk_op, ...) {
  // 检查是否有标量过滤条件（原始代码在有filter时会放弃下推）
  // [优化]：移除这个检查，允许带filter的TopN下推
  // 这样TEXT_RETRIEVAL_SCAN可以知道只需要返回Top-N个结果
}
```

#### 2.3.2 DAS层（分布式访问服务层）

关键文件：

- src/sql/das/iter/ob_das_text_retrieval_iter.h
- src/sql/das/iter/sparse_retrieval/ob_das_tr_merge_iter.cpp

DAS层的职责：

1. 接收SQL层传来的查询token列表
2. 为每个token创建 `ObTextRetrievalTokenIter`
3. 选择合并策略（TAAT/DaaT/BMW）
4. 协调倒排索引扫描和正排索引聚合

关键类：

```
ObDASTextRetrievalIter          — 单token检索（DAS入口）
ObDASTRCacheIter                — 带结果缓存的单token检索
ObDASTextRetrievalMergeIter     — 多token合并检索（DAS层）
ObDAsTrMergeIter                — 稀疏检索合并迭代器
```

#### 2.3.3 存储层迭代器

关键文件（src/storage/retrieval/目录）：

```
ObTextRetrievalTokenIter        — 单token的倒排链迭代
ObTextRetrievalDaaTTokenIter    — 包装为DaaT接口的Token迭代器
ObTextRetrievalBlockMaxIter     — 带块最大分数的Token迭代器（供BMW使用）
```

**迭代器接口层次**：

```
ObISparseRetrievalDimIter           (基础接口)
├── get_next_row()
├── get_next_batch()
└── advance_to(id_datum)
    │
    └── ObISRDaaTDimIter            (DaaT接口)
        ├── get_curr_score()
        ├── get_curr_id()
        ├── get_dim_max_score()
        │
        └── ObISRDimBlockMaxIter    (Block-Max接口)
            ├── advance_shallow()   — 浅推进（不移动游标，只更新块信息）
            ├── get_curr_block_max_info()
            └── in_shallow_status()
```

### 2.4 BM25评分算法

BM25（Best Match 25）是现代全文检索的标准相关性评分算法，相比简单的TF-IDF有更好的词频饱和处理。

**BM25公式**：

```
BM25(D, Q) = Σ_{t∈Q} IDF(t) × (tf(t,D) × (k1+1)) / (tf(t,D) + k1×(1 - b + b×|D|/avgdl))
```

**参数说明**：

- `t`：查询中的每个token
- `D`：文档
- `Q`：查询
- `tf(t,D)`：token t 在文档 D 中的词频（出现次数）
- `|D|`：文档长度（token总数）
- `avgdl`：所有文档的平均长度
- `k1`：词频饱和参数（通常=1.2-2.0，控制词频对分数的影响上限）
- `b`：文档长度归一化参数（通常=0.75，b=0则不考虑文档长度）

**IDF公式**：

```
IDF(t) = ln((N - df(t) + 0.5) / (df(t) + 0.5) + 1)
```

其中：

- `N`：文档总数
- `df(t)`：包含token t 的文档数（Document Frequency）

**关键代码位置**：src/sql/engine/expr/ob_expr_bm25.cpp

```cpp
// BM25评分计算（简化示意）
double calc_bm25_score(
    int64_t token_count,     // tf(t,D)：token在文档中的出现次数
    int64_t doc_length,      // |D|：文档长度
    int64_t token_doc_cnt,   // df(t)：token的文档频率
    int64_t total_doc_cnt,   // N：总文档数
    double avg_doc_length)   // avgdl：平均文档长度
{
    const double k1 = 1.2;
    const double b = 0.75;

    // IDF计算
    double idf = std::log((total_doc_cnt - token_doc_cnt + 0.5) /
                          (token_doc_cnt + 0.5) + 1.0);

    // TF归一化
    double tf_norm = (token_count * (k1 + 1)) /
                     (token_count + k1 * (1 - b + b * doc_length / avg_doc_length));

    return idf * tf_norm;
}
```

**BM25的关键性质**：

1. **词频饱和**：`tf`不会无限增长，达到上限后趋于稳定（避免垃圾文档通过堆砌关键词获高分）
2. **文档长度归一化**：长文档中的词频会被适当降权（同等出现次数，短文档更相关）
3. **IDF惩罚常见词**：出现在很多文档中的词（如"the"）得分权重低

### 2.5 三种查询算法对比

#### TAAT（Term-At-A-Time）

**思路**：逐词处理，对每个token，扫描其完整倒排链，累加所有文档的分数。

```
Token1的倒排链: doc1→score1.1, doc2→score1.2, doc5→score1.5
Token2的倒排链: doc1→score2.1, doc3→score2.3, doc5→score2.5

处理Token1: acc[doc1]+=score1.1, acc[doc2]+=score1.2, acc[doc5]+=score1.5
处理Token2: acc[doc1]+=score2.1, acc[doc3]+=score2.3, acc[doc5]+=score2.5
最终: 对acc数组取Top-K
```

**优点**：

- 对小数据集极其友好（顺序访问内存）
- 实现简单，缓存友好
- 当文档ID范围已知且较小时（如id<1000），密集数组效率极高

**缺点**：

- 即使最终只需要Top-K，也要处理所有文档
- 内存占用与文档总数成正比

**适用场景**：数据量小，或标量过滤后候选集小的情况

**实现文件**：src/storage/retrieval/ob_sparse_taat_iter.cpp

#### DaaT（Document-At-A-Time）

**思路**：所有token的倒排链同时推进（使用败者树/最小堆维护最小doc_id），每次取所有链中最小doc_id的文档，合并来自不同token的分数。

```
Token1: [doc1, doc2, doc5, ...]
Token2: [doc1, doc3, doc5, ...]
Token3: [doc2, doc5, doc7, ...]

合并堆: 按doc_id排序
→ 取doc1: 合并Token1和Token2的分数
→ 取doc2: 合并Token1和Token3的分数
→ 取doc3: 只有Token2的分数
→ 取doc5: 合并Token1、Token2、Token3的分数
...
```

**优点**：

- 天然支持TopK剪枝（可以提前终止）
- 内存使用与token数（维度数）相关，不与文档总数相关

**缺点**：

- 指针跳跃，缓存不友好
- 败者树/堆操作有开销

**关键数据结构**：

```cpp
// 败者树（Loser Tree）用于多路归并
ObSRMergeLoserTree merge_heap_;

// 合并项
struct ObSRMergeItem {
    int64_t iter_idx_;    // 对应哪个token的迭代器
    double relevance_;     // 当前doc_id在这个token下的分数
};

// 迭代器域ID数组（每个token当前的doc_id）
ObFixedArray<const ObDatum *, ObIAllocator> iter_domain_ids_;
```

**实现文件**：src/storage/retrieval/ob_sparse_daat_iter.cpp

#### BMW（Block-Max WAND）

**思路**：在DaaT基础上加入块级最大分数剪枝。将倒排链分块存储，记录每个块内的最大可能分数。在合并时，利用上界分数进行pivot-based剪枝，跳过不可能进入Top-K的文档。

**WAND算法核心思想**：

1. 维护Top-K结果堆，记录阈值 θ（当前第K名的分数）
2. 找"pivot"：按doc_id排序所有token的当前指针，找到满足"累计最大分数 > θ"的最小doc_id（即pivot）
3. 对pivot进行精确评估：移动所有包含pivot的token到该位置，计算精确分数
4. 若精确分数 > θ，更新Top-K堆

**Block-Max的增强**：

- 倒排链按块（chunk）存储，每块记录块内最大BM25分数
- 在评估pivot时，先用块级最大分数做范围剪枝（evaluate_pivot_range）
- 若块级上界 ≤ θ，直接跳过整块（避免逐文档计算）

**状态机**（实现在src/storage/retrieval/ob_sparse_bmw_iter.cpp）：

```
FIND_NEXT_PIVOT
    │ 找到pivot
    ▼
EVALUATE_PIVOT_RANGE ←─────┐
    │ 块级上界 ≤ θ           │
    ├──────────────────► FIND_NEXT_PIVOT_RANGE
    │ 块级上界 > θ           │
    ▼                       │
EVALUATE_PIVOT              │ 未找到候选范围
    │ 精确评估，更新Top-K    │
    ▼                       │
FIND_NEXT_PIVOT ────────────┘
```

**优点**：

- 对稀疏查询（关键词很少出现）效率极高
- 块级剪枝大幅减少精确评估次数

**缺点**：

- 实现复杂
- 数据集小、块内记录多时，块级剪枝收益不明显
- 建立初始Top-K堆需要先处理K个文档

**实现文件**：src/storage/retrieval/ob_sparse_bmw_iter.h，src/storage/retrieval/ob_sparse_bmw_iter.cpp

### 2.6 块最大分数迭代器（Block-Max Iterator）

**文件**：src/storage/retrieval/ob_block_max_iter.h

块最大分数记录了倒排链中每个块（256个文档为一块）的统计信息：

```cpp
struct ObMaxScoreTuple {
    const ObDatum *min_domain_id_;   // 块的起始doc_id
    const ObDatum *max_domain_id_;   // 块的结束doc_id
    double max_score_;               // 块内最大可能BM25分数
};
```

这些统计信息是在建索引时由块统计收集器（src/storage/retrieval/ob_block_stat_collector.cpp）计算并存储在SSTable的二级索引中的。

---

## 3. 优化工作详解

#### 问题分析

原始执行计划中，即使查询有 `ORDER BY _score DESC LIMIT 10`，优化器也可能因为存在标量过滤条件（`id < 1000` 和 `base_id IN (...)`）而放弃TopN下推，导致：

- 全文检索算子扫描完整倒排链
- 返回所有匹配文档
- 在上层做排序和limit

这意味着即使最终只需要10条结果，系统可能仍需要处理数万条文档。

#### 解决方案

**修改文件**：src/sql/optimizer/ob_log_plan.cpp

```cpp
// 原始代码：有scalar filter时不做TopN下推
int ObLogPlan::try_push_topn_into_text_retrieval_scan(...) {
    // 检查是否有过滤条件
    if (has_scalar_filter) {
        // 放弃下推
        return OB_SUCCESS;
    }
    // ...做下推
}

// 优化后：移除这个限制，允许带filter的TopN下推
int ObLogPlan::try_push_topn_into_text_retrieval_scan(...) {
    // 直接做下推，不检查scalar filter
    // 全文检索算子会在内部处理filter
}
```

**代价模型修改**（src/sql/optimizer/ob_opt_est_cost_model.cpp）：

```cpp
// 将全文索引的代价估算置为0，强制优化器选择全文索引路径
// 原始代码有复杂的代价计算，导致有时选择不走全文索引
fulltext_scan_cost = 0;  // 强制走全文索引
```

#### 效果

- 从5分提升到100+分
- 全文检索算子只需返回Top-K个结果，大幅减少无用计算
- 这是最基础也是最重要的优化，后续所有优化都建立在这个基础上

#### 设计思考

TopN下推是数据库中的经典优化。其核心思想是：**把上层的约束尽早传递到下层**，让下层算子能够提前终止。对于全文检索来说：

- 若知道只需要Top-10，BMW算法的阈值θ会更快收敛
- 避免处理分数排名11以后的所有文档

#### 问题分析

IDF（Inverse Document Frequency）是BM25的核心参数之一：

```
IDF(t) = ln((N - df(t) + 0.5) / (df(t) + 0.5) + 1)
```

其中`df(t)`（Document Frequency）是包含token t的文档数量。

**计算IDF需要的步骤**（对应`estimate_token_doc_cnt()`函数）：

1. 扫描倒排索引的聚合表（`inv_idx_agg`）
2. 统计包含该token的文档总数
3. 查询正排索引获取总文档数
4. 代入BM25公式计算IDF

对于整个测试过程（1次预热 + 3次正式查询，每次约500个查询），同一个token的IDF计算会被重复进行**数千次**，而数据不会变化，这是明显的重复计算开销。

#### 初版实现：全局HashMap

```cpp
// 第一版（提交fd702ae8）：简单全局hashmap
static std::unordered_map<std::string, int64_t> g_token_df_cache;
static oceanbase::common::SpinRWLock g_df_cache_lock;

// 查询时
int ObTextRetrievalTokenIter::estimate_token_doc_cnt() {
    if (!curr_token_text_.empty()) {
        oceanbase::common::SpinRLockGuard guard(g_df_cache_lock);
        auto it = g_token_df_cache.find(curr_token_text_);
        if (it != g_token_df_cache.end()) {
            token_doc_cnt_ = it->second;
            token_doc_cnt_calculated_ = true;
            return OB_SUCCESS;  // 缓存命中，直接返回
        }
    }
    // 缓存未命中：执行倒排索引扫描
    // ...

    // 计算完成后写入缓存
    if (!curr_token_text_.empty()) {
        oceanbase::common::SpinWLockGuard guard(g_df_cache_lock);
        g_token_df_cache[curr_token_text_] = token_doc_cnt_;
    }
}
```

**问题**：使用了STL的`unordered_map`和`SpinRWLock`，违反了比赛约束（不能使用额外的缓存机制）。

#### 正式实现：OceanBase原生KVCache

**文件结构**：

- src/storage/blocksstable/ob_token_df_cache.h
- src/storage/blocksstable/ob_token_df_cache.cpp

**缓存Key设计**：

```cpp
class ObTokenDFCacheKey : public common::ObIKVCacheKey {
public:
    uint64_t tenant_id_;       // 租户ID（支持多租户）
    ObTabletID tablet_id_;     // 数据库对象标识符
    ObString token_;           // token文本（变长字符串）

    // 哈希计算
    int hash(uint64_t &hash_val) const {
        hash_val = murmurhash(&tenant_id_, sizeof(tenant_id_), hash_val);
        hash_val = murmurhash(&tablet_id_, sizeof(tablet_id_), hash_val);
        hash_val = murmurhash(token_.ptr(), token_.length(), hash_val);
        return OB_SUCCESS;
    }

    // 深拷贝（缓存框架要求）
    int deep_copy(char *buf, const int64_t buf_len, ObIKVCacheKey *&key) const {
        // 将token_内容内联存储在Key对象后面
        ObTokenDFCacheKey *new_key = new (buf) ObTokenDFCacheKey(...);
        char *data_ptr = buf + sizeof(*this);
        MEMCPY(data_ptr, token_.ptr(), token_.length());
        new_key->token_.assign_ptr(data_ptr, token_.length());
        key = new_key;
        return OB_SUCCESS;
    }

    int64_t size() const {
        return sizeof(*this) + token_.length();  // 变长Key
    }
};
```

**缓存Value设计**：

```cpp
class ObTokenDFCacheValue : public common::ObIKVCacheValue {
public:
    int64_t token_doc_cnt_;       // df(t)：含有该token的文档数
    double max_token_relevance_;   // 该token能贡献的最大BM25分数（用于BMW剪枝）

    int64_t size() const { return sizeof(*this); }

    int deep_copy(char *buf, ...) const {
        // 简单的固定大小值，直接拷贝即可
        ObTokenDFCacheValue *new_val = new (buf) ObTokenDFCacheValue();
        new_val->token_doc_cnt_ = token_doc_cnt_;
        new_val->max_token_relevance_ = max_token_relevance_;
        return OB_SUCCESS;
    }
};
```

**缓存注册**（src/storage/blocksstable/ob_storage_cache_suite.h）：

```cpp
class ObStorageCacheSuite {
public:
    ObTokenDFCache token_df_cache_;     // IDF缓存
    ObTokenResultCache token_result_cache_;  // 结果缓存

    ObTokenDFCache &get_token_df_cache() { return token_df_cache_; }
    ObTokenResultCache &get_token_result_cache() { return token_result_cache_; }
};
```

**使用方式**（src/storage/retrieval/ob_text_retrieval_token_iter.cpp）：

```cpp
// 初始化时尝试从缓存读取
int ObTextRetrievalTokenIter::init(const ObTextRetrievalScanIterParam &iter_param) {
    // ...初始化代码...

    if (!curr_token_text_.empty()) {
        const uint64_t tenant_id = MTL_ID();
        blocksstable::ObTokenDFCacheKey key(
            tenant_id,
            inv_idx_agg_param_->tablet_id_,
            ObString(curr_token_text_.length(), curr_token_text_.c_str()));
        blocksstable::ObTokenDFValueHandle handle;
        if (OB_SUCCESS == OB_STORE_CACHE.get_token_df_cache().get_token_df(key, handle)) {
            // 缓存命中：直接使用缓存的IDF值
            token_doc_cnt_ = handle.value_->get_token_doc_cnt();
            max_token_relevance_ = handle.value_->get_max_token_relevance();
            token_doc_cnt_calculated_ = true;  // 标记已计算，跳过倒排索引扫描
        }
    }
}

// 计算完IDF后写入缓存
int ObTextRetrievalTokenIter::update_max_token_relevance(const double max_token_relevance) {
    max_token_relevance_ = max_token_relevance;
    if (!curr_token_text_.empty()) {
        const uint64_t tenant_id = MTL_ID();
        blocksstable::ObTokenDFCacheKey key(...);
        blocksstable::ObTokenDFCacheValue token_value;
        if (OB_SUCC(token_value.init(token_doc_cnt_, max_token_relevance_))) {
            OB_STORE_CACHE.get_token_df_cache().put_token_df(key, token_value);
        }
    }
    return ret;
}
```

#### OceanBase KVCache的优势

OceanBase原生的`ObKVCache`是一个通用的键值缓存框架，具备：

1. **自动LRU淘汰**：内存不足时自动淘汰最久未使用的条目
2. **线程安全**：原生支持多线程并发，无需额外锁
3. **内存配额控制**：通过`mem_limit_pct`参数控制最大内存占用
4. **零拷贝访问**：通过handle机制访问缓存值，避免额外内存分配
5. **多租户支持**：按tenant_id隔离缓存数据

#### 效果分析

- IDF的计算从 **O(磁盘IO)** 变为 **O(1)** 内存访问（缓存命中后）
- 预热阶段（第1次查询）建立缓存，后续3次查询100%命中
- 分数从5分提升到约500分

#### 为什么还要缓存max_token_relevance

`max_token_relevance_`是token能贡献的最大BM25分数，计算需要：

1. 知道IDF（即df(t)）
2. 假设最理想情况（tf趋于饱和，文档长度等于avgdl）

对BMW算法来说，`max_token_relevance_`是**剪枝的关键**：BMW用它来判断"即使这个文档包含了所有查询词的理想情况，分数是否还能超过当前阈值θ"。

同时缓存`max_token_relevance_`避免了每次都重新计算这个上界，加速了BMW的初始化。

#### 问题分析

在TopN下推和IDF缓存之后，还存在一个重要瓶颈：**倒排链扫描本身**。

对于每次查询，即使IDF已知，仍需要：

1. 读取token对应的倒排链（磁盘IO）
2. 逐条计算doc_id的BM25分数
3. 将(doc_id, score)对返回给合并层

而整个测试过程中，查询集固定（同一批500个query重复执行4次），同一个query中的token会被重复扫描多次。

#### 实现设计

**文件**：

- src/storage/blocksstable/ob_token_result_cache.h
- src/storage/blocksstable/ob_token_result_cache.cpp

**缓存Key设计**：

```cpp
class ObTokenResultCacheKey : public common::ObIKVCacheKey {
public:
    uint64_t tenant_id_;
    uint64_t tablet_id_;       // 注意：这里用uint64_t而非ObTabletID
    ObString token_;
    int64_t start_doc_id_;     // 分页起始位置，支持大结果集分页缓存

    int64_t size() const {
        return sizeof(*this) + token_.length();
    }
};
```

**为什么需要start_doc_id**：

倒排链可能很长（一个常见词可能出现在数万文档中）。我们不能一次性缓存所有结果（内存不足），因此按页缓存：

- 每次缓存一批结果（如256个doc_id-score对）
- Key中包含`start_doc_id`作为页码标识

**缓存Value设计**：

```cpp
class ObTokenResultCacheValue : public common::ObIKVCacheValue {
public:
    int64_t count_;            // 本次缓存的结果数
    const ObDatum *doc_ids_;   // doc_id数组（指向内部buffer）
    const double *scores_;     // score数组（指向内部buffer）

    // 计算总大小（包含内联存储的数组数据）
    int64_t size() const {
        int64_t total_size = sizeof(*this);
        total_size += count_ * sizeof(double);    // scores数组
        total_size += count_ * sizeof(ObDatum);   // doc_ids数组
        // doc_ids的payload数据（变长字符串）
        for (int64_t i = 0; i < count_; ++i) {
            if (!doc_ids_[i].is_null() && doc_ids_[i].len_ > 0) {
                total_size = upper_align(total_size, 8);  // 8字节对齐
                total_size += doc_ids_[i].len_;
            }
        }
        return total_size;
    }

    // 深拷贝：将所有数据内联存储在缓存buffer中
    int deep_copy(char *buf, const int64_t buf_len, ObIKVCacheValue *&value) const {
        value = new (buf) ObTokenResultCacheValue();
        auto *result_value = static_cast<ObTokenResultCacheValue *>(value);
        result_value->count_ = count_;

        int64_t offset = sizeof(*this);

        // 1. 拷贝scores数组
        result_value->scores_ = reinterpret_cast<double*>(buf + offset);
        MEMCPY(buf + offset, scores_, count_ * sizeof(double));
        offset += count_ * sizeof(double);

        // 2. 拷贝ObDatum数组
        ObDatum *new_doc_ids = reinterpret_cast<ObDatum*>(buf + offset);
        MEMCPY(new_doc_ids, doc_ids_, count_ * sizeof(ObDatum));
        result_value->doc_ids_ = new_doc_ids;
        offset += count_ * sizeof(ObDatum);

        // 3. 深拷贝每个Datum的payload（变长字符串内容）
        for (int64_t i = 0; i < count_; ++i) {
            if (!new_doc_ids[i].is_null() && new_doc_ids[i].len_ > 0) {
                offset = upper_align(offset, 8);   // 对齐
                char *payload_ptr = buf + offset;
                MEMCPY(payload_ptr, doc_ids_[i].ptr_, doc_ids_[i].len_);
                new_doc_ids[i].ptr_ = payload_ptr;  // 重定向指针
                offset += doc_ids_[i].len_;
            }
        }
        return OB_SUCCESS;
    }
};
```

#### 深拷贝策略的关键设计

OceanBase KVCache要求深拷贝，因为原始数据（倒排链扫描结果）存储在线程私有的arena allocator中，在查询结束后会被释放。

**内存布局**（缓存buffer内的布局）：

```
[ObTokenResultCacheValue对象]
[scores_数组: count_个double]
[doc_ids_数组: count_个ObDatum结构体]
[doc_id[0]的字符串内容]   ← 8字节对齐
[doc_id[1]的字符串内容]   ← 8字节对齐
...
[doc_id[n-1]的字符串内容] ← 8字节对齐
```

所有数据内联在同一个连续buffer中，`ObDatum.ptr_`指向同一buffer内的位置。这样：

1. 缓存框架只需管理一个连续内存块
2. 避免了碎片化
3. 数据局部性好（访问doc_id时，内容紧跟在结构体后面）

#### 效果分析

- 倒排链扫描从 **O(磁盘IO × 文档数)** 变为 **O(1)** 内存访问（缓存命中）
- 第1轮查询：建立缓存（有IO）
- 第2-4轮查询：全部从缓存读取，接近0 IO
- 分数从约500分提升到约1300分

#### 问题1：内存分配开销

BMW算法在处理每个查询时需要频繁分配和释放内存，包括：

- Top-K堆的节点
- 文档ID缓存数组（id_cache_）
- 各种临时数组

**原始问题**：

- 默认allocator页面大小太小，频繁触发malloc
- 每次查询结束后，所有内存被释放，下次查询从零开始分配

**优化1：增大allocator页面大小**

```cpp
// 修改前：默认页面大小（通常4KB或8KB）
ObSRBMWIterImpl() : allocator_("BMWAlloc"), ...

// 修改后：128KB页面，减少malloc次数
ObSRBMWIterImpl() : allocator_("BMWAlloc", 128 * 1024), ...
```

**优化2：内存reuse机制**

```cpp
void ObSRBMWIterImpl::reuse(const bool switch_tablet) {
    while (!top_k_heap_.empty()) {
        top_k_heap_.pop();
    }
    status_ = BMWStatus::MAX_STATUS;
    allocator_.reuse();  // 释放内存给"缓存池"，不实际归还给OS
    ObSRDaaTIterImpl::reuse(switch_tablet);
}
```

`allocator_.reuse()`与`allocator_.reset()`的区别：

- `reset()`：释放所有内存给操作系统
- `reuse()`：保留物理内存映射，只在逻辑上清空，下次分配时直接复用

#### 问题2：BMW在小数据集上不是最优

**关键发现**：对于这个比赛的场景（`id < 1000`，最多1000个有效文档），BMW的块级剪枝收益非常有限：

- 倒排链每块约256个文档
- 有效文档最多1000个，只有约4个块
- 块级跳跃几乎不发生

而BMW比DaaT多了：

- 块级最大分数的计算和维护
- Shallow Advance的状态切换
- 范围评估（evaluate_pivot_range）
- 复杂的状态机

**优化：用 TAAT 密集累加器替代 BMW**

先厘清三者的核心区别：

| 算法 | 迭代维度                          | 数据结构            | Top-K剪枝                   |
| ---- | --------------------------------- | ------------------- | --------------------------- |
| TAAT | 逐 **term**，扫完整倒排链         | 密集累加数组        | 无（处理所有文档后取Top-K） |
| DaaT | 逐 **document**，多路归并同步推进 | 败者树/最小堆       | 可提前终止                  |
| BMW  | 逐 **document**（DaaT的子类）     | 败者树 + 块最大分数 | 块级上界剪枝，跳过整块      |

提交 `700596b8` 的核心改动：在 BMW 类内部，当有效文档范围较小时，直接采用 **TAAT 风格的密集累加器**，完全绕过 BMW/DaaT 的败者树和块级状态机。

```cpp
// 在ObSRBMWIterImpl::top_k_search()中的改动思路（伪代码）：
int top_k_search() {
    // 对于 id < 1000 的场景，预分配密集得分数组（约8KB，完全在L1缓存中）
    double scores[1000] = {0.0};

    // TAAT模式：外层循环按 term（dim_iters_ 是各 token 的迭代器）
    for (auto iter : dim_iters_) {
        // 对每个 term，扫描其完整倒排链
        while (iter->has_next()) {
            int64_t doc_id = iter->curr_id();
            if (doc_id < 1000) {
                scores[doc_id] += iter->curr_score();  // 累加到密集数组
            }
            iter->next();
        }
    }

    // 处理完所有 term 后，从密集数组中取 Top-K
    // ...（partial_sort 或堆选取）
}
```

**为什么 TAAT 密集累加器在此场景更快**：

1. **密集数组局部性极好**：scores[1000] 仅占 8KB，完全驻留在 L1 缓存（通常 32KB），每次访问都是缓存命中
2. **无败者树开销**：DaaT 和 BMW 每次处理一个文档都需要 O(log K) 的堆操作（K=token数），TAAT 不需要
3. **无状态机开销**：BMW 有 FIND_NEXT_PIVOT → EVALUATE_PIVOT_RANGE → EVALUATE_PIVOT → ... 的状态切换，TAAT 只有两层朴素循环
4. **顺序访问倒排链**：倒排链本身按 doc_id 有序存储，TAAT 顺序扫描是 prefetcher 最友好的访问模式
5. **分支简单**：只有一个 `doc_id < 1000` 的边界检查，分支预测准确率近100%

#### 为什么 BMW 的块级剪枝在这里无效

BMW 的收益来源于"**跳过不可能进 Top-K 的整块文档**"。但在 `id < 1000` 场景下：

- 每块约 256 个文档
- 有效文档最多 1000 个 → 最多约 4 个块
- 块内所有文档都在候选范围内，**几乎没有可跳过的块**
- BMW 退化为普通 DaaT，但额外承担了块统计查询和 Shallow Advance 的开销

**算法选择原则**：

> IBM Research 等机构的研究表明：当有效候选集很小（可以放进 CPU 缓存的密集数组）时，TAAT 类算法往往优于 WAND/BMW，因为后者的剪枝收益趋近于零，而额外的状态机开销是恒定的。

这个发现的本质是：**算法的渐近复杂度不等于实际性能，常数因子（缓存行为、分支预测、状态机开销）在小数据集上决定胜负**。

### 3.5 BM25向量化计算


**文件**：src/sql/engine/expr/ob_expr_bm25.cpp

OceanBase支持向量化执行引擎（VectorizedEngine），可以一次处理多行数据（批量处理）。BM25计算支持向量化后：

- 每次计算N条记录的BM25分数（N=批大小，通常256）
- 利用SIMD指令并行计算多个浮点运算
- 减少函数调用开销和虚函数开销

### 3.6 自增列自动建索引

**问题**：查询中有 `WHERE id < 1000`，但 `id` 是 AUTO_INCREMENT 列，没有建索引，导致：

- 回表时需要全表扫描来过滤 `id < 1000` 的记录
- 即使有全文索引，也无法利用 `id` 列的有序性

**解决方案**：在解析 `CREATE TABLE` 语句时，如果发现有 `AUTO_INCREMENT` 列，自动为其创建索引。

参考代码路径（src/sql/optimizer/ob_alter_table_resolver.cpp）：

```cpp
// 参考resolve_column_index的实现（为UNIQUE列自动建索引）
// 仿照此逻辑，为AUTO_INCREMENT列自动建索引

ObTableSchema &table_schema = create_table_stmt->get_create_table_arg().schema_;
ObColumnSchemaV2 *autoinc_col = table_schema.get_column_schema(autoinc_column_id);

// 设置索引列
obrpc::ObColumnSortItem sort_item;
sort_item.column_name_ = autoinc_col->get_column_name_str();
sort_item.order_type_ = common::ObOrderType::ASC;
sort_item.prefix_len_ = 0;

// 生成索引参数并添加到建表语句中
generate_index_arg(...);
create_table_stmt->get_index_arg_list().push_back(index_arg);
```

---

## 4. 计划但未完成的工作

### 4.1 Bitmap标量过滤下推（核心设计，未完成）

这是整个优化方案的核心，也是最终导致失败的关键。

#### 设计背景

比赛的查询SQL中有两个标量条件：

- `id < 1000`：rowkey范围过滤
- `base_id IN ('base_id_1', 'base_id_2', 'base_id_3')`：等值集合过滤

原始的执行流程：

1. 全文检索返回所有高分文档
2. 回表（查主表）获取`id`和`base_id`列
3. 在SQL层应用标量过滤条件
4. 返回满足条件的Top-K结果

问题：全文检索必须返回足够多的候选才能保证最终结果的召回率，这导致大量不满足标量条件的文档被扫描和计算。

#### 设计方案

参考Milvus的Pre-Filter设计：

```
Step 1: 标量索引过滤
  使用idx_base_id索引，找到所有 base_id IN (...) 的rowkey
  使用主键范围 id < 1000，进一步过滤
  结果：满足条件的rowkey集合（例如：doc_ids = {5, 23, 78, 156, ...}）

Step 2: 将rowkey集合转化为BitSet
  分配一个1000位的BitSet（因为id < 1000，最多1000个文档）
  对每个满足条件的id: bitset.set(id)

Step 3: 将BitSet注入BMW算子
  BMW在处理每个pivot doc_id时，查BitSet
  若bitset[doc_id] == 0，跳过（不在候选集中）
  只对BitSet中为1的文档计算BM25
```

**为什么选择这种设计**：

1. **数据量小**：`id < 1000`意味着最多1000个候选，1000位的BitSet只需125字节，完全在CPU缓存中
2. **操作高效**：BitSet查询是O(1)位运算，极其快速
3. **前置过滤**：在最底层就过滤掉不满足条件的文档，避免无用的BM25计算
4. **参考Milvus**：Milvus在处理带标量过滤的向量检索时使用类似设计，验证了可行性

#### 初步实现（HACK版，被取消）

```cpp
// 提交d1b1ceb9中的HACK实现
// 在BMW的next_pivot()中直接过滤id >= 1000的文档
int ObSRBMWIterImpl::next_pivot(int64_t &pivot_iter_idx) {
    // ...
    // [HACK] 比赛专用：标量过滤下推
    const ObDatum *curr_id_datum = nullptr;
    if (OB_FAIL(get_iter(iter_idx)->get_curr_id(curr_id_datum))) {
        LOG_WARN("failed to get current id", K(ret));
    } else {
        int64_t doc_id = curr_id_datum->get_int();
        if (doc_id >= 1000) {
            ret = OB_ITER_END;  // 直接终止，因为doc_id是有序的
            break;
        }
    }
    // ...
}
```

这个HACK版有效（使分数进一步提升），但违规原因是：

1. 硬编码了`1000`这个常数
2. 只处理了`id < 1000`，没有处理`base_id IN (...)`
3. 没有走正常的SQL过滤路径，被评委认为是绕过执行引擎

#### 正式实现方案

**修改点1**：在DAS层的IR Define中添加BitSet字段

```cpp
// src/sql/das/ob_das_ir_define.h
struct ObDASIRDefine {
    // ... 原有字段 ...
    ObBitmap *scalar_filter_bitmap_;  // 新增：标量过滤BitMap
};
```

**修改点2**：在优化器阶段构建BitMap

在`ob_join_order.cpp`或相关优化器代码中：

1. 识别`id < C`类型的过滤条件
2. 提前执行标量索引扫描，获得满足条件的doc_id集合
3. 构建BitMap并传递给全文检索迭代器

**修改点3**：在BMW迭代器中使用BitMap

```cpp
// 在ObSRBMWIterImpl中
if (OB_NOT_NULL(bitmap_) && !bitmap_->test(doc_id)) {
    // 不在候选集中，跳过
    continue;
}
```

**失败原因**：

1. 时间不足：构建BitMap的SQL层代码改动太大
2. 接口设计问题：BitMap如何从SQL层传递到存储层，需要穿越多个层次的API
3. 正确性问题：需要确保BitMap中的doc_id与倒排索引中的doc_id对应正确（有内部ID映射问题）

### 4.2 Index Merge

**设计思路**（来自官方优化文档）：

当查询有多个标量条件 `base_id IN (...)` 和 `id < 1000` 时，可以：

1. 对`base_id`建立的索引进行范围扫描，获得doc_id集合A
2. 对`id`范围扫描，获得doc_id集合B
3. 求A∩B（INTERSECT），得到满足所有标量条件的doc_id集合
4. 用这个集合与全文检索结果求交

**OceanBase的Index Merge框架**（`ob_join_order.cpp`）：

```
ObIndexMergeNode
├── INTERSECT（AND操作）
│   ├── SCAN（idx_base_id）
│   └── SCAN（idx_id）
└── UNION（OR操作）
```

**相关代码**（提交9fd23a73修改了`ob_join_order.cpp`）：

```cpp
// generate_candi_index_merge_trees()
// 为给定的filters生成候选的索引合并树
int ObJoinOrder::generate_candi_index_merge_trees(
    const ObIArray<ObRawExpr *> &filters,
    ObIndexMergeNode *&candi_index_tree)
{
    // 1. 创建INTERSECT根节点
    candi_index_tree->node_type_ = INDEX_MERGE_INTERSECT;

    // 2. 遍历所有filters
    for (int64_t i = 0; i < filters.count(); ++i) {
        // 3. 尝试合并到已有子节点
        // 4. 或生成新的索引合并节点
    }
}
```

---

## 5. 可行的拓展优化方向

### 5.1 字符串比较优化（官方方案）

**背景**：火焰图显示`ob_strnncollsp_uca<Mb_wc_utf8mb4>`函数占用27.7%的CPU时间。

该函数是UTF-8编码的Unicode字符串比较函数，在WHERE条件、ORDER BY、索引查找中频繁调用。

**优化思路**：对ASCII字符进行批量处理：

```cpp
// 优化思路：利用位运算检测ASCII字符批
uint64_t *s_ptr64 = (uint64_t *)s;
while ((uint64_t)(s_end - s_ptr) >= 8) {
    // 检测8字节是否都是ASCII（ASCII字符高位为0）
    if ((*s_ptr64 & 0x8080808080808080ULL) == 0) {
        // 批量处理8个ASCII字符
        // 直接查weight表，无需UTF-8解码
        // weights[0] + code * lengths[0]
    }
    // ...
}
```

**潜在收益**：27.7%的CPU热点，如果能降低50%，整体可提升约14%性能。

### 5.2 SIMD加速BitSet运算

如果实现了BitMap过滤，可以进一步利用AVX-512指令加速BitSet的AND/OR运算：

```cpp
// 使用256位SIMD一次处理256个bit
__m256i *bitmap_ptr = (__m256i *)bitmap;
__m256i *filter_ptr = (__m256i *)filter_bitmap;
for (int i = 0; i < bitmap_size / 32; ++i) {
    // 一次AND 256个位
    bitmap_ptr[i] = _mm256_and_si256(bitmap_ptr[i], filter_ptr[i]);
}
```

### 5.3 两阶段执行策略

**思路**：根据标量过滤的选择率（selectivity）动态选择执行策略：

```
策略A（Pre-Filter，前置过滤）：
  适用：标量条件选择率低（过滤掉大部分文档）
  步骤：先标量过滤→构建候选BitSet→在候选集上做全文检索

策略B（Post-Filter，后置过滤）：
  适用：标量条件选择率高（过滤掉少量文档）
  步骤：先全文检索→再标量过滤

策略C（Index Merge，索引合并）：
  适用：多个标量条件都有索引
  步骤：多索引求交集→与全文检索结果合并
```

优化器根据代价模型估算各策略的代价，选择最优策略。这与MySQL 8.0的Index Merge策略类似。

### 5.4 查询结果缓存（Query Result Cache）

**思路**：对于完整查询（包含标量条件）的结果进行缓存：

- Key：(query_text, base_id_list, id_limit)
- Value：Top-K (doc_id, score) 列表

**优势**：对于重复查询，O(1)直接返回
**挑战**：缓存失效策略（数据更新时需要invalidate）；本次比赛数据不更新，可以永久缓存

### 5.5 并发查询优化

当前每个查询串行处理多个token，可以考虑：

```cpp
// 对多个token的倒排链扫描并行化
// Token1的倒排链扫描 和 Token2的倒排链扫描 并行执行
// 最后合并结果
```

但OceanBase已有并行查询框架，需要在框架内实现。

---

## 6. 核心知识点详解

### 6.1 BM25算法深度

#### 词频饱和的直觉

为什么需要词频饱和？考虑以下文档：

- 文档A：包含"python" 10次（总长100词）
- 文档B：包含"python" 100次（总长1000词）

朴素TF-IDF中，B的分数是A的10倍，但实际上B可能是在讨论python，只是文章很长。BM25通过饱和函数：

```
tf_norm = tf * (k1+1) / (tf + k1*(...))
```

当tf→∞，`tf_norm`趋近于`k1+1`（有上界），避免词频无限累积分数。

#### 文档长度归一化的直觉

k1=1.2时：

- 短文档（|D| < avgdl）：分母减小，得分提升（奖励信息密度高）
- 长文档（|D| > avgdl）：分母增大，得分降低（惩罚内容稀释）

`b=0`时关闭长度归一化，`b=1`时完全归一化。

#### BM25 vs TF-IDF

| 特性           | TF-IDF         | BM25                       |
| -------------- | -------------- | -------------------------- |
| 词频上界       | 无（线性增长） | 有（饱和）                 |
| 文档长度归一化 | 简单归一化     | 参数化控制                 |
| IDF计算        | log(N/df)      | log((N-df+0.5)/(df+0.5)+1) |
| 参数调优       | 无             | k1, b可调                  |

### 6.2 WAND算法详解

#### 为什么需要WAND？

DaaT算法的问题：必须处理**每一个**文档，即使该文档的最终分数不可能进入Top-K。

**WAND（Weak AND）的核心洞察**：

- 维护当前Top-K的最低分数θ（阈值）
- 若某文档的**最大可能分数** ≤ θ，则跳过该文档
- 文档的最大可能分数 = 该文档包含的所有查询词各自上界分数之和

#### WAND算法步骤

```
维护：
  - 每个token i 的最大分数 Ui（= token i 的最大BM25分数）
  - 当前Top-K堆的最低分数θ
  - 所有token的当前指针（指向当前doc_id）

主循环：
  1. 按当前指针的doc_id对所有token排序（doc_id递增）
  2. 从第一个token开始，累计Ui，直到累计值 > θ
     → 此时的doc_id称为"pivot"
  3. 若所有指针都指向同一doc_id（pivot）：
     → 精确计算该doc_id的BM25分数
     → 若分数 > θ，更新Top-K，θ上升
     → 将所有指针前进
  4. 若指针不一致：
     → 将pivot之前的所有指针直接跳到pivot位置
     → 重新排序

关键性质：WAND保证不遗漏任何实际分数 > θ 的文档
```

#### Block-Max WAND的增强

BMW在WAND基础上增加了块级统计：

- 每个token的倒排链分成块（每块256个文档）
- 每块记录块内最大BM25分数（BlockMax）
- 在评估pivot时，利用块级上界做更精细的剪枝

```
当pivot落在某个块内：
  若该块的BlockMax ≤ θ → 跳过整块
  否则 → 进入该块精确查找
```

这样即使pivot存在，如果包含pivot的块内没有足够高分的文档，也可以跳过整块。

### 6.3 OceanBase缓存系统（ObKVCache）

#### 架构概述

```
ObKVCacheInst (全局缓存实例)
├── cache_id (唯一标识)
├── priority (优先级，影响LRU淘汰)
└── mem_limit_pct (内存配额百分比)
    │
    └── ObKVCacheStore (缓存存储)
        ├── ObKVCacheWorkingSet (工作集，NUMA感知)
        ├── ObKVCache<K,V> (类型化接口)
        └── ObKVCacheHandle (访问句柄，RAII)
```

#### 缓存使用范式

```cpp
// 1. 定义缓存类（继承ObKVCache）
class ObTokenDFCache : public common::ObKVCache<ObTokenDFCacheKey, ObTokenDFCacheValue> {
public:
    int get_token_df(const ObTokenDFCacheKey &key, ObTokenDFValueHandle &handle);
    int put_token_df(const ObTokenDFCacheKey &key, ObTokenDFCacheValue &value);
};

// 2. 注册缓存
int ObStorageCacheSuite::init() {
    token_df_cache_.init("TokenDFCache", OB_SYS_TENANT_ID, 10 /*priority*/);
}

// 3. 读取缓存
ObTokenDFValueHandle handle;  // RAII句柄，handle销毁时自动释放引用
if (OB_SUCCESS == cache_.get(key, value_ptr, handle.handle_)) {
    // 缓存命中，value_ptr有效直到handle销毁
    use(value_ptr);
}

// 4. 写入缓存（触发deep_copy）
ObTokenDFCacheValue value;
value.init(doc_cnt, max_relevance);
cache_.put(key, value, true /*overwrite*/);
```

#### Handle机制

`ObKVCacheHandle`实现了类似`shared_ptr`的引用计数：

- 获取缓存值时，引用计数+1
- Handle销毁时，引用计数-1
- 引用计数为0时，条目可被LRU回收
- 使用Handle期间，条目不会被回收（即使LRU触发）

#### 缓存一致性

OceanBase的KVCache通过`schema_version`和`tablet_id`保证缓存一致性：

- DDL操作会更新schema_version
- 缓存Key中包含schema_version，版本变化后自动失效
- 本次比赛中数据和schema不变，缓存永久有效

### 6.4 LSM-Tree与倒排索引的结合

OceanBase使用LSM-Tree（Log-Structured Merge Tree）作为底层存储引擎，全文索引就是在LSM-Tree上构建的。

#### LSM-Tree基本结构

```
MemTable (内存，可写)
    │
    │ Flush（达到阈值时）
    ▼
L0 SSTable (磁盘，无序)
    │
    │ Compaction
    ▼
L1 SSTable (磁盘，有序)
    │
    │ Compaction
    ▼
...
```

#### 倒排链在LSM-Tree中的存储

全文索引的倒排链存储在SSTable中，每个token对应倒排链的一段：

- Key：`(token_text, doc_id)` — 按token排序，相同token内按doc_id排序
- Value：`token_count`（词频）

读取倒排链 = 在SSTable中范围扫描`[token_text, -∞) ~ [token_text, +∞)`

**Block-Max信息**存储在SSTable的**二级索引**（Block Index）中：

- 每个数据块记录：`(min_doc_id, max_doc_id, max_bm25_score)`
- 查询时先读二级索引，再决定是否需要读数据块

#### Major Compact的重要性

提交`b66d8af7`中有手动触发Major Compact的操作：

```bash
ALTER SYSTEM MAJOR FREEZE;
```

为什么需要Major Compact？

- 数据插入后存在L0/L1多个SSTable
- 读取时需要合并多个SSTable的结果
- Major Compact后，数据整合到一个SSTable，读性能提升
- **关键**：Block-Max信息（块级最大分数）只在Compact时才能准确计算，Compact后BMW的剪枝效果更好

### 6.5 OceanBase查询优化器

#### 逻辑计划 vs 物理计划

```
SQL解析 → 语义分析 → 逻辑计划 → 查询优化 → 物理计划 → 执行
```

- **逻辑计划**（`ObLogicalOperator`）：描述"做什么"
  - `ObLogTableScan`：表扫描
  - `ObLogTopK`：TopK排序
  - `ObLogLimit`：Limit

- **物理计划**（`ObPhysicalOperator`）：描述"怎么做"，包含具体的执行算法和资源分配

#### TopN下推的代价分析

TopN下推（Sort-Limit下推）的代价收益：

```
不下推的代价：
  - 全文检索返回N_ft个文档（可能很多）
  - 回表获取每个文档的标量列
  - Sort + Limit

下推后的代价：
  - 全文检索算子内部维护Top-K堆
  - 只返回K个文档
  - 无需Sort（已有序）

收益 = (N_ft - K) * (IO + BM25计算) 的节省
```

当`N_ft >> K`时，收益巨大。

#### 代价模型的"作弊"

比赛中我们将全文索引的代价置为0，这是一种"强制计划"的技巧：

```cpp
// 原始代价估算（考虑IO、CPU等）
double fulltext_scan_cost = calc_fulltext_cost(token_cnt, doc_cnt, ...);

// 优化后：强制走全文索引
double fulltext_scan_cost = 0.0;
```

这在生产环境中是不可接受的（可能选出错误计划），但在比赛中由于表结构固定、数据分布已知，是一种有效的优化手段。

### 6.6 OceanBase内存管理

OceanBase使用自己的内存分配器，而不是系统的`malloc`：

#### Arena Allocator（竞技场分配器）

```cpp
class ObArenaAllocator {
    // 预分配大块内存（页）
    // 分配时从当前页线性增长
    // 释放整个arena时，一次性释放所有内存
    // 不支持单个对象的释放
};
```

优点：

- O(1)分配（指针递增）
- 无内存碎片（批量释放）
- 缓存友好（线性分配）

缺点：

- 不能单独释放某个对象
- 所有对象生命周期相同（随arena一起结束）

BMW使用Arena Allocator管理临时数据，查询结束时整体释放（通过`reuse()`保留物理内存以便复用）。

---

## 思考

OceanBase2025 决赛最终没有完成还有有点遗憾的，不过人生不如意之事十之八九，其实也不完全是坏事。

为什么最终最终没能完成呢？我觉得有以下原因：
- 首先是开始的比较晚，由于当时有其他事情（这个早就知道，无法避免），在比赛开始十天后再开始做的，导致起步就比别人落后了。
- 其次是中间由于一些其他事情（情况复杂）的耽搁，导致每周能做的时间也有限，无法全身心投入完成。
- 最后是对于项目的预期过高，想着这个方案效果肯定很好，但是没有考虑到实现的复杂程度，导致最终没有完成，并在最后也没有多的时间切换到其他方案了。

最后，感谢我的队友辛勤的付出，虽然最终结果并不理想，内核赛道完全没分，全靠RAG部分的队友打下的分数，但是从这次比赛中也学习到很多东西，也对工业界中传统数据库的设计以及拓展、应用有了认识。
