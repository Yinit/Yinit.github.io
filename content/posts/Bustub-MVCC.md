---
title: "Bustub MVCC"
date: 2025-02-10T18:58:22+08:00
lastMod: 2026-04-05T11:53:25Z
draft: false
author: ["tkk"]

categories: ["数据库"]

tags: ["CMU15445", "Bustub", "MVCC"]

keywords: ["CMU15445", "Bustub", "MVCC", "并发控制"]

description: "CMU 15445 23fall Project4 实现与 MVCC 深度复习，涵盖版本链、时间戳、并发控制等工程细节"
summary: "从 Project4 实现出发，系统复习 BusTub MVCC 设计原理和工程细节"
weight:
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

## bustub 项目历程

从去年十月初到今年一月中旬，历时一个学期，在实验室的安排下着手开始 CMU 15445 23fall 的数据库知识的学习和 Project 的实现，终于在过年前完成所有的 Project。正好寒假在家有时间，就着手建了一个个人博客，写一下我在实现 Project 过程中遇到的问题和收获。关于课程视频，我时间有限，就只看了前几个视频，如果有同学感兴趣，还是推荐看一下，理论知识的学习还是很重要的。

## 前言

老规矩，先看一下最终排名，截止 2025/02/10，总分第 7，单榜第 1，打开 LeaderBoard 第一个就是我了（嘿嘿）

![Project4-LeaderBoard](/images/bustub-mvcc/23fall-Project4-LeaderBoard.png)

个人认为 project4 完全做完应该是 4 个 project 中最难的，花的时间也是最多的。不过开始写的时候刚好考试比较多，没多少时间写，大概花了几天时间把前几个 task 完成了，最后在考试完后回家写了几天才写完。虽然可能没写太久，但是感觉花费的精力挺多的。

---

## 一、MVCC 理论基础

### 1.1 什么是 MVCC？

**MVCC（Multi-Version Concurrency Control，多版本并发控制）** 是数据库系统中最主流的并发控制机制之一。其核心思想是：**每次写操作不覆盖旧数据，而是创建数据的一个新版本；读操作通过时间戳（或版本号）找到它"应该"看到的历史版本**。

这样的设计带来了关键特性：**读操作不会阻塞写操作，写操作不会阻塞读操作**，读写之间完全不争锁，从根本上解决了传统 2PL（Two-Phase Locking）中读写互斥的性能瓶颈。

### 1.2 MVCC 与 2PL 的对比

| 维度       | 2PL（两阶段锁）                 | MVCC                                  |
| ---------- | ------------------------------- | ------------------------------------- |
| 读写关系   | 读写互斥（S 锁 / X 锁）         | 读写不互斥                            |
| 写写关系   | 写写阻塞等待                    | 写写冲突 → abort（First-Writer-Wins） |
| 死锁风险   | 存在（需要死锁检测/预防）       | 无死锁（通过 abort 解决）             |
| 历史版本   | 不保留                          | 保留多版本（UndoLog 链）              |
| 实现复杂度 | 锁管理复杂                      | 版本链管理、GC 复杂                   |
| 典型系统   | InnoDB（历史版本同时也用 MVCC） | PostgreSQL, MySQL InnoDB, BusTub      |
| 读性能     | 受写锁阻塞                      | 高（始终读快照）                      |
| 写吞吐     | 并行度受限                      | 高并发（短事务）                      |

> **补充**：现代工业级数据库如 MySQL InnoDB 同时使用 2PL 和 MVCC：2PL 保证当前读（`SELECT FOR UPDATE`）的正确性，MVCC 提升普通读（快照读）的性能。BusTub 的实现更接近纯 MVCC（读不加锁）。

### 1.3 Snapshot Isolation（快照隔离）

**快照隔离（SI）** 是 MVCC 提供的最常见隔离级别，其核心保证：

> 每个事务在 **Begin 时刻**看到一个数据库的一致性快照，此后该事务只能看到这个快照中的数据，以及**自身**写入的数据。

**SI 的正式定义（三个约束）：**

1. **Snapshot Read**：事务 T 读到的是所有在 T.read_ts 之前已提交的数据版本
2. **Consistent Snapshot**：所有可见数据在同一时间点上是一致的（不会看到一半的事务）
3. **First-Writer-Wins**：若两个并发事务 T1, T2 都想写同一行，先获得写权的事务赢，后来者 abort

**SI 可以解决的问题：**

- 脏读（Dirty Read）：✅ 不会读到未提交的数据
- 不可重复读（Non-Repeatable Read）：✅ 读的是固定快照
- 大部分幻读（Phantom Read）：✅ 快照是固定的

**SI 无法解决的问题（异常）：**

- **写偏序（Write Skew）**：经典例子是两个事务都读了同一个表并各自更新了不同行，违反了整体的完整性约束，但因为没有直接的写写冲突，两者都能提交。

  ```
  初始：doctor_on_call = {A: true, B: true}，要求至少一人值班
  T1 读取：发现 A=true, B=true，将 A 设为 false（A 不值班了）
  T2 读取：发现 A=true, B=true，将 B 设为 false（B 不值班了）
  结果：A=false, B=false → 无人值班，违反约束，但 SI 下两者都提交成功
  ```

### 1.4 Serializable Snapshot Isolation（SSI）

