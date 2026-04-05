---
title: "OSPP2025 回顾"
date: 2026-04-05T06:15:05Z
lastMod: 2026-04-05T06:15:05Z
draft: false # 是否为草稿
author: ["tkk"]

categories: ["Milvus", "SkipIndex", "VectorDB", "OSPP"]

tags: ["Milvus", "OSPP", "SkipIndex"]

keywords: ["Milvus", "OSPP", "SkipIndex"]

description: "OSPP2025 Milvus SkipIndex扩展以及持久化和预加载" # 文章描述，与搜索优化相关
summary: "OSPP2025 Milvus SkipIndex扩展以及持久化和预加载" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: false
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

<!-- more -->

# SkipIndex 开源之夏项目复习文档

> **项目名称**：file stats 透传 skip index，更丰富的 skip index \
> **项目组织**：Milvus（LF AI & Data Foundation）\
> **参与时间**：2025年6月 — 2025年10月 

---

## 第一章：项目概述与背景

### 1.1 项目目标

本项目来自 2025 年中国开源之夏（OSPP，Open Source Promotion Plan），所属组织为 Milvus 向量数据库。项目全称为"file stats 透传 skip index，更丰富的 skip index"，主要目标有两个：

1. **预构建与持久化**：在 IndexNode 为 Segment 构建索引时，预先构建 SkipIndex 并持久化存储，使 QueryNode 加载 Segment 时可以直接读取，无需重复动态构建，节省 QueryNode 的 CPU、带宽和 I/O 资源。

2. **丰富统计信息**：将 SkipIndex 的统计信息从简单的 MinMax 扩展到更丰富的类型，包括 Set（集合）、BloomFilter（布隆过滤器）、NgramBF（N-Gram 布隆过滤器）等，以支持更多查询类型的 Chunk 级跳过优化。

### 1.2 Milvus 整体架构

Milvus 是一个分布式向量数据库，采用存算分离的架构，核心组件分为两类：

**协调者（Coordinator）**：负责元数据管理和任务调度，不直接处理数据。

- **RootCoord**：管理 Collection/Schema 的生命周期，处理 DDL 请求。
- **DataCoord**：管理 Segment 的生命周期（Flushed/Indexed/Compacted 等状态），以及索引任务的调度。
- **QueryCoord**：管理查询资源，决定哪些 Segment 加载到哪些 QueryNode。

**工作节点（Node）**：负责实际执行任务。

- **Proxy**：用户面向接口，接收请求并路由到对应组件。
- **DataNode**：执行数据写入、Segment 压缩（Compaction）、排序（Sort）等任务。
- **QueryNode**：加载 Segment 并执行向量检索和标量过滤查询。
- **IndexNode**：执行索引构建任务（标量索引、向量索引）。
- **StreamingNode**：处理流式写入（WAL）。

所有组件接口定义在 `internal/types/types.go` 中。

### 1.3 Segment 的生命周期

理解 SkipIndex 的构建时机，需要先理解 Segment 的生命周期：

```
写入阶段：
  Insert → Growing Segment（内存中，无序）
           ↓ 数据量达到阈值或时间触发
  Flush → Sealed Segment（持久化到 S3/磁盘，无序）
           ↓ DataCoord 触发 Sort 任务
  Sort  → Sorted Segment（数据按主键排序，写入 Parquet 格式）
           ↓ DataCoord 触发 Index 任务
  Index → Indexed Segment（带向量索引和标量索引）

查询阶段：
  QueryCoord 分配 Segment → QueryNode 加载 Segment → 执行查询
```

**SkipIndex 的构建时机**（经过方案演变后）：在 Segment 执行 Sort 任务写入磁盘时，同步构建 SkipIndex 并随数据一起持久化。这样可以复用数据写入的 I/O，避免单独的读取开销。

### 1.4 SkipIndex 的作用

SkipIndex 是一种**粗粒度过滤**机制，工作在 Chunk（数据块）级别：

- Sealed Segment 的数据被划分为若干 Chunk（对应 Parquet 文件中的 RowGroup）
- 每个 Chunk 存储了该块数据的统计信息（最小值、最大值、布隆过滤器等）
- 执行查询时，先通过 SkipIndex 判断整个 Chunk 是否可以跳过
- 若可以跳过，则无需加载和扫描该 Chunk 的实际数据，节省 I/O 和 CPU

**与传统索引的区别**：

- 传统倒排索引：精确定位到具体的行，但构建开销大，占用更多内存
- SkipIndex：粗粒度跳过整个 Chunk，开销极小，是一种轻量级的优化手段
- 两者互补：有精确索引时用索引，无索引时用 SkipIndex 减少扫描量

### 1.5 项目时间线

| 时间         | 事件                                                         |
| ------------ | ------------------------------------------------------------ |
| 2025年6月    | 提交项目申请书，初步方案：SkipIndex 作为独立文件与 bm25stats 同级 |
| 2025年7月初  | 导师发来设计草稿（SkipIndex-Draft.md），方案改为嵌入 Parquet footer |
| 2025年7月中  | 大量阅读 Milvus 和 milvus-storage 源码，理清写入链路         |
| 2025年8月    | 忙完其他事务后，最终确定方案，开始编码                       |
| 2025年9月底  | 提交结项报告，提交 Issue #44584 和 PR #44581                 |
| 2025年10月   | 社区质疑 Parquet footer 方案的格式绑定问题，需改为独立文件方案 |
| 2025年10月底 | 时间不足，仅完成 SkipIndex 扩展部分，持久化链路未完全打通    |

---

## 第二章：SkipIndex 现有实现分析

### 2.1 整体设计架构

SkipIndex 的实现分布在以下几个文件中：

```
internal/core/src/index/
├── SkipIndex.h                          # 对外公开的 SkipIndex 类
├── SkipIndex.cpp                        # SkipIndex 方法实现
└── skipindex_stats/
    ├── SkipIndexStats.h                 # 统计信息类定义（FieldChunkMetrics 层次）
    ├── SkipIndexStats.cpp               # 统计信息构建实现
    └── utils.h                          # 辅助函数（类型判断、N-gram 提取）
```

整体设计遵循**关注点分离**原则：

- `FieldChunkMetrics`：存储一个 Chunk 的统计信息，并提供跳过判断接口（只读）
- `SkipIndexStatsBuilder`：从原始数据或 Parquet 统计信息构建 `FieldChunkMetrics`（写）
- `SkipIndex`：持有每个字段的 `CacheSlot<FieldChunkMetrics>`，提供查询接口（管理）
- `Translator`：连接 `CacheSlot` 和数据源，实现按需（Lazy）加载

### 2.2 Metrics 类型系统

在 `SkipIndexStats.h` 中，统计信息的值使用 `std::variant` 定义：

```cpp
// 文件: internal/core/src/index/skipindex_stats/SkipIndexStats.h

using Metrics = std::variant<bool,
                             int8_t,
                             int16_t,
                             int32_t,
                             int64_t,
                             float,
                             double,
                             std::string,
                             std::string_view>;
```

`std::variant` 是 C++17 的类型安全联合体（Tagged Union），相比 `union` 的优势：

- 类型安全：运行时知道当前存储的是哪种类型
- 无需手动管理构造/析构
- 可以与 `std::visit` 结合，以 visitor 模式处理不同类型

此外还有辅助类型别名：

```cpp
template <typename T>
using MetricsDataType =
    std::conditional_t<std::is_same_v<T, std::string>, std::string_view, T>;
```

这里使用了 `std::conditional_t`，当 T 是 `std::string` 时，返回 `std::string_view`（避免不必要的字符串拷贝）；否则返回 T 本身。

### 2.3 FieldChunkMetrics 类层次结构

`FieldChunkMetrics` 是抽象基类，定义了所有统计信息对象必须支持的接口：

```cpp
class FieldChunkMetrics {
 public:
    virtual std::unique_ptr<FieldChunkMetrics> Clone() const = 0;
    virtual FieldChunkMetricsType GetMetricsType() const = 0;

    // 一元范围查询：field OP val
    virtual bool CanSkipUnaryRange(OpType op_type, const Metrics& val) const = 0;

    // 二元范围查询：lower_val OP field OP upper_val
    virtual bool CanSkipBinaryRange(const Metrics& lower_val,
                                     const Metrics& upper_val,
                                     bool lower_inclusive,
                                     bool upper_inclusive) const {
        return false; // 默认不跳过（保守实现）
    }

    // IN 查询：field IN [v1, v2, ...]
    virtual bool CanSkipIn(const std::vector<Metrics>& values) const {
        return false;
    }

    // 缓存层接口
    cachinglayer::ResourceUsage CellByteSize() const;
    void SetCellSize(cachinglayer::ResourceUsage cell_size);

 private:
    cachinglayer::ResourceUsage cell_size_{0, 0};
};
```

**设计要点**：

- `Clone()` 方法：实现深拷贝，用于缓存层将 Cell 从 Translator 取出后复制
- 默认实现返回 `false`（不跳过），这是**保守策略**——宁可多扫描，不能漏数据
- `CanSkipBinaryRange` 和 `CanSkipIn` 有默认实现，子类可以选择是否重写

**具体子类**：

#### 2.3.1 NoneFieldChunkMetrics

```cpp
class NoneFieldChunkMetrics : public FieldChunkMetrics {
 public:
    FieldChunkMetricsType GetMetricsType() const override {
        return FieldChunkMetricsType::NONE;
    }
    bool CanSkipUnaryRange(OpType, const Metrics&) const override {
        return false; // 永远不跳过
    }
};
```