**SSI** 是在 SI 基础上，通过在提交时做额外的冲突检测，将隔离级别提升到**可串行化（Serializability）**的方案。

**SSI 的核心思想（基于依赖分析）：**

标准的 SSI 分析事务间的三种依赖关系并检测危险结构（danger triangle）。BusTub 的实现采用的是一种**简化版 SSI**：在提交时，检查本事务**写的行**是否被其他在本事务开始后提交的事务修改过，且这些行满足本事务曾经扫描过的**谓词（scan predicate）**。

即：检测 **"幻读（Phantom）"** 的发生，防止一类经典的不可串行化现象。

### 1.5 First-Writer-Wins 原则

BusTub 实现的是 **First-Writer-Wins（先写者获胜）** 策略：

- 若事务 T1 已经修改了某行（tuple_ts 变为 T1 的临时 ID），事务 T2 也想修改同一行
- T2 检测到 `tuple_ts > T2.read_ts` 且 `tuple_ts ≠ T2.GetTransactionId()`
- T2 立即 **abort（设置 TAINTED 状态，抛出异常）**
- T1 继续正常执行

这与 2PL 的"后来者等待"不同，MVCC 不等待，直接 abort，适合短事务密集的 OLTP 场景。

---

## 二、BusTub MVCC 整体架构

### 2.1 版本链设计

BusTub 的 MVCC 采用 **Undo Log 链式存储**（也叫 **Delta Storage / Diff-based Storage**）。

```
TableHeap（最新版本，通常是正在进行中的事务或最后一次提交的结果）
    ↓  (通过 TransactionManager 中的 version_info_ 映射)
VersionUndoLink { prev_: UndoLink, in_progress_: bool }
    ↓  prev_
UndoLog₁ { ts_, is_deleted_, modified_fields_, tuple_, prev_version_ }
    ↓  prev_version_
UndoLog₂
    ↓
UndoLog₃ → INVALID (链尾)
```

**关键设计决策：**

- **最新版本在 TableHeap 中**（不在 undo log 中），读当前版本最快
- **undo log 存储的是"从当前版本回退一步的差异"**，而非完整快照
- **版本链越新越靠前**，沿链向后找到越旧的版本

对比两种常见的版本存储方式：

| 存储方式                   | 最新版本位置     | 历史版本位置      | 代表系统                 |
| -------------------------- | ---------------- | ----------------- | ------------------------ |
| Undo Log（BusTub, InnoDB） | 主表（in-place） | 回滚段（差量）    | MySQL, BusTub            |
| Append-Only（MVTO）        | 版本链最新节点   | 主表/版本链旧节点 | PostgreSQL（Heap-based） |

### 2.2 核心数据结构

**TupleMeta（存储在 TablePage 中，每条 tuple 配套）**

```cpp
// src/include/storage/table/tuple.h
struct TupleMeta {
  timestamp_t ts_;      // 双重含义：
                        //   < TXN_START_ID → 已提交事务的 commit_ts
                        //   >= TXN_START_ID → 正在进行事务的临时 txn_id
  bool is_deleted_;     // 软删除标志（逻辑删除）
};
static constexpr size_t TUPLE_META_SIZE = 16;  // 固定 16 字节
```

**UndoLink（版本链指针）**

```cpp
// src/include/concurrency/transaction.h
struct UndoLink {
  txn_id_t prev_txn_;       // 指向拥有该 UndoLog 的事务 ID
  int32_t prev_log_idx_;    // 该 UndoLog 在事务 undo_logs_ 数组中的下标

  auto IsValid() const -> bool { ... }  // prev_txn_ != INVALID_TXN_ID
};
```

**UndoLog（差量记录）**

```cpp
struct UndoLog {
  bool is_deleted_;                      // 这一步操作是否是删除
  std::vector<bool> modified_fields_;    // 位图：哪些列在这一步被修改了
  Tuple tuple_;                          // 只存 modified_fields_ 为 true 的列（稀疏存储）
  timestamp_t ts_;                       // 这条 UndoLog 对应的事务提交时间
  UndoLink prev_version_;               // 指向更旧版本的 UndoLog
};
```

> **稀疏存储的意义：** 假设一张 100 列的宽表，每次只更新 2 列，使用全量存储每次需要保存 100 列数据；使用稀疏存储（`modified_fields_` 位图 + 只存变化列）只需要存 2 列数据 + 100 位的位图，空间节省 ~98%。

**VersionUndoLink（版本链头节点的包装）**

```cpp
// src/include/concurrency/transaction_manager.h
struct VersionUndoLink {
  UndoLink prev_;           // 指向最新 UndoLog 的链接
  bool in_progress_;        // 当前是否有事务正在修改此 tuple

  static auto FromOptionalUndoLink(std::optional<UndoLink> undo_link) -> VersionUndoLink;
};
```

`in_progress_` 的作用是一个"轻量级写锁"标记，防止并发写同一 tuple（详见 Task #4 并发实现）。

### 2.3 版本链存储的组织方式

```cpp
// src/include/concurrency/transaction_manager.h
struct PageVersionInfo {
  std::shared_mutex mutex_;   // 保护此 Page 内所有 slot 的版本链头
  std::unordered_map<slot_offset_t, VersionUndoLink> prev_version_;
};

// TransactionManager 成员
std::unordered_map<page_id_t, std::shared_ptr<PageVersionInfo>> version_info_;
```

**设计思路：**

- 不是全局一个大锁，而是**以 Page 为粒度**组织版本链头
- 每个 `PageVersionInfo` 有独立的 `shared_mutex_`，并发访问同一 Page 的不同 slot 时需要竞争同一锁（粒度介于全局锁和 slot 级锁之间）
- 通过 `RID = (page_id, slot_offset)` 定位到对应的 `VersionUndoLink`

### 2.4 TXN_START_ID 常量

```cpp
// src/include/common/config.h
static constexpr txn_id_t TXN_START_ID = 1LL << 62;  // 极大的数
```

`ts_` 字段复用了 `timestamp_t` 和 `txn_id_t` 两种语义：

- **`ts_ < TXN_START_ID`**：已提交的事务时间戳（commit_ts），具体数值从 0 单调递增
- **`ts_ >= TXN_START_ID`**：正在进行的事务的临时 ID（`txn_id = TXN_START_ID + 序号`）

这样只需要一个 `int64_t` 字段，就能区分"已提交"和"未提交"两种状态，比用额外的 bool 字段更节省空间，也便于比较。

---

## 三、Task #1 — Timestamps 与 Watermark

### 3.1 任务概述

task1 主要是熟悉这部分相关代码，写的部分比较简单，总共实现两个部分：

1. 事务管理器中创建事务，赋予时间戳，将其放入 watermark 中
2. watermark 管理所有正在运行的事务，保存正在运行的事务的最小时间戳，创建事务时向其中添加事务时间戳，提交事务时从中删除

### 3.2 时间戳分配策略

BusTub 使用**逻辑时间戳（Logical Timestamp）**，而非物理时钟：

```
事务 Begin 时：read_ts = last_commit_ts（当前最新提交的时间戳）
事务 Commit 时：commit_ts = ++last_commit_ts（原子递增获取新时间戳）
```

每个事务在开始时"拍下"一个快照时间戳（`read_ts`），此后只看 `commit_ts ≤ read_ts` 的已提交数据。Commit 时递增全局计数器获得唯一的提交时间戳。

```cpp
auto TransactionManager::Begin(IsolationLevel isolation_level) -> Transaction * {
  std::unique_lock<std::shared_mutex> l(txn_map_mutex_);
  auto txn_id = next_txn_id_++;
  auto txn = std::make_unique<Transaction>(txn_id, isolation_level);
  auto *txn_ref = txn.get();
  txn_map_.insert(std::make_pair(txn_id, std::move(txn)));

  // 关键：read_ts = 当前最新提交时间戳（原子读取）
  txn_ref->read_ts_ = last_commit_ts_.load();

  // 注册到 Watermark 追踪器
  running_txns_.AddTxn(txn_ref->read_ts_);
  return txn_ref;
}
```

### 3.3 Watermark 的定义与作用

**Watermark（水印）** 是所有活跃事务（正在运行的事务）的 **最小 read_ts**。

```
活跃事务：T1(read_ts=5), T2(read_ts=7), T3(read_ts=9)
Watermark = min(5, 7, 9) = 5
```

**Watermark 的核心用途：垃圾回收（GC）**

`commit_ts ≤ Watermark` 的版本对所有活跃事务都不再需要（因为所有事务都能看到比它更新的版本），可以安全回收。

### 3.4 Watermark O(1) 均摊实现思路

关于这一部分的 O(1) 实现方法，利用事务管理器中的 `last_commit_ts_`，它赋予给所有创建的事务，并且是单调递增，在 commit 时 +1。这样便可以确定 watermark 中时间戳在一定的范围内，时间范围最大为 `last_commit_ts_`，最小为正在运行的事务中的最小时间戳，并且由于时间戳 +1 递增，watermark 也使用 +1 递增寻找符合的事务。

具体实现：除非 watermark 为空，否则添加时不修改 watermark 值，删除时判断是否还有时间戳为 watermark 的事务，没有就递增搜索，直到 `commit_ts`。

```cpp
auto Watermark::AddTxn(timestamp_t read_ts) -> void {
  if (read_ts < commit_ts_) {
    throw Exception("read ts < commit ts");
  }
  // 若 map 为空（没有活跃事务），直接设置 watermark
  if (current_reads_.empty()) {
    watermark_ = read_ts;
  }
  ++current_reads_[read_ts];  // 频率计数
}

auto Watermark::RemoveTxn(timestamp_t read_ts) -> void {
  auto iter = current_reads_.find(read_ts);
  if (iter == current_reads_.end()) {
    throw Exception("read ts not found");
  }
  if (iter->second == 1) {
    current_reads_.erase(iter);   // 最后一个使用此 read_ts 的事务退出
  } else {
    --iter->second;               // 还有其他事务使用相同 read_ts
  }
  // 若移除的是当前 watermark，需要更新 watermark
  if (read_ts == watermark_ && !current_reads_.empty()) {
    // 向前推进 watermark，找到下一个有活跃事务的 read_ts
    // current_reads_ 是有序 map，可直接取 begin()->first，无需循环
  }
}
```