用于：

- 不支持的数据类型（如 JSON）
- 数据为空的 Chunk
- 类型不匹配时的 fallback

#### 2.3.2 BooleanFieldChunkMetrics

```cpp
class BooleanFieldChunkMetrics : public FieldChunkMetrics {
 private:
    bool has_true_ = false;
    bool has_false_ = false;

 public:
    FieldChunkMetricsType GetMetricsType() const override {
        return FieldChunkMetricsType::BOOLEAN;
    }
    bool CanSkipUnaryRange(OpType op_type, const Metrics& val) const override;
    bool CanSkipIn(const std::vector<Metrics>& values) const override;
};
```

布尔类型只有两个可能值，不需要 min/max，而是跟踪该 Chunk 中是否存在 true 和 false：

- `has_true_ = true`：该 Chunk 含有 true 值
- `has_false_ = true`：该 Chunk 含有 false 值

**跳过逻辑示例**：

- 查询 `field = true`，若 `has_true_ == false`，可以跳过
- 查询 `field IN [true]`，若 `has_true_ == false`，可以跳过

#### 2.3.3 FloatFieldChunkMetrics\<T\>

```cpp
template <typename T>  // T = float 或 double
class FloatFieldChunkMetrics : public FieldChunkMetrics {
 private:
    T min_;
    T max_;

 public:
    FieldChunkMetricsType GetMetricsType() const override {
        return FieldChunkMetricsType::FLOAT;
    }
    bool CanSkipUnaryRange(OpType op_type, const Metrics& val) const override;
    bool CanSkipBinaryRange(...) const override;
    bool CanSkipIn(const std::vector<Metrics>& values) const override;
};
```

浮点类型仅使用 min/max，**没有 Bloom Filter**。原因：

- 浮点数的精确相等比较本身就是危险的操作（浮点精度问题）
- Bloom Filter 对浮点等值查询效果有限
- 范围查询（>, <, BETWEEN）用 min/max 就足够了

#### 2.3.4 IntFieldChunkMetrics\<T\>

```cpp
template <typename T>  // T = int8/16/32/64
class IntFieldChunkMetrics : public FieldChunkMetrics {
 private:
    T min_;
    T max_;
    BloomFilterPtr bloom_filter_;  // 可选

 public:
    FieldChunkMetricsType GetMetricsType() const override {
        return FieldChunkMetricsType::INT;
    }
    bool CanSkipUnaryRange(OpType op_type, const Metrics& val) const override;
    bool CanSkipBinaryRange(...) const override;
    bool CanSkipIn(const std::vector<Metrics>& values) const override;
};
```

整数类型的特点：

- 同时使用 min/max（范围查询）和 Bloom Filter（等值查询）
- Bloom Filter 是可选的，低基数（取值少）的字段可以用 Set 替代
- 等值查询时先用 Bloom Filter，范围查询用 min/max

#### 2.3.5 StringFieldChunkMetrics

```cpp
class StringFieldChunkMetrics : public FieldChunkMetrics {
 private:
    std::string min_;
    std::string max_;
    BloomFilterPtr bloom_filter_;        // 用于等值查询
    BloomFilterPtr ngram_bloom_filter_;  // 用于 LIKE 查询

 public:
    FieldChunkMetricsType GetMetricsType() const override {
        return FieldChunkMetricsType::STRING;
    }
    bool CanSkipUnaryRange(OpType op_type, const Metrics& val) const override;
    bool CanSkipBinaryRange(...) const override;
    bool CanSkipIn(const std::vector<Metrics>& values) const override;
};
```

字符串类型最为复杂，包含：

- min/max：字符串的字典序比较，用于范围查询
- bloom_filter_：等值查询（`field = 'abc'`）
- ngram_bloom_filter_：LIKE 查询（`field LIKE '%abc%'`）

### 2.4 SkipIndex 核心 API

`SkipIndex` 类位于 `internal/core/src/index/SkipIndex.h`，是对外暴露的主接口：

```cpp
class SkipIndex {
 private:
    // 类型约束 trait
    template <typename T>
    struct IsAllowedType {
        static constexpr bool isAllowedType =
            std::is_integral<T>::value ||
            std::is_floating_point<T>::value ||
            std::is_same<T, std::string>::value ||
            std::is_same<T, std::string_view>::value;
        static constexpr bool isDisabledType =
            std::is_same<T, milvus::Json>::value ||
            std::is_same<T, bool>::value;
        // 整体是否支持：允许类型 且 非禁用类型
        static constexpr bool value = isAllowedType && !isDisabledType;
        // 是否支持算术运算（仅整数，且非 bool）
        static constexpr bool arith_value =
            std::is_integral<T>::value && !std::is_same<T, bool>::value;
        // 是否支持 IN 查询（比 value 宽松）
        static constexpr bool in_value = isAllowedType;
    };

    // 高精度类型推导：整数用 int64_t，其他保持原类型
    template <typename T>
    using HighPrecisionType =
        std::conditional_t<std::is_integral_v<T> && !std::is_same_v<bool, T>,
                           int64_t,
                           T>;
```

**IsAllowedType 的设计思路**：

- `isAllowedType`：必须是整数、浮点数或字符串类型（这些类型有天然的比较语义）
- `isDisabledType`：JSON 类型（结构复杂，没有单一比较语义）和 bool（bool 的范围比较无意义，用 BooleanFieldChunkMetrics 专门处理）
- 最终 `value = isAllowedType && !isDisabledType`，确保两个条件同时满足

**HighPrecisionType 的设计思路**：

- 算术运算（加减乘除）可能导致整数溢出
- 用 `int64_t` 作为中间计算类型，延迟溢出检测到最后
- 浮点数保持原类型（float/double 不需要精度提升）

#### 2.4.1 CanSkipUnaryRange — 一元范围查询

```cpp
template <typename T>
std::enable_if_t<SkipIndex::IsAllowedType<T>::value, bool>
CanSkipUnaryRange(FieldId field_id, int64_t chunk_id,
                  OpType op_type, const T& val) const {
    auto pw = GetFieldChunkMetrics(field_id, chunk_id);
    auto field_chunk_metrics = pw.get();
    return field_chunk_metrics->CanSkipUnaryRange(op_type, index::Metrics{val});
}
```

`std::enable_if_t` 实现了**基于类型约束的函数重载**（SFINAE）：

- 若 T 满足 `IsAllowedType<T>::value`，编译器选择此重载（返回实际跳过结果）
- 若 T 不满足，编译器选择另一个重载（直接返回 false，不触发错误）

#### 2.4.2 CanSkipBinaryArithRange — 算术范围查询

这是最复杂的一个接口，处理 `field OP value` 这种算术表达式，例如：

```sql
WHERE age + 5 > 30
WHERE salary * 2 < 100000
WHERE score / 10 >= 8
```

```cpp
template <typename T>
std::enable_if_t<SkipIndex::IsAllowedType<T>::arith_value, bool>
CanSkipBinaryArithRange(FieldId field_id, int64_t chunk_id,
                        OpType op_type, ArithOpType arith_type,
                        const HighPrecisionType<T> value,
                        const HighPrecisionType<T> right_operand) const {
    auto check_and_skip = [&](HighPrecisionType<T> new_value_hp, OpType new_op_type) {
        // 溢出检测：转换回 T 类型时是否会溢出？
        if constexpr (std::is_integral_v<T>) {
            if (new_value_hp > std::numeric_limits<T>::max() ||
                new_value_hp < std::numeric_limits<T>::min()) {
                return false; // 溢出，无法安全比较，保守策略：不跳过
            }
        }
        return CanSkipUnaryRange<T>(field_id, chunk_id, new_op_type,
                                    static_cast<T>(new_value_hp));
    };

    switch (arith_type) {
        case ArithOpType::Add:
            // field + C > V  =>  field > V - C
            return check_and_skip(value - right_operand, op_type);

        case ArithOpType::Sub:
            // field - C > V  =>  field > V + C
            return check_and_skip(value + right_operand, op_type);

        case ArithOpType::Mul:
            // field * C > V  =>  field > V / C（C != 0）
            // 若 C < 0，需要翻转比较运算符
            if (right_operand == 0) return false;
            OpType new_op = right_operand < 0 ?
                FlipComparisonOperator(op_type) : op_type;
            return check_and_skip(value / right_operand, new_op);

        case ArithOpType::Div:
            // field / C > V  =>  field > V * C（C != 0）
            if (right_operand == 0) return false; // 除以零
            OpType new_op = right_operand < 0 ?
                FlipComparisonOperator(op_type) : op_type;
            return check_and_skip(value * right_operand, new_op);
    }
}
```

**数学推导**：以 `field + C > V` 为例，等价于 `field > V - C`。这样就把对 `(field + C)` 的范围判断转化为对 `field` 的范围判断，再用 SkipIndex 中存储的 min/max 判断即可。

**负数翻转的数学依据**：对于乘法 `field * C > V`：

- 若 C > 0：两边除以 C，不等号方向不变：`field > V/C`
- 若 C < 0：两边除以 C（负数），不等号方向改变：`field < V/C`

### 2.5 RangeShouldSkip 跳过逻辑

核心跳过判断函数，位于 `SkipIndexStats.h`：

```cpp
template <typename T>
inline bool
RangeShouldSkip(const T& value,        // 查询值
                const T& lower_bound,  // Chunk 最小值
                const T& upper_bound,  // Chunk 最大值
                OpType op_type) {
    bool should_skip = false;
    switch (op_type) {
        case OpType::Equal: {
            // value == field？若 value 在 [min, max] 外，肯定无匹配
            should_skip = value > upper_bound || value < lower_bound;
            break;
        }
        case OpType::LessThan: {
            // field < value？若 value <= min，则所有 field 都 >= value，无匹配
            should_skip = value <= lower_bound;
            break;
        }
        case OpType::LessEqual: {
            // field <= value？若 value < min，则所有 field > value，无匹配
            should_skip = value < lower_bound;
            break;
        }
        case OpType::GreaterThan: {
            // field > value？若 value >= max，则所有 field <= value，无匹配
            should_skip = value >= upper_bound;
            break;
        }
        case OpType::GreaterEqual: {
            // field >= value？若 value > max，则所有 field < value，无匹配
            should_skip = value > upper_bound;
            break;
        }
    }
    return should_skip;
}
```

**为什么这里的逻辑是正确的**？以 `Equal` 为例：

- Chunk 中的所有值都在 `[lower_bound, upper_bound]` 范围内
- 若查询值 `value` 在此范围外，则 Chunk 中不可能存在等于 `value` 的记录
- 因此可以安全跳过整个 Chunk

### 2.6 缓存层架构

SkipIndex 采用了 Milvus 的缓存层（Caching Layer）机制，避免重复计算和内存浪费：

```
SkipIndex
  └── fieldChunkMetrics_: unordered_map<FieldId, CacheSlot<FieldChunkMetrics>>
            │
            └── CacheSlot<FieldChunkMetrics>
                    │
                    └── Translator（数据来源）
                          ├── FieldChunkMetricsTranslator
                          │     （从 ChunkedColumn 按需计算）
                          └── FieldChunkMetricsTranslatorFromStatistics
                                （从 Parquet Statistics 预计算）
```

**CacheSlot 的工作原理**：

- `CacheSlot` 是一个带有缓存的数据容器
- 数据按 `cid_t`（Cell ID，对应 chunk_id）组织
- 首次访问 chunk_id 时，通过 `Translator::get_cells()` 加载数据到缓存
- 后续访问命中缓存，直接返回

**PinWrapper**：RAII 风格的缓存引用计数器：

```cpp
const cachinglayer::PinWrapper<const index::FieldChunkMetrics*>
GetFieldChunkMetrics(FieldId field_id, int chunk_id) const;
```

`PinWrapper` 确保在使用期间缓存不会被驱逐：

- 构造时：增加引用计数，"固定"对应的 Cache Cell
- 析构时：减少引用计数，允许 GC 驱逐

**两种 Translator 对比**：

| 特性     | FieldChunkMetricsTranslator         | FieldChunkMetricsTranslatorFromStatistics |
| -------- | ----------------------------------- | ----------------------------------------- |
| 数据来源 | ChunkedColumnInterface（内存数据）  | Parquet Statistics（已预计算）            |
| 计算时机 | 首次访问时按需计算（Lazy）          | 构造时就已全部计算（Eager）               |
| 适用场景 | 动态构建（无预构建数据时 fallback） | 预构建（已持久化 SkipIndex 时加载）       |
| 性能     | 首次慢，后续快                      | 构造时快，全程快                          |

---

## 第三章：表达式层中 SkipIndex 的使用

### 3.1 总体集成模式

SkipIndex 通过回调函数（`skip_func`）集成到表达式求值框架中：

```cpp
// 典型使用模式（伪代码）
auto skip_func = [&](const SkipIndex& skip_index,
                     FieldId field_id,
                     int chunk_id) -> bool {
    return skip_index.CanSkipXxx(field_id, chunk_id, ...);
};

// 传递给 ProcessDataChunks
ProcessDataChunks(filter_func, skip_func, ...);
```

表达式求值时，对每个 Chunk：

1. 先调用 `skip_func` 检查是否可以跳过
2. 若可以跳过，设置该 Chunk 所有行的结果为 false（不匹配），更新内部游标
3. 若不能跳过，调用实际的 `filter_func` 对每行数据进行过滤

这种设计的优点：

- **正交性**：SkipIndex 逻辑与过滤逻辑完全分离
- **可选性**：`skip_func` 是可选参数，不影响正确性
- **可扩展性**：新增 SkipIndex 类型只需修改 `skip_func`，不影响表达式框架

### 3.2 PhyTermFilterExpr（IN 查询）

位于 `internal/core/src/exec/expression/TermExpr.cpp`：

```cpp
// IN 查询的 SkipIndex 优化
auto skip_func = [&](const SkipIndex& skip_index,
                     FieldId field_id,
                     int chunk_id) -> bool {
    // 对每个查询值，检查是否存在于该 Chunk
    return skip_index.CanSkipInQuery<T>(field_id, chunk_id, terms_);
};
```

`CanSkipInQuery` 的工作原理：

- 将所有查询值打包成 `vector<Metrics>`
- 调用 `FieldChunkMetrics::CanSkipIn()`
- 对于 IntFieldChunkMetrics：先用 Bloom Filter 检查每个值，若所有值都不在 BF 中，则跳过
- 对于 FloatFieldChunkMetrics：只能用 min/max 范围，若所有值都在范围外，则跳过

### 3.3 BinaryArithOpEvalRangeExpr（算术范围查询）

位于 `internal/core/src/exec/expression/BinaryArithOpEvalRangeExpr.cpp`：

```cpp
// 处理 field + C > V 这类表达式
auto skip_func = [&](const SkipIndex& skip_index,
                     FieldId field_id,
                     int chunk_id) -> bool {
    return skip_index.CanSkipBinaryArithRange<T>(
        field_id, chunk_id, op_type_, arith_op_type_,
        value_, right_operand_);
};
```

### 3.4 LogicalUnaryExpr（逻辑否定）

位于 `internal/core/src/exec/expression/LogicalUnaryExpr.cpp`：

```cpp
// NOT 表达式的处理
// NOT (field > V) => field <= V
// 注意：SkipIndex 的跳过逻辑对 NOT 表达式需要取反
```

### 3.5 JsonContainsExpr（JSON 包含查询）

位于 `internal/core/src/exec/expression/JsonContainsExpr.cpp`：

```cpp
// JSON 字段的 CONTAINS 查询
// 利用 JsonKeyStats（JSON 键统计）进行 Chunk 级跳过
```

### 3.6 ProcessDataChunks 的执行流程

`Expr.h` 中定义了通用的 Chunk 处理框架：

```cpp
// 简化示意
template <typename Func, typename SkipFunc>
void ProcessDataChunks(Func filter_func, SkipFunc skip_func) {
    auto& skip_index = segment_->GetSkipIndex();

    for (int chunk_id = 0; chunk_id < num_chunks; ++chunk_id) {
        // 先检查是否可以跳过
        if (skip_func && skip_func(skip_index, field_id_, chunk_id)) {
            // 跳过：将这个 Chunk 的所有行标记为不匹配
            // 但仍需更新内部游标（bitset 位置等）
            HandleSkippedChunk(chunk_id);
            continue;
        }

        // 不能跳过：实际过滤
        auto chunk_data = GetChunkData(chunk_id);
        for (int row = 0; row < chunk_size; ++row) {
            result[offset + row] = filter_func(chunk_data[row]);
        }
        offset += chunk_size;
    }
}
```

**跳过时的处理**：被跳过的 Chunk 不是什么都不做，而是将其结果设为全 false，并正确更新内部状态（如 bitset 偏移量），以保证后续 Chunk 的结果正确写入到 bitset 的正确位置。

---

## 第四章：存储链路分析

### 4.1 Milvus Storage V2 架构

Milvus 的存储体系分为 V1 和 V2 两个版本：

| 特性                | Storage V1                     | Storage V2                                                   |
| ------------------- | ------------------------------ | ------------------------------------------------------------ |
| 底层格式            | 自定义二进制格式（binlog）     | Parquet（通过 milvus-storage）                               |
| Parquet 库          | 不使用                         | Apache Arrow Parquet（Rust 实现，通过 milvus-storage 跨语言调用） |
| 文件组织            | 每个字段每个 Chunk 一个文件    | 列分组（Column Group），多个 RowGroup，~1MB/RowGroup         |
| 写入方式            | Milvus 直接控制，逐行/逐批写入 | 通过 milvus-storage 库，攒批（buffer）后整 RowGroup 写入     |
| 元数据              | 写入 Milvus 自有格式头         | Parquet footer（key-value metadata，可扩展）                 |
| 耦合程度            | Milvus 直接控制写入细节        | 通过 milvus-storage 库接口解耦，Milvus 不感知 RowGroup 边界  |
| Chunk/RowGroup 对应 | 每个文件即一个 Chunk           | 读取阶段每个 RowGroup 对应一个 Chunk（write phase 无 Chunk 概念） |

**Storage V1 格式说明**：V1 使用 Milvus 自定义的 binlog 格式，每个字段单独存储为一个文件（insert binlog），格式头包含数据类型、行数等元信息，后续跟实际数据的二进制编码。这种格式由 Milvus 完全自主控制，但不能利用 Parquet 的列式存储优化（如编码压缩、谓词下推、原生 Statistics 等）。

**Storage V2 的关键改进**：