> **实现优化点：** `current_reads_` 是有序 `std::map`，`RemoveTxn` 的 watermark 更新可以直接取 `begin()->first` 作为新 watermark，无需循环线性扫描，更加高效。

---

## 四、Task #2 — 存储格式与顺序扫描

### 4.1 TupleMeta.ts_ 的完整语义

```
ts_ 值域分析：

[0, TXN_START_ID)      → 已提交事务的 commit_ts（小整数）
[TXN_START_ID, MAX)    → 进行中事务的临时 txn_id（极大整数）
```

**可见性判断核心逻辑：**

```
令 tuple_ts = tuple_meta.ts_
令 txn_ts = txn->GetReadTs()
令 my_txn_id = txn->GetTransactionId()  // >= TXN_START_ID

情况1: tuple_ts <= txn_ts
    → tuple 已提交且时间 ≤ 当前快照，直接可见

情况2: tuple_ts == my_txn_id
    → tuple 是本事务自己修改的，可见（读自己写的数据）

情况3: tuple_ts > txn_ts && tuple_ts != my_txn_id
    → tuple 被一个更新的已提交事务或另一个未提交事务修改
    → 需要沿 undo log 链向后找到对当前快照可见的版本
    → 可能的情况：
       3a: tuple_ts >= TXN_START_ID（另一个正在进行的事务修改）
       3b: tuple_ts < TXN_START_ID 但 > txn_ts（另一个较新的已提交事务修改）
```

### 4.2 UndoLog 稀疏存储的实现细节

```cpp
struct UndoLog {
  bool is_deleted_;                   // true 表示"这步操作是删除"
  std::vector<bool> modified_fields_; // 大小 = schema.GetColumnCount()
                                      // modified_fields_[i] = true → 第 i 列在此 log 中有旧值
  Tuple tuple_;                       // 只包含 modified_fields_ 为 true 的列
                                      // 用 Schema::CopySchema() 生成子 schema 来解析
  timestamp_t ts_;                    // 此 UndoLog 记录的版本对应的时间戳
  UndoLink prev_version_;             // 指向更旧的 UndoLog
};
```

**示例：** 一张 4 列的表 `(a, b, c, d)`，事务将 `b` 和 `d` 从 (2, 4) 改为 (20, 40)：

```
modified_fields_ = [false, true, false, true]
tuple_.data_      = [(b=2), (d=4)]  // 只存修改了的列的旧值
                    // 解析时用 CopySchema([1, 3]) 生成 2 列 schema
```

**版本链中 ts_ 的含义：**

```
TableHeap tuple: ts_=TXN8（事务8临时id，表示事务8正在修改）
    ↓
UndoLog1: ts_=5（表示"若你想看 commit_ts=5 时的状态，apply 这个 log"）
    ↓
UndoLog2: ts_=3
    ↓
INVALID（链尾，再往前没有版本记录）
```

读事务 T with read_ts=6 想读这行：TableHeap 的 ts_=TXN8 不可见，取 UndoLog1，ts_=5 ≤ 6 → apply 此 log 后的版本对 T 可见，返回 apply UndoLog1 后的 tuple。

### 4.3 ReconstructTuple — 元组重建

**函数签名与语义：**

```cpp
auto ReconstructTuple(const Schema *schema,
                      const Tuple &base_tuple,    // TableHeap 中的最新 tuple
                      const TupleMeta &base_meta, // 对应的元数据
                      const std::vector<UndoLog> &undo_logs  // 从新到旧的 undo log 数组
) -> std::optional<Tuple>;
// 返回 nullopt 表示重建后的版本已被删除（即该历史版本不存在）
```

**执行流程图：**

```
base_tuple (TableHeap 当前版本)
    │
    ▼ apply UndoLog₁ (最近的变更 → 还原这一步)
 版本v₁
    │
    ▼ apply UndoLog₂
 版本v₂ ← 这可能就是我们需要的历史版本
    │
    ▼ apply UndoLog₃
 版本v₃（更古老的版本）
```

特别需要注意的点是对**已删除数据的操作**——执行前已删除、执行后已删除这两种情况，`is_deleted` 标志需要随 undo log 一起回退。

**版本链遍历顺序：**

```
新 ←————————————————→ 旧

TableHeap → UndoLog1 → UndoLog2 → UndoLog3 → INVALID
            (最近的变更)          (最早的变更)
```

`ReconstructTuple` 接收的 `undo_logs` 数组是新到旧顺序，apply 时需要正向遍历（先 apply 最新的 undo，再 apply 较旧的 undo），才能从 base_tuple 一步步回退到目标版本。

### 4.4 顺序扫描（SeqScan）可见性逻辑

顺着版本链扫描：如果时间戳满足条件（情况1或2）直接读取；否则顺着 undo log 版本链构建历史版本。

```cpp
// SeqScan 中的使用（seq_scan_executor.cpp）
std::vector<UndoLog> undo_logs;
UndoLink undo_link = txn_manager->GetUndoLink(*rid).value_or(UndoLink{});
auto optional_undo_log = txn_manager->GetUndoLogOptional(undo_link);

// 收集足够多的 undo log，直到时间戳满足条件
while (optional_undo_log.has_value() && tuple_ts > txn_ts) {
  undo_logs.push_back(*optional_undo_log);
  tuple_ts = optional_undo_log->ts_;
  undo_link = optional_undo_log->prev_version_;
  optional_undo_log = txn_manager->GetUndoLogOptional(undo_link);
}

// 检查是否找到了满足条件的版本
if (tuple_ts > txn_ts) {
  continue;  // 整个版本链都比 txn_ts 新，此 tuple 对当前事务不可见
}

auto new_tuple = ReconstructTuple(&schema, *tuple, tuple_meta, undo_logs);
```