- **Parquet 格式**：利用列式存储的压缩和编码优势，降低存储和 I/O 开销
- **milvus-storage 库**：独立仓库，封装了 Parquet 的读写细节；其底层 Parquet 库使用 **Rust 实现**（通过 Apache Arrow 的 Rust 绑定），并在 milvus-storage 内部通过 **FFI（Foreign Function Interface）跨语言调用**，Go/C++ 侧通过 CGO 调用 milvus-storage 的 C 接口
- **攒批写入（Buffered Write）**：`PackedRecordBatchWriter` 将多个 RecordBatch 积累到内存缓冲区，达到约 1MB 阈值后一次性写入一个完整的 RowGroup，减少写放大和小 I/O
- **可扩展元数据**：Parquet footer 的 key-value metadata 可存储自定义统计信息（如 SkipIndex），是本项目 Parquet footer 方案的基础

**Column Group 的概念**：Storage V2 将字段按类型分组到不同的 Parquet 文件：

- 宽列（向量字段、文本字段）：每个字段独占一个 Column Group
- 窄列（标量字段）：所有标量字段共享一个 Column Group

```
Segment 写入 →  Column Group 0（标量字段：id, age, name, ...）→ file_0.parquet
               Column Group 1（向量字段：embedding）           → file_1.parquet
               Column Group 2（文本字段：content）             → file_2.parquet
```

### 4.2 数据写入链路全景

从 DataNode 发起到 Parquet 文件落盘的完整链路：

```
DataNode (Go)
  └── Sort Task 触发
       └── PackedBinlogRecordWriter.Write(Record)    [record_writer.go]
            └── packedRecordWriter.Write(Record)     [record_writer.go]
                 └── PackedWriter.WriteRecordBatch() [packed_writer.go, CGO]
                      └── C.WritePackedRecordBatch() [CGO 桥接]
                           └── C++: milvus_storage::PackedRecordBatchWriter::Write()
                                └── ParquetFileWriter.WriteRowGroup()
                                     ├── 积累到 buffer_size 阈值
                                     ├── 调用 MetadataBuilder.Append(batch)
                                     └── 写入 Parquet RowGroup
```

**关键点**：milvus-storage 中的 `PackedRecordBatchWriter` 是真正控制 RowGroup 边界的地方。它积累 RecordBatch，当内存占用达到阈值（默认约 1MB）时，触发 RowGroup 的写入。这就是 SkipIndex 需要在这里构建的原因。

### 4.3 Go 层的 packedRecordWriter

文件：`internal/storage/record_writer.go`

```go
type packedRecordWriter struct {
    writer       *packed.PackedWriter  // CGO 桥接
    bufferSize   int64
    columnGroups []storagecommon.ColumnGroup
    pathsMap     map[typeutil.UniqueID]string
    schema       *schemapb.CollectionSchema
    arrowSchema  *arrow.Schema
    rowNum       int64
    writtenUncompressed     uint64
    columnGroupUncompressed map[typeutil.UniqueID]uint64
    columnGroupCompressed   map[typeutil.UniqueID]uint64
    storageConfig *indexpb.StorageConfig
}

func (pw *packedRecordWriter) Write(r Record) error {
    var rec arrow.Record
    // 处理普通 Record 转换为 Arrow Record...
    // 统计未压缩大小
    pw.rowNum += int64(r.Len())
    for col, arr := range rec.Columns() {
        size := calculateActualDataSize(arr)
        pw.writtenUncompressed += size
        // 按 ColumnGroup 分类统计
        for _, columnGroup := range pw.columnGroups {
            if lo.Contains(columnGroup.Columns, col) {
                pw.columnGroupUncompressed[columnGroup.GroupID] += size
                break
            }
        }
    }
    // 实际写入：调用 CGO
    return pw.writer.WriteRecordBatch(rec)
}
```

Go 层的主要职责：

1. 将 Milvus 内部的 `Record` 转换为 Arrow `Record`（使用 Arrow 标准格式）
2. 统计写入的数据量（用于 binlog 元数据记录）
3. 将 Arrow Record 通过 CGO 传递给 C++ 层

### 4.4 CGO 桥接层（packed_writer.go）

文件：`internal/storagev2/packed/packed_writer.go`

```go
func NewPackedWriter(filePaths []string, schema *arrow.Schema, ...) (*PackedWriter, error) {
    // Step 1: 将 Go string 转换为 C string
    cFilePaths := make([]*C.char, len(filePaths))
    for i, path := range filePaths {
        cFilePaths[i] = C.CString(path)
        defer C.free(unsafe.Pointer(cFilePaths[i]))  // RAII: 函数返回时释放
    }

    // Step 2: 导出 Arrow Schema 到 C Data Interface
    var cas cdata.CArrowSchema
    cdata.ExportArrowSchema(schema, &cas)
    cSchema := (*C.struct_ArrowSchema)(unsafe.Pointer(&cas))
    defer cdata.ReleaseCArrowSchema(&cas)

    // Step 3: 构建 Column Splits（列分组信息）
    cColumnSplits := C.NewCColumnSplits()
    for _, group := range columnGroups {
        // 分配 C 内存存储列索引数组
        cGroup := C.malloc(C.size_t(len(group.Columns)) * C.size_t(unsafe.Sizeof(C.int(0))))
        defer C.free(cGroup)
        cGroupSlice := (*[1 << 30]C.int)(cGroup)[:len(group.Columns):len(group.Columns)]
        for i, val := range group.Columns {
            cGroupSlice[i] = C.int(val)
        }
        C.AddCColumnSplit(cColumnSplits, (*C.int)(cGroup), C.int(len(group.Columns)))
    }

    // Step 4: 调用 C++ 创建 PackedWriter
    var cPackedWriter C.CPackedWriter
    status = C.NewPackedWriterWithStorageConfig(cSchema, cBufferSize, ...)
    if err := ConsumeCStatusIntoError(&status); err != nil {
        return nil, err
    }
    return &PackedWriter{cPackedWriter: cPackedWriter}, nil
}

func (pw *PackedWriter) WriteRecordBatch(recordBatch arrow.Record) error {
    // 将每一列导出为 Arrow C Data Interface
    cArrays := make([]CArrowArray, recordBatch.NumCols())
    cSchemas := make([]CArrowSchema, recordBatch.NumCols())
    for i := range recordBatch.NumCols() {
        var caa cdata.CArrowArray
        var cas cdata.CArrowSchema
        // ExportArrowArray：零拷贝，通过指针传递数据
        cdata.ExportArrowArray(recordBatch.Column(int(i)), &caa, &cas)
        cArrays[i] = *(*CArrowArray)(unsafe.Pointer(&caa))
        cSchemas[i] = *(*CArrowSchema)(unsafe.Pointer(&cas))
    }
    // 调用 C++ 写入
    status = C.WritePackedRecordBatch(&pw.cPackedWriter, ...)
    return ConsumeCStatusIntoError(&status)
}
```

**Arrow C Data Interface** 是实现 Go → C++ 零拷贝数据传递的关键：

- 定义了一套标准的 C 结构体（`ArrowSchema`, `ArrowArray`）
- 数据通过指针传递，不进行数据拷贝
- 使用引用计数管理内存，由释放回调（`release` 函数指针）处理

### 4.5 BM25Stats 参照模式

BM25Stats 是 SkipIndex 设计时参照的统计信息构建模式，理解它有助于理解 SkipIndex 的设计思路。

**BM25Stats 的 Go 层实现**（`internal/storage/stats.go`）：

```go
type BM25Stats struct {
    rowsWithToken map[uint32]int32  // token ID → 含该 token 的行数
    numRow        int64              // 总行数
    numToken      int64              // 总 token 数
}

// 增量更新（每次写入一条记录时调用）
func (m *BM25Stats) AppendBytes(datas ...[]byte) {
    for _, data := range datas {
        dim := typeutil.SparseFloatRowElementCount(data)
        for i := 0; i < dim; i++ {
            index := typeutil.SparseFloatRowIndexAt(data, i)
            value := typeutil.SparseFloatRowValueAt(data, i)
            m.rowsWithToken[index] += 1        // 该 token 出现的行数 +1
            m.numToken += int64(value)          // 累加 token 权重
        }
        m.numRow += 1
    }
}

// 二进制序列化（版本兼容格式）
func (m *BM25Stats) Serialize() ([]byte, error) {
    buffer := bytes.NewBuffer(...)
    binary.Write(buffer, common.Endian, BM25VERSION)  // 版本号
    binary.Write(buffer, common.Endian, m.numRow)      // 行数
    binary.Write(buffer, common.Endian, m.numToken)    // token 总数
    for key, value := range m.rowsWithToken {
        binary.Write(buffer, common.Endian, key)       // token ID
        binary.Write(buffer, common.Endian, value)     // 该 token 出现行数
    }
    return buffer.Bytes(), nil
}
```

**StatsCollector 接口模式**（`internal/storage/stats_collector.go`）：

```go
type StatsCollector interface {
    Collect(r Record) error                                               // 阶段1：积累
    Digest(...) (map[FieldID]*datapb.FieldBinlog, error)                  // 阶段2：完成
}

// BM25 统计收集器
type Bm25StatsCollector struct {
    bm25Stats map[FieldID]*BM25Stats
}

func (c *Bm25StatsCollector) Collect(r Record) error {
    for fieldID, stats := range c.bm25Stats {
        field, _ := r.Column(fieldID).(*array.Binary)
        for i := 0; i < r.Len(); i++ {
            stats.AppendBytes(field.Value(i))  // 增量更新
        }
    }
    return nil
}

func (c *Bm25StatsCollector) Digest(...) (...) {
    for fid, stats := range c.bm25Stats {
        bytes, _ := stats.Serialize()          // 序列化
        // 写入独立的 stats 文件（作为 binlog）
        blobsWriter(blobs)
    }
}
```