> **SERIALIZABLE 隔离级别处理：** 构造 SeqScanExecutor 时若 `isolation_level == SERIALIZABLE` 且存在 `filter_predicate_`，需记录扫描谓词（`txn->AppendScanPredicate`），用于提交时 `VerifyTxn` 检测幻读。

| 情况       | 条件                                        | 处理                         |
| ---------- | ------------------------------------------- | ---------------------------- |
| 直接可见   | `tuple_ts ≤ txn_ts`                         | 直接使用 TableHeap 中的版本  |
| 本事务自写 | `tuple_ts == txn->GetTransactionId()`       | 直接使用（读自己写的数据）   |
| 需要回溯   | `tuple_ts > txn_ts && tuple_ts ≠ my_txn_id` | 遍历 undo log 链重建历史版本 |

---

## 五、Task #3 — MVCC Executors

task3 是这部分的核心内容，实现增删改查这些基本操作在事务下的执行过程以及事务的提交。

### 5.1 Insert

向 table 中插入数据即可，需要同时向 `write_set` 中传入对应 rid（后续会在 commit 中统一修改对应 Tuple 的时间戳，表明该 Tuple 没有事务在执行；也可以用于 Abort 中取消对 Tuple 的操作）。

```cpp
auto InsertExecutor::Next([[maybe_unused]] Tuple *tuple, RID *rid) -> bool {
  if (has_inserted_) { return false; }  // 确保只执行一次
  has_inserted_ = true;

  int count = 0;
  auto txn = exec_ctx_->GetTransaction();
  auto txn_mgr = exec_ctx_->GetTransactionManager();
  auto catalog = exec_ctx_->GetCatalog();
  auto table_info = catalog->GetTable(plan_->GetTableOid());
  auto indexes = catalog->GetTableIndexes(table_info->name_);

  while (child_executor_->Next(tuple, rid)) {
    InsertTuple(txn, txn_mgr, table_info, indexes, *tuple);
    ++count;
  }
  // 输出受影响行数
  std::vector<Value> result{{TypeId::INTEGER, count}};
  *tuple = Tuple{result, &GetOutputSchema()};
  return true;
}
```

### 5.2 Commit

commit 主要是两个操作：

1. 修改 `last_commit_ts_` 以及当前事务的 `commit_ts`
2. 读取 `write_set`，修改事务执行过写操作的 Tuple 的时间戳为当前事务提交的时间戳 `commit_ts`

```cpp
auto TransactionManager::Commit(Transaction *txn) -> bool {
  std::unique_lock<std::mutex> commit_lck(commit_mutex_);  // 串行化提交
  // commit_ts = ++last_commit_ts_
  // 遍历 write_set_，将所有相关 tuple 的 ts_ 改为 commit_ts
  // running_txns_.RemoveTxn(txn->read_ts_)
  return true;
}
```

**两把锁的用意：**

- `commit_mutex_`：序列化提交操作，确保 `last_commit_ts_` 的递增是原子性的，避免两个事务获得同一个 commit_ts
- `txn_map_mutex_`：保护 `txn_map_` 和 Watermark 的更新

### 5.3 Update && Delete

Update 和 Delete 操作差不多，都是根据读取的数据进行操作，需要判断读取的数据时间戳，有三种情况：

1. **Tuple 时间戳 ≤ 当前事务时间戳**：表明这个 Tuple 没有被当前事务之后的事务操作或正在被其他事务操作，直接操作 Tuple，然后写入 UndoLog、写入 writeSet、更新 VersionLink
2. **Tuple 的时间戳等于当前事务 ID**：表明这个 Tuple 之前被当前事务操作过，需要**更新之前提交的 UndoLog——UndoLog 合并**
3. **其他情况**：包括时间戳比当前事务大（被当前事务之后的事务操作过了）、已经有其他事务在操作了——这两种情况都是**写写冲突**

**UndoLog 合并（第一次写——创建）：**

当 `tuple_meta.ts_ ≤ txn->GetReadTs()`（本事务第一次写这条 tuple）：

```cpp
// 计算哪些列发生了变化
std::vector<bool> new_modified_fields;
std::vector<Value> new_modified_values;
std::vector<uint32_t> cols;
for (uint32_t i = 0; i < schema.GetColumnCount(); ++i) {
  auto old_value = cur_tuple.GetValue(&schema, i);
  if (old_value.CompareEquals(new_values[i]) == CmpBool::CmpTrue) {
    new_modified_fields.emplace_back(false);  // 未变化
  } else {
    new_modified_fields.emplace_back(true);   // 已变化，记录旧值
    new_modified_values.push_back(old_value);
  }
}
// 创建 UndoLog 并追加到事务的 undo_logs_ 数组
undo_link = txn->AppendUndoLog(UndoLog{
    tuple_meta.is_deleted_,
    new_modified_fields,
    new_modified_tuple,
    tuple_meta.ts_,  // 旧版本的时间戳
    undo_link        // 指向更旧的版本
});
```