BM25Stats 的存储方式（**最终作为独立 binlog 文件**）正是 SkipIndex 最初方案和最终决定的方案：作为独立文件，与数据文件（insert binlog）并列，通过 Milvus 的元数据管理系统（etcd）记录其存在。

---

## 第五章：milvus-storage Metadata 接口设计

### 5.1 设计背景与核心挑战

这是整个项目中最具挑战性的技术决策，也是最终的创新点之一。

**核心问题**：SkipIndex 的构建需要在每个 RowGroup 写入时触发，但：

- RowGroup 的划分逻辑在 milvus-storage 内部（`PackedRecordBatchWriter`）
- milvus-storage 是一个独立仓库，对 Milvus 不可见
- 不能在 milvus-storage 中直接引用 Milvus 的 SkipIndex 类（破坏解耦）
- 不能仅为 SkipIndex 设计一个特殊接口（不够通用，难以被社区接受）

**解决思路**：设计一个**通用的元数据构建接口**，让 milvus-storage 只知道有一个"元数据构建者"（MetadataBuilder），而不知道具体是什么元数据。Milvus 这边实现 SkipIndex 版本的 MetadataBuilder，注册到 milvus-storage 的写入流水线中。

### 5.2 Metadata 抽象类

文件：`milvus-storage/cpp/include/milvus-storage/common/metadata.h`

```cpp
class Metadata {
  public:
  virtual ~Metadata() = default;

  // 序列化为字符串（用于存储到 Parquet footer）
  virtual std::string Serialize() const = 0;

  // 从字符串反序列化（用于从 Parquet footer 读取）
  virtual void Deserialize(const std::string_view data) = 0;
};
```

设计特点：

- **最小化接口**：只有两个纯虚函数，极其简洁
- **字符串序列化**：使用 `std::string` 和 `std::string_view`，不依赖任何序列化框架
- **无状态接口**：不持有数据，只定义 I/O 契约

### 5.3 MetadataBuilder 设计

```cpp
class MetadataBuilder {
  private:
  struct MetadataHeader {
    uint32_t magic = 0;    // 魔数，用于数据校验
    uint32_t version = 0;  // 版本号，用于向前兼容
    uint32_t count = 0;    // 条目数量（= RowGroup 数量）
  };

  static constexpr uint32_t kMagicNumber = 0x4D424C44;  // "MBLD" = Metadata BuiLD
  static constexpr uint32_t kCurrentVersion = 1;

  public:
  // 每写入一个 RowGroup 时调用
  void Append(const std::vector<std::shared_ptr<arrow::RecordBatch>>& batch) {
    metadata_collection_.emplace_back(Create(batch));  // 调用子类的 Create
  }

  // 写入完成后调用，返回序列化的整个 metadata 集合
  std::string Finish() {
    return MetadataBuilder::Serialize(metadata_collection_);
  }

  // 静态序列化方法：生成带 header 的二进制格式
  static std::string Serialize(
      const std::vector<std::unique_ptr<Metadata>>& metadata_list) {
    std::stringstream ss(std::ios::binary | std::ios::out);

    // 写入 header（固定 12 字节）
    MetadataHeader header;
    header.magic = kMagicNumber;
    header.version = kCurrentVersion;
    header.count = metadata_list.size();
    ss.write(reinterpret_cast<const char*>(&header), sizeof(header));

    // 写入每个 Metadata（长度前缀 + 数据）
    for (const auto& meta : metadata_list) {
        std::string data = meta->Serialize();
        uint32_t len = data.length();
        ss.write(reinterpret_cast<const char*>(&len), sizeof(len));  // 4字节长度
        ss.write(data.data(), len);                                    // 实际数据
    }
    return ss.str();
  }

  // 模板反序列化：从 Parquet footer 的字符串还原 Metadata 列表
  template <typename MetadataT>
  static std::vector<std::unique_ptr<MetadataT>> Deserialize(
      const std::string_view data) {
    // 读取 header，验证 magic 和 version
    MetadataHeader header;
    std::memcpy(&header, data.data() + offset, sizeof(header));

    if (header.magic != kMagicNumber || header.version != kCurrentVersion) {
        return {};  // 格式不匹配，安全返回空
    }

    std::vector<std::unique_ptr<MetadataT>> result;
    result.reserve(header.count);

    // 逐个读取：先读 4 字节长度，再读数据
    for (uint32_t i = 0; i < header.count; ++i) {
        uint32_t len;
        std::memcpy(&len, data.data() + offset, sizeof(len));
        offset += sizeof(uint32_t);

        std::string_view meta_data(data.data() + offset, len);
        offset += len;

        auto meta = std::make_unique<MetadataT>();
        meta->Deserialize(meta_data);  // 调用具体子类的 Deserialize
        result.emplace_back(std::move(meta));
    }
    return result;
  }

  protected:
  // 纯虚方法：子类实现具体的 Metadata 对象创建逻辑
  virtual std::unique_ptr<Metadata> Create(
      const std::vector<std::shared_ptr<arrow::RecordBatch>>& batch) = 0;

  std::vector<std::unique_ptr<Metadata>> metadata_collection_;
};
```

### 5.4 二进制格式设计

序列化后的二进制格式如下：

```
+----------------+----------------+----------------+
|  magic (4B)    |  version (4B)  |  count (4B)    |  <- Header (12 bytes)
+----------------+----------------+----------------+
|  len_0 (4B)    |  data_0 (len_0 bytes)           |  <- Metadata 0
+----------------+----------------------------------+
|  len_1 (4B)    |  data_1 (len_1 bytes)           |  <- Metadata 1
+----------------+----------------------------------+
|  ...                                             |
+-------------------------------------------------- +
|  len_n (4B)    |  data_n (len_n bytes)           |  <- Metadata n
+----------------+----------------------------------+
```

**魔数 0x4D424C44 的含义**：
将 4 字节十六进制值按 ASCII 解码：`0x4D='M', 0x42='B', 0x4C='L', 0x44='D'` → "MBLD"，即 "Metadata BuiLD" 的缩写。这是一种常见的**魔数（Magic Number）**设计，用于快速识别文件/数据格式。

### 5.5 PackedFileMetadata — 读取接口

文件：`milvus-storage/cpp/include/milvus-storage/common/metadata.h`

```cpp
class PackedFileMetadata {
  public:
  // 模板方法：从 Parquet footer 中读取指定 key 对应的 Metadata 列表
  template <typename MetadataT>
  std::vector<std::unique_ptr<MetadataT>> GetMetadataVector(
      std::string_view key) const {
    // 从 Parquet 文件的 key-value metadata 中读取
    auto key_value_metadata = parquet_metadata_->key_value_metadata();
    auto metadata = key_value_metadata->Get(key);
    if (!metadata.ok()) {
        return {};  // key 不存在（向后兼容）
    }
    const std::string& metadata_str = metadata.ValueOrDie();
    // 调用 MetadataBuilder::Deserialize 进行反序列化
    return MetadataBuilder::Deserialize<MetadataT>(
        std::string_view(metadata_str));
  }

  private:
  std::shared_ptr<parquet::FileMetaData> parquet_metadata_;
  RowGroupMetadataVector row_group_metadata_;
  std::map<FieldID, ColumnOffset> field_id_mapping_;
  GroupFieldIDList group_field_id_list_;
  std::string storage_version_;
};
```

**使用示例**（Milvus 中如何读取 SkipIndex）：

```cpp
// 假设 ChunkSkipIndex 是 Metadata 的子类
auto packed_metadata = column_group->GetPackedFileMetadata();
std::vector<std::unique_ptr<ChunkSkipIndex>> skip_indices =
    packed_metadata->GetMetadataVector<ChunkSkipIndex>("skip_index");

// 将加载的 SkipIndex 注册到 SkipIndex 对象
for (size_t chunk_id = 0; chunk_id < skip_indices.size(); ++chunk_id) {
    skip_index.RegisterPrebuilt(field_id, chunk_id, std::move(skip_indices[chunk_id]));
}
```

### 5.6 设计权衡：虚函数 vs CRTP

在设计 `MetadataBuilder` 时，我考虑了两种方案：

**方案一：虚函数（最终选择）**

```cpp
class MetadataBuilder {
  protected:
  virtual std::unique_ptr<Metadata> Create(...) = 0;  // 虚函数
};
```

**方案二：CRTP（Curiously Recurring Template Pattern）**

```cpp
template <typename Derived>
class MetadataBuilder {
  protected:
  // 静态多态，编译时确定
  std::unique_ptr<Metadata> Create(...) {
      return static_cast<Derived*>(this)->CreateImpl(...);
  }
};
```

最终选择虚函数的原因：

1. **异构容器需求**：ParquetFileWriter 需要持有多个不同类型的 MetadataBuilder，需要类型擦除
2. **复杂度**：CRTP 会导致模板参数层层传递，增加代码复杂度
3. **性能**：`Create()` 是每个 RowGroup 调用一次，频率较低，虚函数开销可忽略
4. **可维护性**：虚函数对使用者更友好，没有模板知识也能理解

---

## 第六章：SkipIndex 扩展设计

### 6.1 统计信息类型全览

在我的方案中，SkipIndex 被扩展以支持以下统计信息类型：