**UndoLog 合并（同一事务再次写）：**

当 `tuple_meta.ts_ == txn->GetTransactionId()`（本事务已经写过这条 tuple）时，合并现有 UndoLog 而非新建：

```cpp
// 目标：undo_log 始终记录"从事务开始前的状态到当前状态的完整差异"

// 事务第一次写：a=1→2  → undo_log: {a: 1}（记录 a 的原始值）
// 事务第二次写：a=2→3, b=5→6  → undo_log 合并为: {a: 1, b: 5}
//            （a 的原始值仍是 1，b 新加入，原始值是 5）

// 若不合并而是新建：
//   undo_log1: {a: 2, b: 5} → undo_log0: {a: 1}
//   回滚时需要 apply 两次，链条更长
txn->ModifyUndoLog(undo_link.prev_log_idx_, UndoLog{
    undo_log->is_deleted_,
    merged_modified_fields,
    merged_modified_tuple,
    undo_log->ts_,
    undo_log->prev_version_
});
```

**Update 两阶段执行——解决 Halloween Problem：**

Update 的实现分为两个明确的阶段：

```
阶段一（收集）: while(child->Next()) → old_tuples, new_tuples, rids 全部收集
阶段二（写入）: 遍历收集好的数据，执行实际的写操作
```

经典问题描述：假设对每个薪资 < 1000 的员工加薪 10%，若边读边写：

1. 扫描到 Alice（薪资 900），更新为 990
2. 表迭代器继续推进，再次扫描到 Alice（新薪资 990，仍 < 1000）再次更新为 1089
3. 循环往复，Alice 的薪资无限增长

先将所有满足条件的 (old_tuple, new_tuple, rid) 全部收集，然后批量写入，避免读到自己刚写的数据。

### 5.4 Stop-the-world Garbage Collection

垃圾回收实现比较简单，主要利用 task1 中的 watermark：对于所有已提交的事务，如果它的提交时间比 `watermark_ts` 小或者 `undo_log` 为空，这两种情况都表明这个事务不再被访问，直接回收即可，总共就一个循环加上一个判断。

- **watermark** 本来就是为了垃圾回收机制而设计的
- **undoLog 为空的事务**：不会有版本回退访问到的情况
- **提交时间戳小于 watermark 的事务**：undoLog 是存放修改之前的数据状态，正在运行的事务不会需要这段历史了（仔细想一想，undoLog 是存放修改之前的数据状态）

---

## 六、Task #4 — Primary Key Index

task4 中引入索引，而且是单索引，这时候对版本链的操作发生了变化。由于索引指向的地址不会发生变化，对于同一个索引指向的 Tuple 存在删除数据后又重新插入的情况，同时在插入时需要判断索引冲突，引入了索引版本链才算完整（从 Reconstruction 的操作来看，存在从删除状态到未删除状态）。

> **兼容性说明：** 由于可扩展哈希不支持多索引指向同一个 Tuple，Project4 和 Project3 不兼容。底层实现也有冲突：Project3 更新操作是删除原数据、创建新的 Tuple（Tuple 大小不固定），但 Project4 是原地修改（Tuple 大小固定）。

### 6.1 Inserts（含并发）

这一部分便是重构 Insert，考虑存在索引的情况。大致实现（包括并发）：

1. **判断索引冲突**：有就直接报写写冲突，同时记录插入的 Tuple 的索引是否存在
2. **索引存在时**：判断是否存在写写冲突，即索引指向位置的 Tuple 是否已删除且没有其他事务正在操作，满足条件就直接插入，修改 versionLink、UndoLog 以及 writeSet
3. **索引不存在时**：和原本的 Insert 一样，插入数据，更新 VersionLink（第一次插入没有 undoLog），然后创建对应索引
4. **并发处理关键**：在步骤 3 中进行索引创建时检查冲突——如果有多个指向同一个索引的 Tuple 插入，便会发生竞争插入，最终只有一个插入成功，其余全部失败

**RID 复用的重要性：**

当一行数据先被删除再被插入相同主键时，BusTub 复用原来的 RID：节省空间，维护索引一致性，简化 GC。

```
提交状态: (deleted, ts=5) → UndoLog₁(is_deleted=false, ..., ts=3) → INVALID
                                      ↑ 旧的有效数据

插入后:   (value=X, ts=TXN8) → UndoLog_new(is_deleted=true, ts=5) → UndoLog₁ → INVALID
                              ↑ 新插入的 UndoLog 记录"我是在删除状态上覆盖的"
```

**主键约束与并发插入：**

```
场景：两个事务同时插入相同主键

T1 begin(read_ts=5):
  InsertTuple(key=X) → 索引插入: key=X → RID=1, tuple_ts=TXN1

T2 begin(read_ts=5):（与 T1 并发）
  InsertTuple(key=X)
  → ScanKey 找到 RID=1，tuple_ts=TXN1 > read_ts=5 && != T2的id → 写写冲突 → abort T2
```

这里体现了 First-Writer-Wins：T1 先完成索引插入，T2 看到冲突后 abort。

### 6.2 Index Scan, Deletes and Updates（含并发）

由于引入了索引，根据 project3 中实现的优化，这里会调用 IndexScan，需要修改 IndexScan 使其能够适应事务的操作，实现大致和 SeqScan 差不多。Delete 和 Update 操作没有太大变化，主要是由于索引存在，版本链和之前不同，存在对已删除的地方进行插入，所以对 undoLog 和 versionLink 的操作有小部分修改。

**并发实现——原子自旋锁设计：**

并发使用 versionLink 来实现原子操作。实现的关键是 `UpdateVersionLink` 和 `GetVersionLink` 函数中有锁，由 txn_manager 管理，保证所有事务对 versionLink 的操作是原子的（原子操作是实现并发的关键——执行过程不会被其他事务插队）。这类似于操作系统中利用底层原子操作来实现并发的方法——**自旋检查，检查过程是原子的**，通过这部分实现并发。

具体实现步骤：

1. **先检查写写冲突**，有写写冲突直接退出
2. **自旋检查** `version_link` 中 `in_progress` 是否为 false（表明没有其他事务在操作该 rid 指向的 tuple）
3. **调用 `UpdateVersionLink` 函数**，传入检查函数，原子地检查是否已经有其他事务获取了 `in_progress`，然后修改 `in_progress` 为 true，其他事务只能自旋等待。如果执行失败跳转步骤 1 重新检查并进入自旋等待
4. **再次检查写写冲突**（double-check，这部分貌似没有被执行过，但其他大佬都写了，我也写一下）
5. **个人实现要点（其他大佬没写的部分）：**
   - **versionLink 始终非空**：在最开始插入时，原本的设计 versionLink 为空，但这样自旋判断存在问题（没有 versionLink 就没有 `in_progress`）。因此在一开始就插入 versionLink，让它指向一个不存在的 UndoLog（默认值），通过 `GetUndoLogOptional` 函数来判断是否到版本链终点，而不是用 UndoLog 指向为空来表示空链。这其实也是我最开始就有的想法——让 versionLink 的指针指向为空表示没有 UndoLog，而不是 UndoLog 指向为空，这样写也更舒服。
   - **commit 时统一释放写锁**：在 commit 时统一从 write_set 中修改 versionLink 的 `in_progress` 为 false，这样和 write_set 同步了，不需要额外的数据结构，也防止被其他事务插队，正好符合设计。

### 6.3 Primary Key Updates

主键更新也是大坑，倒是没有并发，但是是单索引，不符合 Project3 的测试。实现的关键点：

1. 首先获取所有的数据
2. 对其进行更新操作，判断是否有索引冲突（可以通过 Expression 中修改的位置下标来判断是否有索引变化）
3. 统一删除原本的数据（先获取所有数据再统一删除，防止影响自身的读取）
4. 插入新的位置

这里 Delete 然后 Insert 同一行时，`InsertTuple` 会走 **RID 复用** 路径，因为 `ScanKey` 仍能找到旧的索引项对应的已删除 tuple。

---

## 七、Bonus Task #1 — Abort

### 7.1 实现思路

Abort 主要通过 `write_set` 将事务操作过的数据恢复原本的状态，同时回退 versionLink 和 UndoLog。这部分主要是针对前面写写冲突抛出的 TAINTED 的事务，将其所做的操作复原。

**关键点：**

- 通过 `write_set_` 精确知道"我修改了哪些行"，不需要全表扫描
- 对每行，通过第一个 UndoLog（本事务创建的）进行还原
- 若没有 UndoLog（新插入的 tuple），直接标记为删除并设 ts_=0
- undo_log 本身不会从事务中删除，由 GC 负责

```cpp
void TransactionManager::Abort(Transaction *txn) {
  std::unique_lock<std::mutex> commit_lck(commit_mutex_);
  // 遍历 write_set_
  // 对每行：通过第一 UndoLog 还原，或直接删除（新插入行）
  running_txns_.RemoveTxn(txn->read_ts_);
}
```

### 7.2 并发控制方式的变化

这里更重要的一个变化是：**并发的实现方式发生了改变**。原本是利用 `in_progress` 来实现并发，从这里开始使用底层的 page 锁来实现并发了，原先的 versionLink 锁的部分可以删除了，并发的实现也变得更简单了。

---

## 八、Bonus Task #2 — Serializable Verification

### 8.1 实现思路

这一部分主要检查在事务并发过程中，是否存在并发过程中事务执行顺序不同导致不同的结果——即序列化的正确性，如果存在这种情况就需要进行 Abort。主要实现 `VerifyTxn` 函数，检查**提交在当前事务创建之后的已提交事务**中是否存在和当前事务有序列化冲突的情况。

### 8.2 SSI 检测原理

**检测逻辑：** 在本事务 `read_ts` 之后提交的其他事务，是否修改了本事务扫描谓词覆盖的数据行？

```
事务时间线：
    T1 begin(read_ts=5) ─────────────── T1 commit
    T2 begin ──────── T2 commit(commit_ts=7)

T2 的 write set: {row_A}
T1 的 scan predicates: {table=employees, predicate=(salary < 10000)}

VerifyTxn 检测：
  row_A 在 commit_ts=7 时是否满足 (salary < 10000)?
  如果满足 → T1 扫描时"本应"看到这行变化 → 幻读 → abort T1
```