| 类型        | 适用场景                  | 数据结构           | 误判率       | 内存开销     |
| ----------- | ------------------------- | ------------------ | ------------ | ------------ |
| MinMax      | 范围查询（>, <, BETWEEN） | 2个值（min + max） | 0%           | 极小         |
| Set         | 低基数等值/IN 查询        | 哈希集合           | 0%           | 与基数成正比 |
| BloomFilter | 高基数等值/IN 查询        | 位数组             | 1%（可配置） | ~1.14MB/1M行 |
| NgramBF     | LIKE 查询（模式匹配）     | N-gram 位数组      | 概率性       | 较大         |
| TokenBF     | 全文搜索（已决定去除）    | Token 位数组       | 概率性       | 较大         |

**TokenBF 被去除的原因**：与 Milvus 中已有的 `TextMatchIndex`（全文搜索索引）功能重叠，且 SkipIndex 作为轻量级过滤手段，不应承担全文搜索的职责。

### 6.2 构建策略选择逻辑

```cpp
// 伪代码：根据数据类型决定构建哪些统计信息
FieldChunkMetrics* BuildMetrics(DataType type, const ColumnData& data) {
    switch (type) {
        case BOOL:
            // 只需要 has_true / has_false
            return BuildBoolMetrics(data);

        case INT8/INT16/INT32/INT64:
            // 首先构建 MinMax
            auto metrics = new IntFieldChunkMetrics(min, max);
            // 根据基数决定 Set 还是 BloomFilter
            if (data.cardinality() < SET_THRESHOLD) {
                metrics->BuildSet(data);       // 低基数：精确集合
            } else {
                metrics->BuildBloomFilter(data);  // 高基数：概率过滤器
            }
            return metrics;

        case FLOAT/DOUBLE:
            // 只有 MinMax，不使用 BloomFilter（浮点精度问题）
            return new FloatFieldChunkMetrics(min, max);

        case VARCHAR/STRING:
            // MinMax（字符串字典序）
            auto metrics = new StringFieldChunkMetrics(min, max);
            // 等值查询优化：Set 或 BloomFilter
            if (data.cardinality() < SET_THRESHOLD) {
                metrics->BuildSet(data);
            } else {
                metrics->BuildBloomFilter(data);
            }
            // LIKE 查询优化：NgramBF
            metrics->BuildNgramBloomFilter(data);
            return metrics;
    }
}
```

### 6.3 N-Gram Bloom Filter

**什么是 N-Gram**：

N-Gram 是将字符串分割为长度为 N 的子串序列的技术：

```
字符串: "hello"
2-gram: {"he", "el", "ll", "lo"}
3-gram: {"hel", "ell", "llo"}
```

**在 SkipIndex 中的作用**：

对于 `LIKE '%abc%'` 查询（substring search），我们可以：

1. 将查询字符串 "abc" 分割为 N-gram
2. 检查每个 N-gram 是否在 Chunk 的 N-gram Bloom Filter 中
3. 若任意一个 N-gram 不在 BF 中，则整个 Chunk 可以跳过

这是一种**必要不充分条件**的应用：

- BF 中所有 N-gram 都存在 → 该 Chunk 可能包含目标字符串（不能跳过）
- BF 中有 N-gram 不存在 → 该 Chunk 一定不含目标字符串（可以跳过）

**ICU Unicode 支持**：

`utils.h` 中使用 ICU（International Components for Unicode）库进行 N-gram 提取：

```cpp
// ExtractNgrams 函数（简化）
std::vector<std::string> ExtractNgrams(std::string_view text, int min_n, int max_n) {
    // 使用 ICU 将文本转换为 Unicode code points
    // 以 code point 为单位进行 N-gram 分割
    // 保证多语言字符（中文、日文等）的正确处理
}
```

这很重要：如果直接按字节分割，一个中文字符可能被截断。使用 ICU 按 Unicode code point 分割，保证了多语言正确性。

### 6.4 类型系统设计的演变

在最终设计之前，我考虑过以下方案：

**方案A（被放弃）：模板 + 组合**

```cpp
template <typename T>
class StatsHolder {
    // MinMax<T>
    // Set<T>
    // BloomFilter<T>
    // 通过组合实现"一种数据类型持有多种统计信息"
};
```

放弃原因：

- 模板实例化组合爆炸（每种统计信息 × 每种数据类型 = 大量模板实例）
- 统计信息的组合逻辑复杂，维护困难
- 过于泛化，超出了当前需求

**方案B（最终选择）：继承**

```cpp
class IntFieldChunkMetrics : public FieldChunkMetrics {
    // 直接包含该类型可能用到的所有统计信息
    T min_, max_;
    BloomFilterPtr bloom_filter_;  // 可选
};

class StringFieldChunkMetrics : public FieldChunkMetrics {
    std::string min_, max_;
    BloomFilterPtr bloom_filter_;
    BloomFilterPtr ngram_bloom_filter_;
};
```

选择理由：

- **YAGNI 原则**（You Aren't Gonna Need It）：当前数据类型种类少，不需要过度泛化
- **简单直接**：每个子类直接表达"这种类型需要这些统计信息"
- **可读性好**：不需要理解复杂的模板机制就能理解代码
- **开销小**：继承的虚函数开销在查询时可以接受

---

## 第七章：方案演变与设计争议

### 7.1 初始方案（2025年6月）

**方案描述**：SkipIndex 作为独立文件存储，与 bm25stats、textstats 同级。

```
Segment 元数据：
  - insert binlog:  /segments/{id}/insert/
  - stats binlog:   /segments/{id}/stats/     （主键统计）
  - bm25 binlog:    /segments/{id}/bm25stats/ （BM25 统计）
  - skip binlog:    /segments/{id}/skipstats/ （SkipIndex，新增）
```

**触发时机**：Segment Sealed 后，DataCoord 下发 BuildSkipIndex 任务给 DataNode，DataNode 读取数据，计算每个 Chunk 的统计信息，写入独立文件。

**优点**：

- 与现有模式（bm25stats）一致，架构清晰
- 存储格式无关（不依赖 Parquet）
- 元数据管理路径清晰（etcd 中有记录）

**缺点**（当时认为）：

- 需要单独读取数据来计算统计信息（额外 I/O）
- 需要新增任务调度链路（DataCoord → DataNode）

### 7.2 导师方案（2025年7月）

**方案描述**：将 SkipIndex 存储到 Parquet 文件的 footer 中，与数据共存。

```
Parquet 文件结构：
  RowGroup 0
  RowGroup 1
  ...
  RowGroup N
  Footer:
    - Parquet 原生 Statistics（每个列每个 RowGroup 的 min/max）
    - Key-Value Metadata:
        "skip_index_field_1": <serialized SkipIndex for field 1>
        "skip_index_field_2": <serialized SkipIndex for field 2>
```

**触发时机**：在 Sort Task 中，写入 Parquet 数据的同时，同步构建并写入 SkipIndex（就像 milvus-storage 写入 RowGroupMetadata 一样）。

**优点**（导师认为）：

- 数据与统计信息共置（co-located），读取时一次 I/O 即可获取
- 不需要额外的任务调度
- 复用现有的写入流水线

**挑战**：

- milvus-storage 与 Milvus 解耦，需要设计通用接口
- 构建时机从"后台任务"改为"写入时同步"，逻辑更复杂

### 7.3 Issue #44584 的挑战（2025年10月）

**讨论背景**：我在提交 PR #44581 时，同时提交了 Issue #44584 描述整体设计。Milvus 官方人员（Reviewer）在 Issue 中提出了质疑。

**质疑的核心**：

> Milvus 计划支持新的存储格式（Vortex、Lance 等），将 SkipIndex 存储在 Parquet footer 中意味着 SkipIndex 的存储与 Parquet 格式强绑定。当切换到新格式时，SkipIndex 的读写逻辑需要全部重写。

这是一个非常合理的架构问题——**扩展性**。

**最终决定**：回到最初的独立文件方案。具体来说：

- SkipIndex 序列化为独立的二进制文件
- 通过 Milvus 的元数据系统（etcd）记录 SkipIndex 文件的路径
- 读取时通过文件路径加载，与存储格式无关

**影响**：这意味着整个写入和读取链路需要重新设计，而此时已是十月中旬，距离项目截止还剩两周。

### 7.4 最终完成状态

| 功能模块                                   | 完成状态                 | 备注                         |
| ------------------------------------------ | ------------------------ | ---------------------------- |
| SkipIndex 统计信息扩展（BF、NgramBF、Set） | ✅ 基本完成               | 在 PR #44581 中              |
| 表达式层集成（更多 OpType 支持）           | ✅ 完成                   | TermExpr、BinaryArithExpr 等 |
| milvus-storage Metadata 接口设计           | ✅ 完成（但后来方案变更） | 接口设计本身是有价值的       |
| SkipIndex 构建触发链路                     | ⚠️ 未完全打通             | 需要重新按独立文件方案实现   |
| SkipIndex 持久化（写入）                   | ⚠️ 未完全打通             | 独立文件方案需要时间         |
| SkipIndex 预加载（读取）                   | ⚠️ 未完全打通             | 独立文件方案需要时间         |

---

## 第八章：技术知识点深度解析

### 8.1 C++ 模板元编程

#### 8.1.1 SFINAE（Substitution Failure Is Not An Error）

SkipIndex 中大量使用 SFINAE 技术，通过 `std::enable_if_t` 实现类型约束：

```cpp
// 版本1：T 满足约束时启用
template <typename T>
std::enable_if_t<SkipIndex::IsAllowedType<T>::value, bool>
CanSkipUnaryRange(FieldId field_id, int64_t chunk_id, OpType op_type,
                  const T& val) const {
    // 实际逻辑
}

// 版本2：T 不满足约束时启用（fallback）
template <typename T>
std::enable_if_t<!SkipIndex::IsAllowedType<T>::value, bool>
CanSkipUnaryRange(FieldId field_id, int64_t chunk_id, OpType op_type,
                  const T& val) const {
    return false;  // 不支持的类型，保守策略
}
```

`std::enable_if_t<Cond, ReturnType>` 的工作原理：

- 若 `Cond == true`：`enable_if_t` 等于 `ReturnType`，函数签名合法，参与重载解析
- 若 `Cond == false`：`enable_if_t` 不存在，函数签名非法，**但这不是错误**（SFINAE），只是从重载集合中排除

这样保证了：调用 `CanSkipUnaryRange<Json>()` 时，只有版本2可以实例化，不会编译错误。

#### 8.1.2 std::variant 与 std::visit

`Metrics` 使用 `std::variant` 实现类型安全的联合体：

```cpp
using Metrics = std::variant<bool, int8_t, int16_t, int32_t, int64_t,
                             float, double, std::string, std::string_view>;

// 使用 std::get_if 安全访问
void ProcessMetrics(const Metrics& m) {
    if (auto* val = std::get_if<int64_t>(&m)) {
        // 是 int64_t 类型
        std::cout << "int64: " << *val << std::endl;
    } else if (auto* val = std::get_if<std::string_view>(&m)) {
        // 是 string_view 类型
        std::cout << "string: " << *val << std::endl;
    }
}

// 或者使用 std::visit（更通用）
std::visit([](const auto& val) {
    // val 的类型在编译时确定
    std::cout << val << std::endl;
}, metrics);
```

相比 C union 的优势：

- `union` 不追踪当前存储的类型，访问错误类型是 UB（Undefined Behavior）
- `variant` 追踪当前类型，访问错误类型会抛出 `std::bad_variant_access`
- `variant` 在存储对象时调用构造函数，在析构时调用析构函数（RAII 友好）

#### 8.1.3 std::conditional_t

```cpp
// 根据编译时条件选择类型
template <typename T>
using HighPrecisionType =
    std::conditional_t<std::is_integral_v<T> && !std::is_same_v<bool, T>,
                       int64_t,  // 满足条件时的类型
                       T>;       // 不满足条件时的类型
```

`std::conditional_t<B, T, F>` 等效于：若 B 为真，则为 T，否则为 F。是编译时的三目运算符。

#### 8.1.4 if constexpr（C++17）

```cpp
auto check_and_skip = [&](HighPrecisionType<T> new_value_hp, OpType new_op_type) {
    if constexpr (std::is_integral_v<T>) {  // 编译时 if
        // 只有整数类型才编译此块
        if (new_value_hp > std::numeric_limits<T>::max() ||
            new_value_hp < std::numeric_limits<T>::min()) {
            return false;  // 溢出检测
        }
    }
    // 浮点类型不进入上面的块
    return CanSkipUnaryRange<T>(...);
};
```

`if constexpr` 与普通 `if` 的区别：

- 普通 `if`：两个分支都会编译，只是运行时不执行
- `if constexpr`：条件为 false 的分支**不参与编译**，避免编译错误

这里如果用普通 `if`，`std::numeric_limits<T>::max()` 对浮点类型也会实例化，可能引发类型不匹配的问题。

### 8.2 Bloom Filter 原理

Bloom Filter 是一种概率性数据结构，用于快速判断一个元素**是否可能存在**于集合中：

**工作原理**：

1. 初始化：一个大小为 m 的位数组，全部置 0
2. 插入元素 x：计算 k 个哈希值 h1(x), h2(x), ..., hk(x)，将位数组中对应位置置 1
3. 查询元素 x：计算 k 个哈希值，若所有对应位都为 1，则**可能存在**；若有任何一位为 0，则**一定不存在**

**特性**：

- 无假阴性（No False Negative）：若元素在集合中，BF 一定返回"存在"
- 有假阳性（False Positive）：若 BF 返回"可能存在"，元素不一定真的在集合中

这对 SkipIndex 来说是**完美匹配**：

- 无假阴性保证不会漏掉匹配的数据（正确性）
- 有假阳性只是不能跳过某些 Chunk（不影响结果正确性，只影响性能）

**大小计算**：

对于 n 个元素，误判率 p，最优 Bloom Filter 参数为：

```
m（位数）= -n * ln(p) / (ln 2)^2
k（哈希函数数）= m/n * ln 2
```

以 n = 1,000,000，p = 0.01 为例：

```
m = -1,000,000 * ln(0.01) / (ln 2)^2
  ≈ 1,000,000 * 4.605 / 0.480
  ≈ 9,585,058 位 ≈ 1.14 MB
k = 9,585,058 / 1,000,000 * ln 2 ≈ 6.6 ≈ 7
```

### 8.3 设计模式详解

#### 8.3.1 策略模式（Strategy Pattern）

`FieldChunkMetrics` 的类层次结构是经典的策略模式：

```
FieldChunkMetrics（接口/策略）
    ├── NoneFieldChunkMetrics     （无跳过策略）
    ├── BooleanFieldChunkMetrics  （布尔跳过策略）
    ├── FloatFieldChunkMetrics<T> （浮点跳过策略）
    ├── IntFieldChunkMetrics<T>   （整数跳过策略）
    └── StringFieldChunkMetrics   （字符串跳过策略）
```

`SkipIndex` 持有 `FieldChunkMetrics*`，在运行时根据数据类型选择具体策略：

- 调用 `CanSkipUnaryRange()` 时，不关心是整数还是字符串，统一接口
- 具体的跳过逻辑封装在各个子类中，互不干扰

#### 8.3.2 建造者模式（Builder Pattern）

`SkipIndexStatsBuilder` 和 `MetadataBuilder` 都是建造者模式的应用：

```cpp
// MetadataBuilder 的使用方式
MetadataBuilder* builder = new ChunkSkipIndexBuilder();

// 逐步构建（每个 RowGroup 调用一次）
builder->Append(batch_0);  // RowGroup 0 的数据
builder->Append(batch_1);  // RowGroup 1 的数据
builder->Append(batch_2);  // RowGroup 2 的数据

// 最终产出（序列化所有 RowGroup 的统计信息）
std::string result = builder->Finish();
```

建造者模式将"对象的构建过程"与"对象的表示"分离：

- 构建过程：`Append()` 逐步积累状态
- 表示：`Finish()` 返回最终序列化结果
- 好处：支持增量构建，可以在数据流式到达时逐步处理

#### 8.3.3 缓存模式（Cache-Aside）

SkipIndex 的 `CacheSlot + PinWrapper + Translator` 组合实现了 Cache-Aside 模式：

```
查询操作：
  1. 尝试从 CacheSlot 获取 FieldChunkMetrics（命中 → 返回）
  2. 若未命中，通过 Translator 加载数据（计算 → 存入 Cache → 返回）
  3. PinWrapper RAII 确保使用期间不被驱逐
```

这是 Milvus 通用缓存层（Caching Layer）的标准使用模式，SkipIndex 只是其中的一个用户。

#### 8.3.4 RAII（Resource Acquisition Is Initialization）

`PinWrapper` 是 RAII 的典型应用：

```cpp
// 伪代码
class PinWrapper {
    CacheEntry* entry_;
 public:
    PinWrapper(CacheEntry* e) : entry_(e) {
        entry_->pin_count++;  // 构造时增加引用
    }
    ~PinWrapper() {
        entry_->pin_count--;  // 析构时减少引用（自动释放）
    }
    const FieldChunkMetrics* get() const { return entry_->data; }
};

// 使用方式：自动管理生命周期
{
    auto pw = GetFieldChunkMetrics(field_id, chunk_id);
    // pw 存在期间，对应的 Cache Cell 不会被驱逐
    auto result = pw.get()->CanSkipUnaryRange(...);
}  // pw 析构，引用计数 --，Cell 可被驱逐
```

`defer C.free(unsafe.Pointer(...))` 在 Go CGO 层也是类似的 RAII 思想——资源在获取时注册释放逻辑，函数返回时自动清理。

#### 8.3.5 类型擦除（Type Erasure）

在最初的设计中，为了让 `ParquetFileWriter` 持有不同类型的 `MetadataBuilder`，使用了类型擦除：

```cpp
// 问题：这两个 Builder 类型不同，无法放入同一容器
ChunkSkipIndexBuilder skip_builder;     // 对 SkipIndex 的 MetadataBuilder
SomeOtherMetadataBuilder other_builder;

// 解决：MetadataBuilder 基类 + 指针多态（虚函数本质上就是类型擦除）
std::vector<std::unique_ptr<MetadataBuilder>> builders;
builders.push_back(std::make_unique<ChunkSkipIndexBuilder>());
builders.push_back(std::make_unique<SomeOtherMetadataBuilder>());

// 统一调用，无需知道具体类型
for (auto& builder : builders) {
    builder->Append(batch);
}
```

### 8.4 Parquet 文件格式

Parquet 是一种面向列的存储格式，广泛用于大数据生态：

**文件结构**：

```
┌──────────────────────────────────┐
│           Magic Number            │  4 bytes "PAR1"
├──────────────────────────────────┤
│           Row Group 0             │
│  ┌────────────────────────────┐  │
│  │  Column Chunk 0 (col A)    │  │
│  │  Column Chunk 1 (col B)    │  │
│  └────────────────────────────┘  │
├──────────────────────────────────┤
│           Row Group 1             │
│  ┌────────────────────────────┐  │
│  │  Column Chunk 0 (col A)    │  │
│  │  Column Chunk 1 (col B)    │  │
│  └────────────────────────────┘  │
├──────────────────────────────────┤
│             Footer                │
│  ┌────────────────────────────┐  │
│  │  FileMetaData (Thrift)     │  │
│  │    - schema                │  │
│  │    - row_groups info       │  │
│  │    - key_value_metadata    │  │  ← SkipIndex 存储在这里
│  └────────────────────────────┘  │
│  Footer Length (4 bytes)          │
│  Magic Number (4 bytes) "PAR1"   │
└──────────────────────────────────┘
```