| 异常类型   | SI                          | SSI (BusTub 实现)                               |
| ---------- | --------------------------- | ----------------------------------------------- |
| 脏读       | ✅ 防止                      | ✅ 防止                                          |
| 不可重复读 | ✅ 防止                      | ✅ 防止                                          |
| 丢失更新   | ✅ 防止（First-Writer-Wins） | ✅ 防止                                          |
| 写偏序     | ❌ 不完全防止                | ✅ 防止（谓词检测）                              |
| 幻读       | 部分防止                    | ✅ 防止                                          |
| 额外开销   | 无                          | Commit 时 VerifyTxn（O(txn×writes×predicates)） |

### 8.3 VerifyTxn 的局限性

1. **保守性（False Positive）**：只检测"写集 × 谓词"的交集，而非精确的循环依赖图，可能误判（abort 了实际上可串行化的事务）

   ```
   T2 commit_ts=7：UPDATE employees SET dept='HR' WHERE id=42
   T1 scan predicate：dept='Engineering'
   row_42 在 commit_ts=7 时 dept='Engineering' → VerifyTxn 返回 false，T1 abort
   但实际上先 T1 后 T2 完全可以串行化，不是真正的冲突
   ```

2. **性能开销**：需要遍历 `txn_map_` 中所有已提交事务（O(n)）× 写集大小 × 谓词数量，高并发时开销显著
3. **谓词粒度粗**：只记录 SeqScan/IndexScan 时的谓词，不记录点查、Join 等操作的访问集，可能漏掉部分冲突

---

## 九、事务完整生命周期

### 9.1 数据流图

```
Begin()
  └── read_ts = last_commit_ts（原子读）
  └── txn_id = next_txn_id_++
  └── 加入 txn_map_，注册到 Watermark

执行阶段（Read/Write）
  └── Read: 可见性检查 → 按需 ReconstructTuple
  └── Write: 冲突检测 → 更新 TableHeap → 创建/合并 UndoLog → 记录 write_set_

Commit()
  └── [commit_mutex_] commit_ts = ++last_commit_ts
  └── write_set_ 中所有 tuple: ts_ = commit_ts
  └── VersionLink.in_progress = false（释放"写锁"标记）
  └── running_txns_.RemoveTxn(read_ts)

Abort()
  └── 遍历 write_set_，通过 UndoLog 还原所有修改
  └── 无 UndoLog 的 tuple → is_deleted=true, ts_=0
  └── running_txns_.RemoveTxn(read_ts)

GarbageCollection()
  └── watermark = running_txns_.GetWatermark()
  └── 从 txn_map_ 删除 commit_ts < watermark 的已完成事务
```

### 9.2 一条 tuple 的版本演变示例

```
初始状态（commit_ts=2）:
  TableHeap: (a=1, b=5), ts_=2, is_deleted_=false
  version_info_: prev_ = INVALID

T1(read_ts=2) 执行 UPDATE a=10:
  创建 UndoLog: {is_deleted_=false, modified_fields_=[true,false], tuple_=(a=1), ts_=2, prev_=INVALID}
  TableHeap: (a=10, b=5), ts_=TXN1
  version_info_: prev_ → UndoLog_T1

T1 Commit (commit_ts=5):
  TableHeap: (a=10, b=5), ts_=5

T2(read_ts=5) 执行 DELETE:
  创建 UndoLog: {is_deleted_=false, modified_fields_=[true,true], tuple_=(a=10,b=5), ts_=5, prev_→UndoLog_T1}
  TableHeap: ts_=TXN2, is_deleted_=true
  version_info_: prev_ → UndoLog_T2 → UndoLog_T1

T2 Commit (commit_ts=8):
  TableHeap: (a=10, b=5), ts_=8, is_deleted_=true

T3(read_ts=4) 读这行:
  tuple_ts=8 > read_ts=4 → 需要重建
  collect: UndoLog_T2(ts_=5>4) → UndoLog_T1(ts_=2≤4), 停止
  apply UndoLog_T2: 撤销 T2 的删除 → (a=10, b=5, deleted=false)
  结果: (a=10, b=5) 可见（T1 的修改对 T3 可见，T2 的删除对 T3 不可见）
```

---

## 思考

到此 bustub 23fall 的 4 个 project 已经圆满完成，所有分数已经拿齐，能做的优化也基本做了，除了 project3 的 LeaderBoard，其他的都做了，也取得了不错的成绩。接下来按照安排是去实现 tinykv，可能也会写一写文章吧，还有 bustub 24fall 可能也会去做，稍微瞄了一眼，24fall 也改了不少，而且 B+树的实现是必须要去做的。总体感觉 bustub 一年比一年难了，也更加完整了，也是变得越来越好了，希望这门课程变得越来越好吧。通过这 4 个 project 也是学到了很多东西，对 C++、对数据库、对代码能力都是巨大的提升，有兴趣的同学都可以来写一写，还是挺不错的。

最后，感谢 CMU 15445 的老师和助教们的辛勤付出。

考虑到网上实现最后这部分内容的文章比较少，代码也是没有，我把我的[代码](https://github.com/thekingking/bustub)放在这，这部分课程也是即将结束，应该也没几个人写了，有兴趣的同学可以参考一下。