**key_value_metadata**：Parquet footer 的 `FileMetaData` 中有一个 `key_value_metadata` 字段，可以存储任意的 key-value 字符串对。这就是我们选择存储 SkipIndex 的位置（导师方案）。

**Parquet 原生 Statistics**：每个 Column Chunk 有内置的 Statistics，包括：

- `min_value`, `max_value`：列在该 RowGroup 中的最小/最大值
- `null_count`：空值数量

这实际上就是原始版本的 SkipIndex（MinMax），但只支持原生类型，不支持 Bloom Filter 等。

### 8.5 CGO 与 Arrow C Data Interface

#### 8.5.1 CGO 内存管理

Go 和 C 之间的内存管理是 CGO 编程的核心挑战：

```go
// C 字符串：C 分配的内存必须用 C.free 释放
cStr := C.CString("hello")
defer C.free(unsafe.Pointer(cStr))  // 保证释放

// Go 字符串：只需普通 Go GC 管理
goStr := "hello"  // Go 管理，不需要手动释放

// C 动态数组
cArr := C.malloc(C.size_t(n) * C.size_t(unsafe.Sizeof(C.int(0))))
defer C.free(cArr)  // 必须手动释放
```

**关键规则**：

- `C.CString()` 创建的 C 字符串由 C 分配，必须用 `C.free()` 释放
- `defer C.free()` 保证即使函数中途 panic 也能释放内存
- Go 的 GC 不追踪 C 内存，因此**永远不要把 Go 指针存到 C 结构体中**（Go GC 可能移动 Go 对象）

#### 8.5.2 Arrow C Data Interface

Arrow C Data Interface（ABI）定义了在不同语言/库之间传递 Arrow 数据的标准 C 结构体：

```c
struct ArrowSchema {
    const char* format;
    const char* name;
    const char* metadata;
    int64_t flags;
    int64_t n_children;
    struct ArrowSchema** children;
    struct ArrowSchema* dictionary;
    void (*release)(struct ArrowSchema*);  // 释放回调
    void* private_data;
};

struct ArrowArray {
    int64_t length;
    int64_t null_count;
    int64_t offset;
    int64_t n_buffers;
    int64_t n_children;
    const void** buffers;
    struct ArrowArray** children;
    struct ArrowArray* dictionary;
    void (*release)(struct ArrowArray*);  // 释放回调
    void* private_data;
};
```

**零拷贝原理**：`buffers` 指针直接指向数据内存，不进行任何拷贝。消费者（C++层）通过指针读取数据后，调用 `release` 函数通知生产者（Go层）可以释放内存。

### 8.6 数据库 Zone Map / Skip Index 理论

SkipIndex 本质上是数据库领域中"Zone Map"（也叫 Small Materialized Aggregates）的一种实现。

**Zone Map 的基本思想**：

- 将数据按物理存储顺序划分为 Zone（区域，对应 Milvus 的 Chunk/RowGroup）
- 为每个 Zone 存储汇总统计信息（min, max, count, null_count 等）
- 查询时用这些统计信息快速判断 Zone 是否与查询条件相交
- 若不相交，跳过整个 Zone（无需读取实际数据）

**与 LSM-Tree 的类比**：

Milvus 的存储结构与 LSM-Tree 有高度相似性：

| LSM-Tree           | Milvus                                   |
| ------------------ | ---------------------------------------- |
| MemTable           | Growing Segment（内存中，无序）          |
| Immutable MemTable | Sealed Segment（不可写，等待持久化）     |
| L0 SSTable         | Flushed Segment（已持久化，可能无序）    |
| L1+ SSTable        | Sorted/Indexed Segment（有序，已建索引） |
| Compaction         | Compaction（Merge/Sort Compaction）      |
| SSTable Block      | RowGroup/Chunk                           |
| Block Index        | SkipIndex                                |

在 RocksDB（典型 LSM 实现）中，SSTable 的 Block Index 存储每个 Block 的最后一个 key，用于快速定位；Milvus 的 SkipIndex 存储每个 Chunk 的统计信息，用于快速跳过。

---

## 思考

回顾整个OSPP2025 Milvus的任务，从中学习了到很多东西——C++新语法、大型项目中的接口扩展性设计、大型项目代码阅读能力、AI Coding辅助等等。

对于最终项目未能开发完成，也是多方面原因。
- 首先是初次接触大型项目是一头雾水，只做过Bustub、TinyKV这种课程项目，对于Milvus这种大型项目光是阅读项目理解需求就花费了很多时间。
- 其次是对于开源社区协作开发开始时也并不了解，在提交PR时才意识到需要先开Issue，导致后续方案变动时已没有时间完成剩余任务。
- 最后是项目开发过程中缺少沟通交流，导致项目开发陷入自我思考的圈子，对于方案选择思考过度，想着一开始就选择最好最合适的方案，导致开发进展缓慢。

通过这次 OSPP2025 的活动，大型项目的开发最重要的是沟通交流、清晰的方案、协作开发流程以及对需求的精确认识，大部分的时间可能就花在了代码阅读、确立方案以及最后的测试上。

---

## 附录：关键代码位置索引

| 模块             | 文件路径                                                     | 核心内容                                                 |
| ---------------- | ------------------------------------------------------------ | -------------------------------------------------------- |
| SkipIndex 主接口 | `internal/core/src/index/SkipIndex.h`                        | SkipIndex 类，IsAllowedType，HighPrecisionType           |
| SkipIndex 实现   | `internal/core/src/index/SkipIndex.cpp`                      | GetFieldChunkMetrics, LoadSkip                           |
| 统计信息定义     | `internal/core/src/index/skipindex_stats/SkipIndexStats.h`   | Metrics variant, FieldChunkMetrics 层次, RangeShouldSkip |
| 统计信息构建     | `internal/core/src/index/skipindex_stats/SkipIndexStats.cpp` | SkipIndexStatsBuilder 实现                               |
| 类型辅助工具     | `internal/core/src/index/skipindex_stats/utils.h`            | SupportsSkipIndex, ExtractNgrams                         |
| TermExpr 集成    | `internal/core/src/exec/expression/TermExpr.cpp`             | IN 查询 SkipIndex                                        |
| 算术范围查询     | `internal/core/src/exec/expression/BinaryArithOpEvalRangeExpr.cpp` | 算术变换 SkipIndex                                       |
| Go 写入层        | `internal/storage/record_writer.go`                          | packedRecordWriter                                       |
| CGO 桥接         | `internal/storagev2/packed/packed_writer.go`                 | NewPackedWriter, WriteRecordBatch                        |
| Metadata 接口    | `milvus-storage/cpp/include/milvus-storage/common/metadata.h` | Metadata, MetadataBuilder, PackedFileMetadata            |

---

## 附录：重要术语对照

| 术语                   | 含义                                                         |
| ---------------------- | ------------------------------------------------------------ |
| Segment                | Milvus 中数据的基本存储单元，Growing（内存）或 Sealed（持久化） |
| Chunk                  | Sealed Segment 中数据的基本处理单元，对应一个 Parquet RowGroup |
| RowGroup               | Parquet 文件中的逻辑行分组，包含该组所有列的数据             |
| Column Group           | milvus-storage 中的列分组，多个字段共享一个 Parquet 文件     |
| SkipIndex              | Chunk 级统计信息，用于快速跳过不匹配的 Chunk                 |
| Zone Map               | 数据库领域 SkipIndex 的通用名称                              |
| DataCoord              | 管理数据 Segment 生命周期的协调者组件                        |
| DataNode               | 执行数据写入、排序、压缩的工作节点                           |
| QueryNode              | 加载 Segment 并执行查询的工作节点                            |
| Binlog                 | Milvus 中数据文件的统称（insert binlog、stats binlog 等）    |
| SFINAE                 | C++ 模板替换失败不是错误，用于编译时类型选择                 |
| CRTP                   | 奇异递归模板模式，实现编译时多态                             |
| RAII                   | 资源获取即初始化，通过对象生命周期管理资源                   |
| Arrow C Data Interface | Apache Arrow 定义的跨语言零拷贝数据传输 ABI                  |

---

## 附录：学习资源

- [Milvus 官方文档](https://milvus.io/docs)
- [Apache Parquet 格式规范](https://parquet.apache.org/docs/file-format/)
- [Apache Arrow C Data Interface](https://arrow.apache.org/docs/format/CDataInterface.html)
- [Bloom Filter 原理（维基百科）](https://en.wikipedia.org/wiki/Bloom_filter)
- [Milvus GitHub 仓库](https://github.com/milvus-io/milvus)
- [milvus-storage GitHub 仓库](https://github.com/milvus-io/milvus-storage)
- [相关 PR #44581](https://github.com/milvus-io/milvus/pull/44581)
- [相关 Issue #44584](https://github.com/milvus-io/milvus/issues/44584)
- C++ 参考：Scott Meyers《Effective Modern C++》（涵盖模板、移动语义、std::variant 等）
- Go CGO 文档：[https://pkg.go.dev/cmd/cgo](https://pkg.go.dev/cmd/cgo)

