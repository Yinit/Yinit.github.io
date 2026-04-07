---
title: "TinyKV 分布式事务实现与复习"
date: 2026-04-07T06:58:45Z
lastMod: 2026-04-07T06:58:45Z
draft: false # 是否为草稿
author: ["tkk"]

categories: ["数据库"]

tags: ["TinyKV", "分布式事务", "Percolator", "MVCC", "2PC"]

keywords: ["TinyKV", "Percolator", "MVCC", "2PC", "分布式事务", "快照隔离", "两阶段提交"]

description: "深入解析 TinyKV Project4 分布式事务实现，涵盖 Percolator 协议、MVCC 存储模型、两阶段提交与事务冲突处理全流程" # 文章描述，与搜索优化相关
summary: "深入解析 TinyKV 基于 Percolator 协议的分布式事务实现，涵盖 MVCC 存储模型、2PC 流程与异常恢复机制" # 文章简单描述，会展示在主页
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

## 1. 概述与背景

### 1.1 Project4 整体目标

在 TinyKV 的前三个 Project 中，我们构建了一个具有 Multi-Raft 能力的分布式 KV 存储：每个 Region 内部通过 Raft 协议保证了数据的强一致性与高可用性。然而，这个系统只提供了**单 key 的原子性操作**，无法保证跨 key、跨 Region 的原子提交。

Project4 的目标是在 Multi-Raft KV 之上实现**分布式事务**，让多个 key 的读写操作能够以原子、隔离的方式执行。具体地，TinyKV 采用 **Percolator 协议**（Google 2010 年提出）实现了一套基于 **MVCC（Multi-Version Concurrency Control）** 的两阶段提交（2PC）分布式事务。

### 1.2 为什么需要分布式事务

考虑一个银行转账场景：

```
账户 A: 扣减 100 元
账户 B: 增加 100 元
```

如果这两个操作不是原子的，可能出现：

- A 扣减成功，B 增加失败 → 钱凭空消失
- 中间状态被其他事务读到 → 脏读
- 两个并发转账同时读到旧值 → 丢失更新

在分布式系统中，账户 A 和账户 B 可能分布在不同的 Region（不同的 Raft Group），这使得原子性保证更加复杂。

### 1.3 TinyKV 事务架构概览

```
┌─────────────────────────────────────────────────────────────┐
│                        Client SDK                           │
│  (获取 TSO, 构造 PrewriteRequest/CommitRequest, 重试逻辑)    │
└───────────────────────────┬─────────────────────────────────┘
                            │ gRPC
┌───────────────────────────▼─────────────────────────────────┐
│                    TinyKV Server (server.go)                 │
│  KvGet | KvPrewrite | KvCommit | KvScan                     │
│  KvCheckTxnStatus | KvBatchRollback | KvResolveLock         │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│                  MVCC Layer (transaction.go)                 │
│  MvccTxn: GetLock/PutLock | GetValue/PutValue               │
│           CurrentWrite | MostRecentWrite                    │
└───────────────────────────┬─────────────────────────────────┘
                            │
┌───────────────────────────▼─────────────────────────────────┐
│              RaftStore (Multi-Raft KV Storage)               │
│  三个列族: CfLock | CfWrite | CfDefault                     │
│  底层: badger (LSM-Tree)                                     │
└─────────────────────────────────────────────────────────────┘
```

### 1.4 Percolator 协议来源与适用场景

Google Percolator（2010）最初是为了支持 Google 网页索引的**增量更新**而设计的。传统批处理方式（MapReduce）每次需要重新处理所有网页，效率低下。Percolator 通过在 BigTable 之上实现跨行事务，支持对索引的增量修改。

Percolator 的核心设计思路：

- 利用底层存储（BigTable/RocksDB）已有的**单行原子性**
- 通过 **Primary Key** 充当事务协调者，避免独立协调者的单点问题
- 基于 **MVCC** 实现快照隔离，读操作不需要锁
- **乐观并发控制**：先尝试加锁，冲突时回滚重试

Percolator 特别适合：

- 读多写少的场景（读不加锁）
- 跨行、跨节点的原子更新
- 需要快照隔离的 OLTP 工作负载

---

## 2. 单机事务理论基础

### 2.1 ACID 四要素深度解析

事务是数据库的基本操作单位，ACID 是衡量事务质量的四个属性：

#### 2.1.1 Atomicity（原子性）

**定义**：事务中的所有操作要么全部成功，要么全部失败回滚，不存在部分执行的中间状态。

**实现机制**：

1. **Undo Log（回滚日志）**：在修改数据之前，先将原始数据写入 undo log。若事务需要回滚，通过 undo log 恢复数据到修改前的状态。
   - InnoDB 的 undo log 存储在系统表空间（或独立的 undo 表空间）
   - 每条记录包含：事务 ID、操作类型（INSERT/UPDATE/DELETE）、旧值
   - MVCC 也依赖 undo log 实现历史版本的读取

2. **WAL（Write-Ahead Log）**：所有数据修改在写入磁盘前，必须先将日志写入 redo log。这保证了崩溃恢复时能够重放已提交事务的修改。
   - InnoDB 的 redo log 以循环方式写入（ib_logfile0, ib_logfile1）
   - `innodb_flush_log_at_trx_commit` 控制刷盘时机：
     - `0`：每秒刷盘，性能最好但最多丢失 1 秒数据
     - `1`（默认）：每次提交都刷盘，最安全
     - `2`：写入 OS 缓存但每秒 fsync，折中方案

**两段式提交（在单机 InnoDB 中）**：

```
事务执行阶段: 修改 buffer pool + 写 undo log + 写 redo log buffer
提交阶段:
  1. 写 binlog（如果开启）
  2. 写 redo log（prepare 状态）
  3. 写 binlog（commit 标记）
  4. 写 redo log（commit 状态）
```

**注**：InnoDB 内部的 2PC 是为了保证 redo log 和 binlog 的一致性，与跨节点的分布式 2PC 概念不同。

#### 2.1.2 Consistency（一致性）

**定义**：事务执行前后，数据库从一个合法的状态转变到另一个合法的状态，满足所有预定义的约束和规则。

**关键理解**：

- 一致性是事务的**目的**，而 AID 是实现一致性的**手段**
- 一致性包含两个层面：
  - **数据库层面**：主键唯一性、外键约束、非空约束等
  - **业务层面**：账户余额不能为负、库存不能超卖等（由应用程序保证）
- 某些文献中，C 被认为是 AID 的结果，而非独立属性

**注意**：CAP 定理中的 Consistency 是指**线性一致性（Linearizability）**，与 ACID 中的 Consistency 含义不同，需要加以区分。

#### 2.1.3 Isolation（隔离性）

**定义**：并发执行的多个事务之间相互隔离，一个事务的中间状态对其他事务不可见。

**并发异常类型**：

| 并发异常                              | 描述                                     | 示例                                                         |
| ------------------------------------- | ---------------------------------------- | ------------------------------------------------------------ |
| **脏读（Dirty Read）**                | 读到未提交的数据                         | T1 修改 x=10（未提交），T2 读到 x=10，T1 回滚                |
| **不可重复读（Non-Repeatable Read）** | 同一事务内两次读同一行，值不同           | T1 读 x=5，T2 提交 x=10，T1 再读 x=10                        |
| **幻读（Phantom Read）**              | 同一事务内两次范围查询，结果集不同       | T1 查询 age>18 得 10 条，T2 插入新行，T1 再查得 11 条        |
| **丢失更新（Lost Update）**           | 两事务同时读旧值，先后写入，后者覆盖前者 | T1 读 x=5，T2 读 x=5，T1 写 x=6，T2 写 x=7（T1 的更新丢失）  |
| **读偏斜（Read Skew）**               | 读到不一致的数据组合                     | T1 先读 x，T2 修改 x 和 y，T1 再读 y，读到不一致的 x, y 组合 |
| **写偏斜（Write Skew）**              | 基于快照做出的决策，破坏了业务约束       | 见下方详解                                                   |

**写偏斜（Write Skew）详解**：

```
约束: 值班医生数 >= 1
T1: 查询在岗医生 = 2，决定让 A 下班，修改 A.on_call = false
T2: 查询在岗医生 = 2，决定让 B 下班，修改 B.on_call = false
结果: A 和 B 都下班，值班医生 = 0，违反约束
```

写偏斜在快照隔离（Snapshot Isolation）下也会发生，需要串行化快照隔离（SSI）才能解决。

#### 2.1.4 Durability（持久性）

**定义**：已提交的事务对数据的修改是永久性的，即使系统崩溃也不会丢失。

**实现机制**：

1. **WAL（Write-Ahead Log）**：已提交事务的 redo log 必须在数据页刷盘之前先持久化到磁盘。崩溃后可通过重放 redo log 恢复未刷盘的数据。

2. **fsync 系统调用**：确保数据真正写入磁盘（而非仅在 OS 缓冲区）。`innodb_flush_log_at_trx_commit=1` 时每次提交都会调用 fsync。

3. **Checkpoint 机制**：定期将 buffer pool 中的脏页刷盘，减少崩溃恢复时需要重放的 redo log 量。

4. **Double Write Buffer**：防止"部分写"（Partial Write）问题——当 16KB 的数据页只写了一半时系统崩溃，通过 doublewrite buffer 保证页面完整性。

### 2.2 事务隔离级别

SQL 标准定义了四个事务隔离级别，从低到高依次为：

#### 2.2.1 四种隔离级别定义

**READ UNCOMMITTED（读未提交）**：

- 允许读到其他事务未提交的修改
- 存在脏读、不可重复读、幻读问题
- 几乎不使用，无实际价值

**READ COMMITTED（读已提交）**：

- 只能读到已提交的数据，防止脏读
- 仍存在不可重复读、幻读问题
- SQL Server、Oracle 的默认级别
- 实现：每次读操作都创建新的 Read View（最新快照）

**REPEATABLE READ（可重复读）**：

- 同一事务内多次读同一行，结果相同，防止不可重复读
- 在某些实现中可防止幻读（InnoDB 通过间隙锁）
- MySQL InnoDB 的默认级别
- 实现：事务开始时创建 Read View，之后的读操作都使用同一个快照

**SERIALIZABLE（可串行化）**：

- 最强隔离，事务完全串行执行（或等价于串行）
- 防止所有并发异常，包括幻读和写偏斜
- 性能最差，通常使用 2PL（两阶段锁）实现

#### 2.2.2 各级别允许/禁止的并发异常

| 隔离级别         | 脏读      | 不可重复读 | 幻读                   | 写偏斜 |
| ---------------- | --------- | ---------- | ---------------------- | ------ |
| READ UNCOMMITTED | ✓（允许） | ✓          | ✓                      | ✓      |
| READ COMMITTED   | ✗（禁止） | ✓          | ✓                      | ✓      |
| REPEATABLE READ  | ✗         | ✗          | ✓（标准）/ ✗（InnoDB） | ✓      |
| SERIALIZABLE     | ✗         | ✗          | ✗                      | ✗      |

**注**：

- InnoDB 的 REPEATABLE READ 通过间隙锁（Gap Lock）和临键锁（Next-Key Lock）在大多数场景下防止了幻读，但严格来说并非完全等价于 SQL 标准的 SERIALIZABLE
- 写偏斜（Write Skew）在快照隔离（SI）下依然存在，需要 SSI 才能防止

#### 2.2.3 MySQL InnoDB 如何防止幻读

InnoDB 在 REPEATABLE READ 级别下通过**间隙锁（Gap Lock）**防止幻读：

```sql
-- 事务 T1
SELECT * FROM orders WHERE amount > 100 FOR UPDATE;
-- 对查询范围加 Next-Key Lock（临键锁 = 行锁 + 间隙锁）
-- 其他事务无法在此范围内插入新行

-- 事务 T2（被阻塞）
INSERT INTO orders (amount) VALUES (150);  -- 阻塞！
```

**Next-Key Lock** = 行锁 + 该行之前的间隙锁，范围为 `(previous_key, current_key]`

**注意**：

- `SELECT ... FOR UPDATE` 和 `SELECT ... IN SHARE MODE` 是**当前读**，会加锁
- 普通 `SELECT` 是**快照读**，使用 MVCC，不加锁

### 2.3 MVCC 多版本并发控制

#### 2.3.1 核心思想

MVCC（Multi-Version Concurrency Control，多版本并发控制）的核心思想是：**为数据维护多个历史版本，使读操作不需要等待写操作，写操作也不需要等待读操作**。

传统锁机制的问题：

- 读写互斥：读需要等写完成，写需要等读完成
- 高并发下锁竞争严重，吞吐量下降

MVCC 的解决方案：

- 写操作不修改原有数据，而是创建新版本
- 读操作根据事务开始时的**快照版本**读取对应版本，不受其他并发写的影响
- 读写并发，互不阻塞

#### 2.3.2 快照读 vs 当前读

**快照读（Snapshot Read）**：

- 读取某个时间点的数据快照，可能不是最新版本
- 不加锁，不影响其他事务
- SQL：`SELECT * FROM table WHERE ...`（普通 SELECT）
- InnoDB 通过 undo log 版本链实现历史版本的访问

**当前读（Current Read）**：

- 读取最新版本的数据
- 加锁（共享锁或排他锁）
- SQL：
  - `SELECT ... FOR UPDATE`（加排他锁）
  - `SELECT ... IN SHARE MODE`（加共享锁）
  - `INSERT`, `UPDATE`, `DELETE`（也是当前读，因为要修改最新数据）

**两者的本质区别**：快照读通过 MVCC 实现非阻塞读，当前读需要通过锁保证读到最新且一致的数据。

#### 2.3.3 InnoDB MVCC 实现

InnoDB 的每行数据都有两个隐藏字段：

| 字段          | 说明                               |
| ------------- | ---------------------------------- |
| `DB_TRX_ID`   | 最后修改该行的事务 ID              |
| `DB_ROLL_PTR` | 回滚指针，指向 undo log 中的旧版本 |
| `DB_ROW_ID`   | 隐藏主键（如果表没有显式主键）     |

**Undo Log 版本链**：

```
当前版本（row）:  trx_id=100, name="Alice", roll_ptr ──►
                                                         │
undo log v1:  trx_id=80, name="Bob", roll_ptr ──────────►
                                                         │
undo log v2:  trx_id=50, name="Charlie", roll_ptr ──────► NULL
```

每次更新都会将旧版本写入 undo log，形成链式结构。

#### 2.3.4 Read View 与可见性判断

**Read View** 是事务在执行快照读时创建的一个"视图"，包含：

```go
type ReadView struct {
    m_ids          []uint64  // 创建时活跃的事务 ID 列表
    m_low_limit_id uint64    // 创建时最大事务 ID + 1（>=此值的事务不可见）
    m_up_limit_id  uint64    // m_ids 中最小的事务 ID（<此值的事务可见）
    m_creator_trx_id uint64  // 创建此 Read View 的事务 ID
}
```

**可见性判断规则**（对某行的 `trx_id` 进行判断）：

```
1. trx_id == m_creator_trx_id  → 可见（自己修改的，当然能读）
2. trx_id < m_up_limit_id      → 可见（创建 Read View 前已提交）
3. trx_id >= m_low_limit_id    → 不可见（创建 Read View 后才开始的事务）
4. m_up_limit_id <= trx_id < m_low_limit_id:
   - trx_id 在 m_ids 中 → 不可见（创建时仍活跃，未提交）
   - trx_id 不在 m_ids 中 → 可见（已提交）
```

如果当前版本不可见，沿 `roll_ptr` 链找到旧版本，重复判断，直到找到可见版本或链尾（返回空）。

#### 2.3.5 RC 和 RR 的 Read View 创建时机差异

| 隔离级别            | Read View 创建时机                       | 效果                           |
| ------------------- | ---------------------------------------- | ------------------------------ |
| **READ COMMITTED**  | 每次执行 SELECT 都创建新的 Read View     | 能读到已提交的最新数据         |
| **REPEATABLE READ** | 事务内第一次执行 SELECT 时创建，之后复用 | 整个事务期间读到相同的数据快照 |

这就是为什么：

- RC 存在不可重复读（每次读视图不同，可以看到新提交的数据）
- RR 防止了不可重复读（整个事务使用同一快照）

### 2.4 并发控制机制对比

#### 2.4.1 悲观并发控制（PCC / 2PL）

**两阶段锁（Two-Phase Locking, 2PL）**：

- **加锁阶段**：事务可以加锁，不能释放锁
- **释放阶段**：事务开始释放锁后，不能再加新锁

**锁的类型**：

- 共享锁（S Lock）：允许并发读，阻塞写
- 排他锁（X Lock）：阻塞所有并发读写

**锁兼容矩阵**：

|        | S Lock   | X Lock   |
| ------ | -------- | -------- |
| S Lock | 兼容 ✓   | 不兼容 ✗ |
| X Lock | 不兼容 ✗ | 不兼容 ✗ |

**优缺点**：

- 优点：实现简单，能防止写冲突
- 缺点：读写互斥，高并发下性能差；可能死锁（需要死锁检测）

**适用场景**：写多、冲突率高的场景

#### 2.4.2 乐观并发控制（OCC）

**三个阶段**：

1. **读取（Read）**：读取数据，记录版本号
2. **验证（Validate）**：提交前检查是否有冲突（版本号是否改变）
3. **写入（Write）**：无冲突则提交，有冲突则回滚重试

**优缺点**：

- 优点：读操作不加锁，高并发下吞吐量高
- 缺点：冲突时需要回滚重试，冲突率高时性能差

**适用场景**：读多写少、冲突率低的场景（Percolator 就是乐观并发控制）

#### 2.4.3 时间戳排序（TO）

基于全局时间戳对事务进行排序，保证等价于某个串行执行顺序。

- 每个事务分配一个唯一的时间戳
- 读操作检查：不能读到更新时间戳的写
- 写操作检查：不能写已被更大时间戳读取的数据

Percolator 中的 MVCC 本质上就是时间戳排序的一种变体，通过 startTS 和 commitTS 确定事务的"位置"。

#### 2.4.4 选择策略总结

```
冲突率高（写密集型）  → 悲观锁（2PL）减少无效重试
冲突率低（读密集型）  → 乐观锁（OCC/MVCC）提高并发
长事务              → 避免持锁太久，考虑乐观锁
短事务/点查询        → 悲观锁开销可接受
```

---

## 3. 分布式事务理论

### 3.1 分布式系统的挑战

在单机数据库中，ACID 的实现相对简单：undo log、redo log、锁机制已经足够。但在分布式系统中，面临更多挑战：

#### 3.1.1 网络不可靠

- **消息丢失**：包括请求和响应都可能丢失
- **消息延迟**：网络延迟不确定，超时不意味着失败
- **消息乱序**：路由变化可能导致包乱序到达
- **网络分区**：节点间可能完全断联

一个常见的困境：协调者发送 Commit 消息后网络分区，参与者不知道该提交还是回滚。

#### 3.1.2 节点故障

- **崩溃故障**：节点宕机后无限期不响应（fail-stop）
- **拜占庭故障**：节点发送错误或恶意消息（更难处理）

TinyKV 只考虑崩溃故障（非拜占庭），但崩溃期间的状态恢复仍然复杂。

#### 3.1.3 时钟不一致

分布式系统中每个节点的时钟可能不同步：

- **时钟漂移**：石英晶振频率略有差异，随时间累积误差
- **NTP 同步误差**：通常 1-100ms，无法保证毫秒级精确排序

这意味着简单地用本地时钟作为事务 ID 无法保证全局唯一和单调递增，需要特殊的时钟方案（TSO、HLC、TrueTime）。

#### 3.1.4 CAP 定理与 BASE 理论

**CAP 定理**（Brewer, 2000）：在分布式系统中，以下三个属性最多只能同时满足两个：

- **C（Consistency）**：所有节点看到相同的数据（线性一致性）
- **A（Availability）**：每个请求都能得到响应（不一定是最新数据）
- **P（Partition Tolerance）**：网络分区时系统继续运行

在实际中，网络分区（P）是不可避免的，所以需要在 C 和 A 之间取舍：

- **CP 系统**（如 ZooKeeper、HBase）：分区时牺牲可用性，保证一致性
- **AP 系统**（如 Cassandra、DynamoDB）：分区时牺牲一致性，保证可用性

**BASE 理论**（对 CAP 的一种工程实践）：

- **Basically Available（基本可用）**：允许降级（响应延迟、部分数据不可用）
- **Soft State（软状态）**：允许系统处于中间状态（数据同步中）
- **Eventually Consistent（最终一致性）**：经过一定时间后，所有副本数据最终一致

BASE 是对强一致性（ACID）的放宽，适合对一致性要求不那么严格的场景。

### 3.2 两阶段提交（2PC）

2PC 是实现分布式事务的经典协议，也是 XA 协议和 Percolator 的理论基础。

#### 3.2.1 角色与流程

**角色**：

- **Coordinator（协调者）**：发起事务，协调所有参与者的提交/回滚决策
- **Participant（参与者）**：执行具体的数据修改，接受协调者的指令

**Phase 1：Prepare（准备阶段）**：

```
Coordinator                    Participant A     Participant B
     │                               │                │
     │─── Prepare ──────────────────►│                │
     │─── Prepare ────────────────────────────────────►│
     │                               │                │
     │      执行事务操作，写 undo log + redo log，加锁    │
     │                               │                │
     │◄── Vote Yes/No ───────────────│                │
     │◄── Vote Yes/No ────────────────────────────────│
```

- 协调者向所有参与者发送 Prepare 消息
- 参与者执行事务操作（写日志、加锁），但不提交
- 参与者回复 Yes（准备好了）或 No（无法执行）
- 此时参与者处于"就绪"状态，等待最终决定

**Phase 2：Commit/Abort（提交/回滚阶段）**：

```
所有 Yes:  Coordinator ─── Commit ───► 所有参与者 ─── Ack ────► 协调者
有 No:     Coordinator ─── Abort ────► 所有参与者 ──No Ack ────► 协调者
```

- 若所有参与者回复 Yes：协调者发送 Commit 消息，参与者提交并释放锁
- 若有任一 No：协调者发送 Abort 消息，参与者回滚并释放锁

#### 3.2.2 2PC 的问题

**问题 1：同步阻塞**

- Prepare 阶段后，参与者持锁等待协调者的 Commit/Abort 决定
- 等待期间，持锁资源无法被其他事务使用，导致吞吐下降

**问题 2：协调者单点故障**

```
情景：
1. 协调者向所有参与者发送 Commit
2. 参与者 A 收到并提交，协调者此时宕机
3. 参与者 B 没收到 Commit，进入无限等待

结果：A 已提交，B 处于未知状态 → 数据不一致
```

- 协调者宕机后，参与者处于"悬挂（in-doubt）"状态
- 必须等待协调者恢复才能解决（可能导致长时间阻塞）

**问题 3：网络分区**

```
情景：
1. 协调者发送 Commit 后网络分区
2. 参与者 A 收到 Commit 并提交
3. 参与者 B 超时，不知道是 Commit 还是 Abort
```

- 即使参与者有超时机制，也不知道是否该单方面提交

**问题 4："最后提交者宕机"**

- 协调者 Prepare 阶段收到所有 Yes 后，向第一个参与者发送 Commit
- 第一个参与者提交后，协调者宕机
- 其他参与者不知道整体事务是否提交

#### 3.2.3 2PC 的优化

**预提交日志**：协调者在发送 Commit 前将决定持久化，崩溃恢复后可继续发送。

**超时机制**：参与者等待超时后，可主动询问协调者或其他参与者。但如果无法联系协调者，仍然无法确定（不一致风险）。

**假设提交（Presumed Commit）**：默认所有决定未记录的事务都是提交状态，减少日志写入。

### 3.3 三阶段提交（3PC）

3PC 是对 2PC 的改进，增加了一个 CanCommit 阶段，并引入超时自动提交机制。

#### 3.3.1 三个阶段

**Phase 1：CanCommit**：

- 协调者询问各参与者是否可以执行事务（只是询问，不实际执行）
- 参与者回复 Yes/No（不加锁）

**Phase 2：PreCommit**：

- 所有回复 Yes：协调者发送 PreCommit，参与者执行操作并加锁（写日志）
- 有 No：协调者直接发送 Abort

**Phase 3：DoCommit**：

- 所有参与者确认 PreCommit：协调者发送 DoCommit，参与者最终提交
- 如果参与者超时未收到 DoCommit：**自动提交**（假设协调者已决定提交）

#### 3.3.2 3PC 的改进与局限

**改进**：

- CanCommit 阶段降低了无效等待（快速失败）
- 超时自动提交减少了阻塞时间
- 协调者宕机时，参与者不会永久阻塞

**局限性**：

- 仍无法解决网络分区下的脑裂问题
- 假设场景：
  1. PreCommit 后网络分区
  2. 协调者决定 Abort（因为某个参与者返回 No）
  3. 分区的参与者超时后自动提交
  4. 结果：部分提交，部分回滚 → 数据不一致

3PC 在实践中并不比 2PC 更受欢迎，主要原因是增加了网络往返次数，且仍无法处理分区+超时的极端情况。

### 3.4 Paxos/Raft 与分布式共识

#### 3.4.1 分布式共识与事务的关系

**分布式共识（Consensus）**解决的问题：让多个节点就某个值达成一致（即使有节点故障）。

**Raft/Paxos 解决的是日志复制问题**，而不是跨分区的事务协调问题：

- 单个 Raft Group 内：保证日志的线性一致性（所有副本有相同的日志序列）
- 跨 Raft Group：需要额外的协调协议（如 2PC/Percolator）

#### 3.4.2 Raft + 2PC 的协作

在 TiKV/TinyKV 的架构中，两者协作如下：

```
分布式事务（跨 Region）
        │
        ▼
    2PC 协调（Percolator）
    │           │
    ▼           ▼
Region A     Region B
(Raft Group A) (Raft Group B)
    │           │
    ▼           ▼
  持久化       持久化
（Raft 保证每个 Region 内的强一致性）
```

- **2PC/Percolator** 负责：跨 Region 的原子性协调（Who commits? When?)
- **Raft** 负责：每个 Region 内部的日志复制和持久化（确保单节点的 D 和 C）

这两个机制是互补的，缺一不可。

### 3.5 分布式事务隔离与一致性

#### 3.5.1 外部一致性（External Consistency）

**定义**：如果事务 T1 在事务 T2 开始之前提交，则 T2 必须能看到 T1 的所有修改。更正式地：事务的提交顺序必须与实际时钟顺序一致（即线性化）。

**为什么难以实现**：

- 需要全局精确的时钟
- 分布式节点之间的时钟不可能完全同步

**Spanner 的解决方案（TrueTime）**：

- 使用 GPS 天线 + 原子钟提供有界不确定性的时间（ε 区间）
- 提交时等待 `2ε` 确保时间戳唯一且正确顺序

#### 3.5.2 快照隔离（Snapshot Isolation, SI）

**定义**：每个事务在开始时读取一个一致的数据快照，事务的写操作在提交前对其他事务不可见。

**快照隔离能防止**：脏读、不可重复读、大多数幻读

**快照隔离不能防止**：写偏斜（Write Skew）

Percolator 实现的是**快照隔离**（通过 startTS 实现快照读，通过 commitTS 的可见性控制）。

#### 3.5.3 串行化快照隔离（SSI）

**SSI** 在 SI 的基础上，通过检测**读写冲突环**来识别并阻止写偏斜：

- 如果事务 T1 读了某个数据项，T2 写了该数据项，且 T1 也写了 T2 读的数据项 → 形成冲突环 → 中止一个事务

PostgreSQL 的 SERIALIZABLE 级别就是基于 SSI 实现的。

#### 3.5.4 全局时钟方案对比

| 方案                    | 原理                      | 精度           | 实现难度       |
| ----------------------- | ------------------------- | -------------- | -------------- |
| **TSO（中心化时间戳）** | 专用服务器分发单调递增 ID | 取决于网络延迟 | 低             |
| **HLC（混合逻辑时钟）** | 物理时钟 + 逻辑时钟结合   | 有界误差       | 中             |
| **TrueTime**            | GPS + 原子钟              | ±数毫秒        | 高（专用硬件） |
| **Lamport Clock**       | 纯逻辑时钟                | 仅保证偏序     | 低             |

### 3.6 分布式事务常见协议

#### 3.6.1 XA 协议

XA 是 Open Group 制定的**跨数据库**分布式事务标准，基于 2PC：

```sql
-- 应用程序通过 XA 接口控制多个数据库
XA START 'xid';
  UPDATE accounts SET balance = balance - 100 WHERE id = 1;
XA END 'xid';
XA PREPARE 'xid';    -- Prepare 阶段

XA COMMIT 'xid';     -- 或 XA ROLLBACK 'xid'
```

**特点**：

- 标准化接口，支持多种数据库混用
- 基于 2PC，有协调者单点问题
- 性能较差（同步阻塞、持锁时间长）
- 实际应用：JTA（Java Transaction API）

#### 3.6.2 TCC（Try-Confirm-Cancel）

TCC 是一种**业务层面**的补偿事务协议：

| 阶段        | 操作     | 说明                       |
| ----------- | -------- | -------------------------- |
| **Try**     | 资源预留 | 冻结库存/余额，不实际扣减  |
| **Confirm** | 确认执行 | 正式扣减，释放预留资源     |
| **Cancel**  | 取消执行 | 释放预留资源，回到原始状态 |

**示例（电商下单）**：

```
Try: 冻结库存 1 件，冻结余额 100 元
Confirm: 扣减库存，扣减余额
Cancel: 解冻库存，解冻余额
```

**特点**：

- 侵入性强：需要为每个操作实现 Try/Confirm/Cancel 三个方法
- 没有数据库锁，性能好
- 适合核心业务（订单、支付）

#### 3.6.3 Saga 长事务

Saga 将一个长事务分解为一系列本地事务，每个本地事务都有对应的**补偿事务**：

```
正向流程: T1 → T2 → T3 → ... → Tn
补偿流程: C1 ← C2 ← C3 ← ... ← Cn
```

**失败处理**：

- 如果 Ti 失败，执行 Ci-1, Ci-2, ..., C1 进行补偿回滚
- 最终数据可能经过一段时间才达到一致（最终一致性）

**实现方式**：

- **Orchestration（编排）**：有一个 Saga Orchestrator 协调所有步骤
- **Choreography（编舞）**：每个服务在完成本地事务后发布事件，其他服务监听事件执行

**特点**：

- 适合**长事务**（跨多个微服务、可能持续数分钟）
- 最终一致性，不提供隔离性
- 需要业务上接受"暂时不一致"状态

#### 3.6.4 本地消息表

一种基于消息队列的**最终一致性**方案：

```
1. 本地事务：业务操作 + 写消息到本地消息表（原子操作）
2. 异步服务：扫描本地消息表，发送到消息队列
3. 消费者：消费消息，执行目标操作，幂等处理
4. 确认机制：消费成功后删除/标记消息
```

**特点**：

- 最终一致性，适合异步场景
- 实现简单，依赖消息队列可靠性
- 不适合强一致性要求的场景

#### 3.6.5 各协议对比

| 协议           | 一致性   | 协调者            | 侵入性 | 性能 | 适用场景     |
| -------------- | -------- | ----------------- | ------ | ---- | ------------ |
| **XA/2PC**     | 强一致   | 需要              | 低     | 差   | 少量跨库     |
| **TCC**        | 强一致   | 需要              | 高     | 中   | 金融核心业务 |
| **Percolator** | SI       | 无（Primary Key） | 中     | 中   | 通用 KV 事务 |
| **Saga**       | 最终一致 | 可选              | 高     | 好   | 微服务长事务 |
| **本地消息表** | 最终一致 | 无                | 中     | 好   | 异步通知     |
| **Spanner**    | 外部一致 | TSO（TrueTime）   | 低     | 中   | 全球数据库   |

---

## 4. Percolator 事务模型深度解析

### 4.1 Percolator 背景与设计哲学

#### 4.1.1 论文背景

Google Percolator（2010 年 OSDI 论文《Large-scale Incremental Processing Using Distributed Transactions and Notifications》）最初是为了解决网页索引的增量更新问题：

- **问题**：Google 每天有数十亿个网页更新，用 MapReduce 重新处理整个索引效率太低
- **方案**：只处理更新的部分，需要支持**跨行原子更新**

#### 4.1.2 核心设计哲学

**去中心化**：传统 2PC 有一个独立的协调者节点，是单点故障风险所在。Percolator 通过将事务状态存储在**Primary Key 所在的行**，利用底层存储（BigTable/RocksDB）的单行原子性来替代协调者角色。

**乐观并发控制**：

- Prewrite 阶段检测冲突（乐观地假设没有冲突）
- 发现冲突时回滚重试，而非提前加锁等待

**MVCC 支持快照读**：

- 读操作不加锁，使用快照版本读取
- 写操作通过 lock 防止并发修改

**依赖的底层能力**：

- 单行原子性（对同一 key 的多列读写是原子的）
- 单调递增时间戳服务（TSO）

### 4.2 三列族设计

Percolator 使用三个列族（Column Family）来实现分布式事务：

#### 4.2.1 CfDefault（data 列族）

**存储内容**：用户数据的实际值

**键格式**：`EncodeKey(userKey, startTS)` 即 `(userKey, startTS)`

**值格式**：用户数据的原始字节

**语义**：在 startTS 时刻，某个写事务对 userKey 写入的值。注意是用 startTS 而非 commitTS 作为版本，这样读操作可以通过 Write CF 的 commitTS 找到对应的 startTS，再去 Default CF 中读取。

```
CfDefault:
Key                          Value
(Alice, startTS=100)    ─►   "Bob"       ← 事务 100 写入的值
(Alice, startTS=80)     ─►   "Alice"     ← 事务 80 写入的值
(Bob, startTS=95)       ─►   "100"       ← 事务 95 写入的值
```

#### 4.2.2 CfLock（lock 列族）

**存储内容**：事务在 Prewrite 阶段写入的锁，表示事务正在进行中

**键格式**：`userKey`（不含时间戳，每个 key 同时只能有一个锁）

**值格式**（Lock 结构体）：

```protobuf
message Lock {
    LockType   lock_type   // PUT / DELETE / ROLLBACK / LOCK
    bytes      primary     // Primary Key（用于崩溃恢复时查找事务状态）
    uint64     ts          // startTS（用于区分哪个事务的锁）
    uint64     ttl         // TTL，超时后可强制回滚
    uint64     txn_size    // 事务大小（决定 TTL）
}
```

**语义**：key 上有锁 = 有事务正在修改这个 key，还未提交。

```
CfLock:
Key      Value
Alice ─► Lock{type=PUT, primary="Alice", ts=100, ttl=3000}
Bob   ─► Lock{type=PUT, primary="Alice", ts=100, ttl=3000}
```

注意：同一事务对 Alice 和 Bob 的锁都指向同一个 primary（"Alice"），这样通过任何一个锁都能找到事务的决策点。

#### 4.2.3 CfWrite（write 列族）

**存储内容**：事务提交记录，记录哪个 commitTS 对应哪个 startTS

**键格式**：`EncodeKey(userKey, commitTS)` 即 `(userKey, commitTS)`

**值格式**（Write 结构体）：

```protobuf
message Write {
    WriteKind kind           // PUT / DELETE / ROLLBACK
    uint64    start_ts       // 对应的 startTS（指向 CfDefault 中的值）
}
```

**语义**：在 commitTS 时刻，userKey 的值来自 startTS 时刻写入的 CfDefault 数据。读操作通过扫描 CfWrite 找到最新的可见版本，再通过 startTS 去 CfDefault 取值。

```
CfWrite:
Key                          Value
(Alice, commitTS=105)   ─►   Write{kind=PUT, startTS=100}
(Alice, commitTS=85)    ─►   Write{kind=PUT, startTS=80}
(Bob, commitTS=98)       ─►   Write{kind=PUT, startTS=95}
```

#### 4.2.4 三列族时间线图

```
时间线（数字代表 TS）:

事务 80（已提交）: startTS=80, commitTS=85
事务 100（进行中）: startTS=100, commitTS 待定

CfDefault:
  Alice: TS=80 → "Alice"(旧值),  TS=100 → "Bob"(新值,尚未提交)

CfLock:
  Alice: Lock{ts=100, primary="Alice", ...}  ← Prewrite 写入，Commit 后删除

CfWrite:
  Alice: TS=85 → {PUT, startTS=80}  ← 事务80 Commit 后写入

读事务（startTS=90）读取 Alice:
  1. 查 CfLock[Alice]：有锁 ts=100 > 90，忽略（锁的事务在我的快照之后）
  2. 查 CfWrite[Alice]: 找 commitTS <= 90 的最新记录 → commitTS=85 → startTS=80
  3. 查 CfDefault[Alice, TS=80] → "Alice"  ✓

读事务（startTS=110）读取 Alice:
  1. 查 CfLock[Alice]：有锁 ts=100 <= 110，需要等待或清理
  2. 等待或通过 CheckTxnStatus 判断是否可清理
```

### 4.3 时间戳服务（TSO）

#### 4.3.1 TSO 的作用

**TSO（Timestamp Oracle，时间戳服务）** 是 Percolator 的核心组件之一，提供：

- **全局单调递增的时间戳**：确保每个时间戳唯一且单调递增
- **因果一致性**：如果 T1 在 T2 开始之前提交，则 T1.commitTS < T2.startTS

在 TiDB/TinyKV 中，TSO 由 PD（Placement Driver）提供。

#### 4.3.2 startTS 与 commitTS 的语义

**startTS（事务开始时间戳）**：

- 在事务开始时从 TSO 获取
- 定义了事务的**读快照**：只能看到 commitTS <= startTS 的已提交数据
- 用作 CfDefault 的键（数据版本号）

**commitTS（事务提交时间戳）**：

- 在事务准备提交时从 TSO 获取，commitTS > startTS
- 定义了事务的**提交时序**：其他事务的 startTS > commitTS 才能看到此事务的数据
- 用作 CfWrite 的键（提交版本号）

#### 4.3.3 时间戳保证全局一致快照

假设 T1 提交（commitTS=100），T2 开始（startTS=105）：

- T2 的快照 = 所有 commitTS <= 105 的事务 → T2 能看到 T1 的修改 ✓

假设 T2 开始（startTS=95），T1 提交（commitTS=100）：

- T2 的快照 = 所有 commitTS <= 95 的事务 → T2 看不到 T1 的修改 ✓（快照隔离）

这个保证依赖于 TSO 的**全局单调递增**性质：一旦 T2 获取了 startTS=95，之后 T1 获取的 commitTS 必然 > 95（因为时间戳单调递增）。

### 4.4 Primary Key 机制

#### 4.4.1 为什么需要 Primary Key

传统 2PC 依赖独立的协调者节点来做 Commit/Abort 决策，协调者是单点故障风险。

Percolator 的创新：**利用事务的某一个写 key（Primary Key）来充当协调者**。

关键观察：

- 底层存储（BigTable/RocksDB）支持**单行（单 key）的原子操作**
- 对 Primary Key 的 Commit（写 Write CF + 删 Lock CF）是原子的
- 因此：**Primary Key 的 Write CF 有 Commit 记录 = 事务已提交**
- 任何观察者通过检查 Primary Key 的状态，就能确定整个事务的结果

#### 4.4.2 Primary Key 的工作原理

**事务状态决策点**：

```
事务状态判断（通过 PrimaryKey）:
  - CfWrite[PrimaryKey] 有 WriteKind_Put 记录 → 事务已提交
  - CfWrite[PrimaryKey] 有 WriteKind_Rollback 记录 → 事务已回滚
  - CfLock[PrimaryKey] 存在 → 事务仍在进行（根据 TTL 决定是否清理）
  - CfLock[PrimaryKey] 不存在 且 CfWrite 无记录 → 事务已回滚（锁被清理）
```

**崩溃恢复场景**：

场景 1：Prewrite 完成，Commit 前崩溃

```
CfLock[Alice] 存在（Primary），CfLock[Bob] 存在（Secondary）
→ 检查 CfWrite[Alice]：无记录
→ 检查 Alice.lock 的 TTL：如果超时 → 清理所有锁，视为回滚
→ 如果未超时 → 等待，原事务可能还在执行
```

场景 2：Primary Key Commit 成功，Secondary Key 未 Commit，崩溃

```
CfWrite[Alice] 有 Write{PUT, startTS=100}（Primary 已提交）
CfLock[Bob] 存在（Secondary 未提交）
→ 其他事务遇到 Bob 的锁 → 检查 Primary Key → 已提交
→ 替原事务提交 Bob：DeleteLock[Bob] + PutWrite[Bob, TS=105]
```

#### 4.4.3 Primary Key 的选择

TinyKV/TiKV 中，Primary Key 通常是**第一个写入的 key**（或随机选择）。

Primary Key 的选择会影响热点：如果所有事务都以某个热门 key 作为 Primary，该 key 会成为瓶颈。

### 4.5 Prewrite 阶段详解

Prewrite 是事务的第一阶段，负责：

1. 检测冲突（写冲突 + 锁冲突）
2. 写入数据（CfDefault）
3. 写入锁（CfLock）

#### 4.5.1 完整流程

```
客户端发起 Prewrite(mutations, primaryKey, startTS, ttl):

FOR each mutation in mutations:
    // 步骤1: 检查写冲突
    write, commitTS = MostRecentWrite(mutation.key)
    IF write != nil AND commitTS >= startTS:
        RETURN WriteConflict{commitTS, startTS, key}

    // 步骤2: 检查锁冲突
    lock = GetLock(mutation.key)
    IF lock != nil:
        RETURN KeyError{locked: lock}  // 提示客户端等待或清理

    // 步骤3: 写入数据（值）
    PutValue(mutation.key, mutation.value)

    // 步骤4: 写入锁
    PutLock(mutation.key, Lock{
        kind:    mutation.kind,    // PUT / DELETE
        primary: primaryKey,
        ts:      startTS,
        ttl:     ttl,
    })

// 批量提交到存储层（原子性写入）
storage.Write(txn.Writes())
```

#### 4.5.2 写冲突检测细节

**条件**：存在 `commitTS >= startTS` 的写记录

**语义**：从 startTS 开始，已经有另一个事务在 startTS 之后提交了（commitTS >= startTS），这意味着如果当前事务也提交，两个事务的修改顺序违反了一致性。

```
例：
T1 startTS=100, T2 startTS=90
T2 先提交，commitTS=95
T1 尝试 Prewrite：MostRecentWrite 返回 commitTS=95 >= 100? 不，95 < 100
T1 没有冲突，继续

另一种情况：
T1 startTS=100, T2 startTS=90
T2 先提交，commitTS=105（发生在 T1 开始后！）
T1 尝试 Prewrite：MostRecentWrite 返回 commitTS=105 >= 100？是的
T1 返回 WriteConflict，客户端重试
```

#### 4.5.3 锁冲突检测细节

**条件**：key 上存在任意锁（不论 lockTS 大小）

**语义**：有另一个事务正在修改这个 key，若同时写入可能导致冲突。

客户端收到锁冲突后的处理策略：

1. **等待**：短暂等待后重试（TTL 未超时时）
2. **强制回滚**：通过 CheckTxnStatus 检查锁的状态，若 TTL 超时则协助回滚
3. **推进提交**：若锁的 Primary 已提交，协助完成 Secondary 的提交

### 4.6 Commit 阶段详解

Commit 是事务的第二阶段，将 Prewrite 的"意图"变为"事实"：

```
客户端发起 Commit(keys, startTS, commitTS):

FOR each key in keys:
    // 步骤1: 验证锁还在（防止 Prewrite 被回滚）
    lock = GetLock(key)
    IF lock == nil OR lock.ts != startTS:
        RETURN retryable error  // 锁已被清理，事务已回滚

    // 步骤2: 写提交记录
    PutWrite(key, commitTS, Write{
        kind:     lock.kind,   // PUT / DELETE
        start_ts: startTS,
    })

    // 步骤3: 删除锁（释放）
    DeleteLock(key)

// 批量提交
storage.Write(txn.Writes())
```

**关键点**：Primary Key 的 Commit 是整个事务的"决策点"，Primary Key 的 Write CF 写入成功即代表事务提交，Secondary Keys 的提交可以异步进行。

**幂等性**：如果同一 Commit 请求被重试，通过 `CurrentWrite` 检查发现已有匹配的 Write 记录，直接返回成功。

### 4.7 Read 流程

```
客户端发起 Get(key, version=startTS):

// 步骤1: 检查锁冲突
lock = GetLock(key)
IF lock != nil AND lock.ts <= startTS:
    // 有事务（startTS <= version）正在修改这个 key
    // 需要等待或协助清理
    RETURN KeyError{locked: lock}

// 步骤2: 在 Write CF 中找最新可见提交
Write = GetValue(key)  // 内部实现：Seek CfWrite[key, startTS]

IF write == nil:
    RETURN nil  // key 不存在（在此快照下）

RETURN value  // 通过 write.startTS 从 CfDefault 读取实际值
```

**注意**：只有 `lock.ts <= startTS` 时才阻塞读。如果 `lock.ts > startTS`，说明锁是在当前读快照之后的事务加的，对当前快照不可见，可以忽略。

### 4.8 Rollback 流程

当事务需要回滚时（TTL 超时、手动回滚、崩溃恢复）：

```
Rollback(key, startTS):

// 写 Rollback 标记（重要：即使 CfDefault 无数据也要写）
PutWrite(key, startTS, Write{kind: ROLLBACK, start_ts: startTS})

// 删除 CfDefault 中的临时数据
DeleteValue(key)

// 删除锁（如果锁存在）
lock = GetLock(key)
IF lock != nil AND lock.ts == startTS:
    DeleteLock(key)
```

**为什么必须写 Rollback 标记**：

- 如果不写，后续可能有另一个并发事务也尝试 Rollback，然后提交一个新事务
- 如果这时候原事务的 Commit 消息到来（因为网络延迟），会发现 lock 已不存在，导致不确定状态
- 有了 Rollback 标记：后续 Commit 发现 `CurrentWrite` 已有 Rollback 记录，直接返回错误，防止"已回滚的事务被错误提交"

### 4.9 锁冲突与 Stale Lock 处理

#### 4.9.1 TTL 机制

每个锁都有 TTL（Time To Live），用于检测事务是否"卡死"（崩溃、长期无响应）：

```
TTL = base_ttl + size_factor * txn_size
```

- 大事务（写入数据多）的 TTL 更长，防止误伤
- 小事务 TTL 较短，崩溃后快速清理

#### 4.9.2 CheckTxnStatus 流程

当读操作遇到 Stale Lock（TTL 超时）时，调用 `CheckTxnStatus` 检查事务状态：

```
CheckTxnStatus(primaryKey, lockTS, currentTS):

// 检查事务是否已提交
write, commitTS = CurrentWrite(primaryKey)  // 找匹配 startTS=lockTS 的 write
IF write != nil AND write.kind != ROLLBACK:
    RETURN {action: NoAction, commitTs: commitTS}  // 已提交

// 检查是否已回滚
IF write != nil AND write.kind == ROLLBACK:
    RETURN {action: NoAction, commitTs: 0}  // 已回滚

// 检查锁是否存在
lock = GetLock(primaryKey)
IF lock == nil:
    // 锁不存在且无 Write 记录 → 写 Rollback 标记
    PutWrite(primaryKey, lockTS, Rollback)
    RETURN {action: LockNotExistRollback, commitTs: 0}

// 检查 TTL
IF physicalTime(currentTS) >= physicalTime(lockTS) + lock.ttl:
    // TTL 超时，强制回滚
    Rollback(primaryKey, lockTS)
    RETURN {action: TTLExpireRollback, commitTs: 0}

// TTL 未超时，等待
RETURN {action: NoAction, commitTs: 0, lockTtl: lock.ttl}
```

#### 4.9.3 Resolve Lock 流程

当确定事务结果后（已提交或需要回滚），通过 `ResolveLock` 批量处理该事务的所有锁：

```
ResolveLock(startTS, commitTS):
  // 扫描所有 lockTS == startTS 的锁
  locks = AllLocksForTxn(startTS)

  FOR each lock in locks:
    IF commitTS == 0:
        // 回滚
        DeleteValue(lock.key)
        DeleteLock(lock.key)
        PutWrite(lock.key, startTS, Rollback)
    ELSE:
        // 提交
        DeleteLock(lock.key)
        PutWrite(lock.key, commitTS, Write{PUT, startTS})
```

---

## 5. Project4 代码实现详解

### 5.1 存储层结构与键编码

#### 5.1.1 EncodeKey 实现

```go
// kv/transaction/mvcc/transaction.go
func EncodeKey(key []byte, ts uint64) []byte {
    encodedKey := codec.EncodeBytes(key)
    newKey := append(encodedKey, make([]byte, 8)...)
    binary.BigEndian.PutUint64(newKey[len(encodedKey):], ^ts)  // XOR 取反
    return newKey
}
```

**关键点：时间戳倒序存储（`^ts` = XOR 取反）**

为什么时间戳要倒序？

```
正常存储（升序）: TS=100 < TS=200 < TS=300
倒序存储（降序）: ~TS=100 > ~TS=200 > ~TS=300
               (对应字节值: 大 > 中 > 小)
```

**好处**：

1. **Seek 到最新版本更高效**：在 CfWrite 中 Seek `EncodeKey(key, startTS)` 后，**第一个扫到的**就是 `commitTS <= startTS` 中最大的（因为倒序，大 commitTS 对应小字节值，排在前面）

```
CfWrite（逻辑上按 key ASC, ts DESC 排序）:
  Alice/TS=~300 (commitTS=300)  ← 排最前
  Alice/TS=~200 (commitTS=200)
  Alice/TS=~100 (commitTS=100)

Seek(Alice, startTS=250):
  → 找到第一个 key=Alice 且 ~TS <= ~250 (即 TS >= 250) 的记录
  → 因为倒序，"第一个"就是 commitTS 最大且 <= 250 的记录
  → 找到 Alice/TS=~200 (commitTS=200) ✓
```

2. **与 MVCC 语义自然契合**：读操作总是想要最新的可见版本，倒序存储使 Seek 后第一条就是目标。

#### 5.1.2 DecodeUserKey 和 decodeTimestamp

```go
func DecodeUserKey(key []byte) []byte {
    userKey, _ := codec.DecodeBytes(key)
    return userKey
}

func decodeTimestamp(key []byte) uint64 {
    left, _ := codec.DecodeBytes(key)  // 跳过 user key 部分
    return ^binary.BigEndian.Uint64(key[len(left):])  // XOR 还原
}
```

#### 5.1.3 三个 CF 在 badger 中的存储布局

```
CfDefault:
  [EncodeKey("Alice", 100)] → "Bob"
  [EncodeKey("Alice", 80)]  → "Alice"

CfLock:
  ["Alice"] → Lock{ts=100, primary="Alice", ...}

CfWrite:
  [EncodeKey("Alice", ~105)] → Write{kind=PUT, startTS=100}
  [EncodeKey("Alice", ~85)]  → Write{kind=PUT, startTS=80}
```

### 5.2 MvccTxn 结构体

```go
// kv/transaction/mvcc/transaction.go
type MvccTxn struct {
    StartTS uint64              // 事务的快照版本（startTS）
    Reader  storage.StorageReader  // 存储读接口（不可变）
    writes  []storage.Modify    // 写操作缓冲（Commit 时批量提交）
}
```

**设计要点**：

1. **写缓冲（writes slice）**：所有的 PutLock、PutValue、PutWrite、DeleteLock、DeleteValue 都只是追加到 `writes` 缓冲中，不立即写存储。只有调用 `storage.Write(txn.Writes())` 时才原子地提交所有修改。

   这保证了：

   - 单个 RPC（如 KvPrewrite）的多个写操作原子性
   - 写失败时不会留下部分状态

2. **Reader 不可变**：Reader 是只读的存储快照，通过 `Reader.GetCF(cf, key)` 和 `Reader.IterCF(cf)` 读取数据。

### 5.3 Part A：核心 MVCC 方法

#### 5.3.1 GetLock 和 PutLock

```go
func (txn *MvccTxn) GetLock(key []byte) (*Lock, error) {
    // 直接读 CfLock，无需时间戳编码
    value, err := txn.Reader.GetCF(engine_util.CfLock, key)
    if err != nil || value == nil {
        return nil, err
    }
    lock, err := ParseLock(value)  // 反序列化 Lock 结构体
    return lock, err
}

func (txn *MvccTxn) PutLock(key []byte, lock *Lock) {
    txn.writes = append(txn.writes, storage.Modify{
        Data: storage.Put{
            Key:   key,
            Value: lock.ToBytes(),  // 序列化
            Cf:    engine_util.CfLock,
        },
    })
}
```

**注意**：CfLock 的键是原始 userKey，不含时间戳。每个 key 同时只能有一个锁。

#### 5.3.2 GetValue 实现（关键）

```go
func (txn *MvccTxn) GetValue(key []byte) ([]byte, error) {
    // 在 CfWrite 中找 commitTS <= startTS 的最新 Write
    iter := txn.Reader.IterCF(engine_util.CfWrite)
    defer iter.Close()

    // Seek 到 EncodeKey(key, startTS) 处
    // 因为时间戳倒序，第一条 >= startTS 的就是我们想要的
    iter.Seek(EncodeKey(key, txn.StartTS))

    if !iter.Valid() {
        return nil, nil
    }

    item := iter.Item()
    userKey := DecodeUserKey(item.Key())

    // 验证是同一个 user key（防止跨 key）
    if !bytes.Equal(userKey, key) {
        return nil, nil
    }

    // 反序列化 Write 记录
    value, err := item.Value()
    write, err := ParseWrite(value)

    // 如果是 DELETE 或 ROLLBACK，说明此版本不是有效值
    if write.Kind != WriteKindPut {
        return nil, nil
    }

    // 通过 write.StartTS 从 CfDefault 取实际值
    return txn.Reader.GetCF(engine_util.CfDefault, EncodeKey(key, write.StartTs))
}
```

**关键实现细节**：

- Seek 后需要验证 `userKey == key`，防止 Seek 越过了目标 key（扫到了下一个 key）
- 必须检查 `write.Kind != WriteKindPut`：若是 DELETE 或 ROLLBACK，说明该版本数据已被删除，应返回 nil

#### 5.3.3 CurrentWrite 实现（关键）

```go
func (txn *MvccTxn) CurrentWrite(key []byte) (*Write, uint64, error) {
    // 从最大时间戳向小扫描，找匹配 startTS 的 Write
    iter := txn.Reader.IterCF(engine_util.CfWrite)
    defer iter.Close()

    // 从 TsMax 开始扫描（相当于最大 commitTS）
    for iter.Seek(EncodeKey(key, TsMax)); iter.Valid(); iter.Next() {
        item := iter.Item()
        userKey := DecodeUserKey(item.Key())

        if !bytes.Equal(userKey, key) {
            break  // 已扫过此 key 的所有版本
        }

        value, _ := item.Value()
        write, _ := ParseWrite(value)

        if write.StartTs == txn.StartTS {
            commitTS := decodeTimestamp(item.Key())
            return write, commitTS, nil
        }
    }
    return nil, 0, nil
}
```

**作用**：

- 在 KvCommit 中检查幂等性（已提交则直接成功）
- 在 KvBatchRollback 中检查是否已处理（已 Rollback 则跳过）
- 在 KvCheckTxnStatus 中确认事务状态

**实现细节**：从 TsMax 向小扫描，这样能遍历所有 commitTS，找到 startTS 匹配的记录。

#### 5.3.4 MostRecentWrite 实现

```go
func (txn *MvccTxn) MostRecentWrite(key []byte) (*Write, uint64, error) {
    iter := txn.Reader.IterCF(engine_util.CfWrite)
    defer iter.Close()

    iter.Seek(EncodeKey(key, TsMax))

    if !iter.Valid() {
        return nil, 0, nil
    }

    item := iter.Item()
    userKey := DecodeUserKey(item.Key())
    if !bytes.Equal(userKey, key) {
        return nil, 0, nil
    }

    value, _ := item.Value()
    write, _ := ParseWrite(value)
    commitTS := decodeTimestamp(item.Key())
    return write, commitTS, nil
}
```

**作用**：在 KvPrewrite 中检测写冲突（若 commitTS >= startTS 则冲突）。

### 5.4 Scanner 实现

```go
// kv/transaction/mvcc/scanner.go
type Scanner struct {
    iter engine_util.DBIterator  // CfWrite 的迭代器
    txn  *MvccTxn
    key  []byte  // 上一次返回的 user key（用于跳过同 key 的旧版本）
}

func NewScanner(startKey []byte, txn *MvccTxn) *Scanner {
    scanner := &Scanner{
        iter: txn.Reader.IterCF(engine_util.CfWrite),
        txn:  txn,
    }
    // Seek 到 startKey 的最新版本
    scanner.iter.Seek(EncodeKey(startKey, txn.StartTS))
    return scanner
}

func (scan *Scanner) Next() ([]byte, []byte, error) {
    for {
        if !scan.iter.Valid() {
            return nil, nil, nil  // 已遍历完
        }

        item := scan.iter.Item()
        currentKey := DecodeUserKey(item.Key())

        // 跳过与上次相同的 user key（同一 key 的旧版本）
        if bytes.Equal(currentKey, scan.key) {
            scan.iter.Next()
            continue
        }

        // 检查此版本是否在快照范围内（commitTS <= startTS）
        commitTS := decodeTimestamp(item.Key())
        if commitTS > scan.txn.StartTS {
            scan.iter.Next()
            continue
        }

        scan.key = currentKey

        // 获取值（内部处理 DELETE/ROLLBACK 返回 nil）
        value, err := scan.txn.GetValue(currentKey)
        scan.iter.Next()

        if value == nil {
            continue  // 此 key 已被删除或不可见，继续扫下一个
        }

        return currentKey, value, err
    }
}
```

**设计要点**：

- Scanner 维护 `key` 字段：记录上一次返回的 user key，用于跳过同一 key 的多个版本（只取最新可见版本）
- 每次 `Next()` 后移动迭代器，确保不会重复返回同一个 key
- `GetValue` 内部已处理 DELETE/ROLLBACK 情况，返回 nil 时 Scanner 继续跳过

### 5.5 Part B：核心 RPC 处理器

#### 5.5.1 KvGet 实现

```go
// kv/server/server.go
func (server *Server) KvGet(_ context.Context, req *kvrpcpb.GetRequest) (*kvrpcpb.GetResponse, error) {
    reader, _ := server.storage.Reader(req.Context)
    defer reader.Close()

    txn := mvcc.NewMvccTxn(reader, req.Version)

    // 检查锁冲突
    lock, _ := txn.GetLock(req.Key)
    if lock != nil && lock.Ts <= req.Version {
        // 有事务（startTS <= version）持有此 key 的锁，需要等待
        return &kvrpcpb.GetResponse{
            Error: &kvrpcpb.KeyError{
                Locked: lock.Info(req.Key),
            },
        }, nil
    }

    // 读取值
    value, _ := txn.GetValue(req.Key)
    if value == nil {
        return &kvrpcpb.GetResponse{NotFound: true}, nil
    }
    return &kvrpcpb.GetResponse{Value: value}, nil
}
```

**注意**：只有 `lock.Ts <= req.Version` 时才阻塞读。如果锁的 startTS > 版本，说明锁是在快照之后加的，读操作可以忽略。

#### 5.5.2 KvPrewrite 实现

```go
func (server *Server) KvPrewrite(_ context.Context, req *kvrpcpb.PrewriteRequest) (*kvrpcpb.PrewriteResponse, error) {
    reader, _ := server.storage.Reader(req.Context)
    defer reader.Close()

    txn := mvcc.NewMvccTxn(reader, req.StartVersion)
    var keyErrors []*kvrpcpb.KeyError

    for _, mutation := range req.Mutations {
        // 1. 写冲突检测
        write, commitTS, _ := txn.MostRecentWrite(mutation.Key)
        if write != nil && commitTS >= req.StartVersion {
            keyErrors = append(keyErrors, &kvrpcpb.KeyError{
                Conflict: &kvrpcpb.WriteConflict{
                    StartTs:    req.StartVersion,
                    ConflictTs: commitTS,
                    Key:        mutation.Key,
                    Primary:    req.PrimaryLock,
                },
            })
            continue
        }

        // 2. 锁冲突检测
        lock, _ := txn.GetLock(mutation.Key)
        if lock != nil {
            keyErrors = append(keyErrors, &kvrpcpb.KeyError{
                Locked: lock.Info(mutation.Key),
            })
            continue
        }

        // 3. 写入数据和锁
        switch mutation.Op {
        case kvrpcpb.Op_Put:
            txn.PutValue(mutation.Key, mutation.Value)
            txn.PutLock(mutation.Key, &mvcc.Lock{
                Primary: req.PrimaryLock,
                Ts:      req.StartVersion,
                Ttl:     req.LockTtl,
                Kind:    mvcc.WriteKindPut,
            })
        case kvrpcpb.Op_Del:
            txn.DeleteValue(mutation.Key)
            txn.PutLock(mutation.Key, &mvcc.Lock{
                Primary: req.PrimaryLock,
                Ts:      req.StartVersion,
                Ttl:     req.LockTtl,
                Kind:    mvcc.WriteKindDelete,
            })
        }
    }

    if len(keyErrors) > 0 {
        return &kvrpcpb.PrewriteResponse{Errors: keyErrors}, nil
    }

    // 批量原子提交
    server.storage.Write(req.Context, txn.Writes())
    return &kvrpcpb.PrewriteResponse{}, nil
}
```

#### 5.5.3 KvCommit 实现

```go
func (server *Server) KvCommit(_ context.Context, req *kvrpcpb.CommitRequest) (*kvrpcpb.CommitResponse, error) {
    reader, _ := server.storage.Reader(req.Context)
    defer reader.Close()

    txn := mvcc.NewMvccTxn(reader, req.StartVersion)

    // Latch 防止并发修改同一 key
    server.Latches.WaitForLatches(req.Keys)
    defer server.Latches.ReleaseLatches(req.Keys)

    for _, key := range req.Keys {
        // 检查幂等性：是否已提交
        write, _, _ := txn.CurrentWrite(key)
        if write != nil {
            if write.Kind == mvcc.WriteKindRollback {
                // 已回滚，不能再提交
                return &kvrpcpb.CommitResponse{
                    Error: &kvrpcpb.KeyError{Retryable: "already rolled back"},
                }, nil
            }
            // 已提交（幂等），继续下一个 key
            continue
        }

        // 验证锁
        lock, _ := txn.GetLock(key)
        if lock == nil || lock.Ts != req.StartVersion {
            // 锁不存在或 ts 不匹配（可能被其他事务抢走了）
            return &kvrpcpb.CommitResponse{
                Error: &kvrpcpb.KeyError{Retryable: "lock not found"},
            }, nil
        }

        // 写提交记录
        txn.PutWrite(key, req.CommitVersion, &mvcc.Write{
            StartTs: req.StartVersion,
            Kind:    lock.Kind,
        })

        // 删除锁
        txn.DeleteLock(key)
    }

    server.storage.Write(req.Context, txn.Writes())
    return &kvrpcpb.CommitResponse{}, nil
}
```

### 5.6 Part C：辅助 RPC 处理器

#### 5.6.1 KvScan 实现

```go
func (server *Server) KvScan(_ context.Context, req *kvrpcpb.ScanRequest) (*kvrpcpb.ScanResponse, error) {
    reader, _ := server.storage.Reader(req.Context)
    defer reader.Close()

    txn := mvcc.NewMvccTxn(reader, req.Version)
    scanner := mvcc.NewScanner(req.StartKey, txn)
    defer scanner.Close()

    var pairs []*kvrpcpb.KvPair

    for i := uint32(0); i < req.Limit; i++ {
        key, value, err := scanner.Next()
        if key == nil {
            break
        }

        if err != nil {
            // 处理 lock 错误：记录到 pairs，而非终止
            if keyErr, ok := err.(*mvcc.KeyError); ok {
                pairs = append(pairs, &kvrpcpb.KvPair{
                    Error: keyErr.Err,
                })
                continue
            }
            return nil, err
        }

        pairs = append(pairs, &kvrpcpb.KvPair{
            Key:   key,
            Value: value,
        })
    }

    return &kvrpcpb.ScanResponse{Pairs: pairs}, nil
}
```

**注意**：KvScan 遇到 lock 错误时，应将错误记录到 pairs 中继续扫描，而不是直接返回错误中止整个扫描。这让客户端知道哪些 key 有锁冲突，可以选择性地处理。

#### 5.6.2 KvCheckTxnStatus 实现

```go
func (server *Server) KvCheckTxnStatus(_ context.Context, req *kvrpcpb.CheckTxnStatusRequest) (
    *kvrpcpb.CheckTxnStatusResponse, error) {

    reader, _ := server.storage.Reader(req.Context)
    defer reader.Close()

    txn := mvcc.NewMvccTxn(reader, req.LockTs)

    // 1. 检查是否已提交
    write, commitTS, _ := txn.CurrentWrite(req.PrimaryKey)
    if write != nil && write.Kind != mvcc.WriteKindRollback {
        return &kvrpcpb.CheckTxnStatusResponse{
            CommitVersion: commitTS,
            Action:        kvrpcpb.Action_NoAction,
        }, nil
    }

    // 2. 检查是否已回滚
    if write != nil && write.Kind == mvcc.WriteKindRollback {
        return &kvrpcpb.CheckTxnStatusResponse{
            Action: kvrpcpb.Action_NoAction,
        }, nil
    }

    // 3. 检查锁是否存在
    lock, _ := txn.GetLock(req.PrimaryKey)
    if lock == nil {
        // 锁不存在，写 Rollback 标记
        txn.DeleteValue(req.PrimaryKey)
        txn.PutWrite(req.PrimaryKey, req.LockTs, &mvcc.Write{
            StartTs: req.LockTs,
            Kind:    mvcc.WriteKindRollback,
        })
        server.storage.Write(req.Context, txn.Writes())
        return &kvrpcpb.CheckTxnStatusResponse{
            Action: kvrpcpb.Action_LockNotExistRollback,
        }, nil
    }

    // 4. 检查 TTL
    currentPhysical := mvcc.PhysicalTime(req.CurrentTs)
    lockPhysical := mvcc.PhysicalTime(req.LockTs)
    if currentPhysical >= lockPhysical+lock.Ttl {
        // TTL 超时，强制回滚
        txn.DeleteValue(req.PrimaryKey)
        txn.DeleteLock(req.PrimaryKey)
        txn.PutWrite(req.PrimaryKey, req.LockTs, &mvcc.Write{
            StartTs: req.LockTs,
            Kind:    mvcc.WriteKindRollback,
        })
        server.storage.Write(req.Context, txn.Writes())
        return &kvrpcpb.CheckTxnStatusResponse{
            Action: kvrpcpb.Action_TTLExpireRollback,
        }, nil
    }

    // 5. TTL 未超时，返回锁信息
    return &kvrpcpb.CheckTxnStatusResponse{
        LockTtl: lock.Ttl,
        Action:  kvrpcpb.Action_NoAction,
    }, nil
}
```

#### 5.6.3 KvBatchRollback 实现

```go
func (server *Server) KvBatchRollback(_ context.Context, req *kvrpcpb.BatchRollbackRequest) (
    *kvrpcpb.BatchRollbackResponse, error) {

    reader, _ := server.storage.Reader(req.Context)
    defer reader.Close()

    txn := mvcc.NewMvccTxn(reader, req.StartVersion)

    for _, key := range req.Keys {
        // 检查是否已处理
        write, _, _ := txn.CurrentWrite(key)
        if write != nil {
            if write.Kind == mvcc.WriteKindRollback {
                continue  // 已回滚，幂等跳过
            }
            // 已提交，回滚失败
            return &kvrpcpb.BatchRollbackResponse{
                Error: &kvrpcpb.KeyError{Abort: "already committed"},
            }, nil
        }

        // 删除锁（如果存在且属于此事务）
        lock, _ := txn.GetLock(key)
        if lock != nil && lock.Ts == req.StartVersion {
            txn.DeleteLock(key)
        }

        // 删除数据，写 Rollback 标记
        txn.DeleteValue(key)
        txn.PutWrite(key, req.StartVersion, &mvcc.Write{
            StartTs: req.StartVersion,
            Kind:    mvcc.WriteKindRollback,
        })
    }

    server.storage.Write(req.Context, txn.Writes())
    return &kvrpcpb.BatchRollbackResponse{}, nil
}
```

#### 5.6.4 KvResolveLock 实现

```go
func (server *Server) KvResolveLock(_ context.Context, req *kvrpcpb.ResolveLockRequest) (
    *kvrpcpb.ResolveLockResponse, error) {

    reader, _ := server.storage.Reader(req.Context)
    defer reader.Close()

    txn := mvcc.NewMvccTxn(reader, req.StartVersion)

    // 获取所有属于此事务的锁
    locks, _ := mvcc.AllLocksForTxn(txn)

    for _, lockInfo := range locks {
        txn2 := mvcc.NewMvccTxn(reader, req.StartVersion)

        if req.CommitVersion == 0 {
            // 回滚
            txn2.DeleteValue(lockInfo.Key)
            txn2.DeleteLock(lockInfo.Key)
            txn2.PutWrite(lockInfo.Key, req.StartVersion, &mvcc.Write{
                StartTs: req.StartVersion,
                Kind:    mvcc.WriteKindRollback,
            })
        } else {
            // 提交
            txn2.DeleteLock(lockInfo.Key)
            txn2.PutWrite(lockInfo.Key, req.CommitVersion, &mvcc.Write{
                StartTs: req.StartVersion,
                Kind:    lockInfo.Lock.Kind,
            })
        }

        server.storage.Write(req.Context, txn2.Writes())
    }

    return &kvrpcpb.ResolveLockResponse{}, nil
}
```

### 5.7 Latches 并发控制

```go
// 在 KvCommit 中使用 Latches
server.Latches.WaitForLatches(req.Keys)
defer server.Latches.ReleaseLatches(req.Keys)
```

**Latches 的作用**：

- `Latches` 是一个**内存级别的行锁**（不是分布式锁）
- 作用范围：同一个 TinyKV 节点上的并发请求
- 防止问题：同一节点上两个并发 Commit 请求操作同一个 key 时，可能产生竞争条件

**与 MVCC 的关系**：

- **MVCC** 解决的是多版本读写并发问题（通过时间戳实现快照隔离）
- **Latches** 解决的是同一节点上的写-写并发问题（防止内存状态竞争）
- 两者互补：MVCC 是逻辑层面的并发控制，Latches 是物理层面的并发保护

### 5.8 实现注意事项与坑

#### 5.8.1 GetValue 的 WriteKind 检查

```go
// 错误做法：不检查 WriteKind
if write != nil {
    return txn.Reader.GetCF(CfDefault, EncodeKey(key, write.StartTs))
}

// 正确做法：必须检查 WriteKind
if write.Kind != WriteKindPut {
    return nil, nil  // DELETE 或 ROLLBACK 不返回值
}
```

**原因**：DELETE 操作在 Write CF 中写的是 `WriteKindDelete`，如果不检查，会尝试从 Default CF 读值，但 Default CF 中可能没有（或读到旧版本的值），导致数据语义错误。

#### 5.8.2 KvCommit 的 Retryable 错误

```
锁不存在（lock == nil）：
  可能原因：锁已被 Rollback 清理（TTL 超时被其他事务清理）
  正确行为：返回 Retryable Error，客户端应该重新执行整个事务

锁 ts 不匹配（lock.ts != startVersion）：
  可能原因：有另一个事务在同一 key 上加了新锁（说明当前事务的锁已被清理）
  正确行为：返回 Retryable Error
```

**Retryable vs Abort**：

- `Retryable`：客户端可以重试（如重新获取 startTS，重新执行事务）
- `Abort`：事务无法继续（如尝试提交已回滚的事务）

#### 5.8.3 KvScan 中 lock 错误的处理

```go
// 错误做法：遇到 lock 错误直接返回
if err != nil {
    return nil, err
}

// 正确做法：将 lock 错误加入 pairs 继续扫描
pairs = append(pairs, &kvrpcpb.KvPair{Error: keyErr.Err})
continue
```

**原因**：客户端期望获取尽可能多的结果，对于有锁冲突的 key，记录错误信息让客户端决定如何处理，而不是直接终止整个扫描。

#### 5.8.4 KvCheckTxnStatus 无锁时的处理

```go
// 无锁时，必须写 Rollback 标记，即使 Default CF 中没有数据
txn.DeleteValue(req.PrimaryKey)   // 即使不存在也无害
txn.PutWrite(req.PrimaryKey, req.LockTs, Rollback)
```

**原因**：不写 Rollback 标记的风险 ——

- 时序：T1 的锁因网络问题消失 → CheckTxnStatus 认为已回滚，但未写标记
- 后来 T1 的 Prewrite 消息迟到 → 锁又被加上
- 或 T1 的 Commit 消息迟到 → 没有 Rollback 标记，Commit 成功！
- 数据不一致

写了 Rollback 标记：Commit 时 `CurrentWrite` 发现 Rollback 记录，拒绝提交 ✓

#### 5.8.5 Scanner 跳过旧版本

```go
// 维护 scan.key，跳过相同 user key 的多个版本
if bytes.Equal(currentKey, scan.key) {
    scan.iter.Next()
    continue
}
scan.key = currentKey
```

如果不跳过：同一个 key 的多个历史版本都会被返回，客户端会看到重复 key，违反 Scan 语义。

---

## 6. 工业界分布式事务方案对比

### 6.1 TiDB/TiKV 事务演进

TiKV 是 TinyKV 的工业原型，其事务方案也基于 Percolator，但在工程实践中进行了多项优化。

#### 6.1.1 基础方案：Percolator + TSO

TiKV 的基础事务方案与 TinyKV 实现基本一致：

- **TSO** 由 PD（Placement Driver）提供全局单调递增时间戳
- **两阶段提交**：Prewrite 阶段写锁和数据，Commit 阶段写提交记录并删锁
- **Primary Key** 作为事务的状态决策点
- **快照隔离（SI）**：通过 startTS 实现一致性快照读

每次事务需要**两次 TSO 请求**（获取 startTS 和 commitTS）以及**两次存储写入**（Prewrite 和 Commit），这在高并发场景下成为性能瓶颈。

#### 6.1.2 Async Commit（异步提交）

**动机**：标准 2PC 的 Commit 阶段需要客户端等待所有 key 的写操作完成，延迟较高。

**核心思想**：Prewrite 完成后立即返回给客户端（不等待 Commit），Commit 阶段在后台异步完成。

**实现关键**：

- Prewrite 时记录所有 Secondary Key 的位置（`secondaries` 列表）
- 计算 `minCommitTS`：所有 Secondary Key 所在 Region 的 maxTS 中取最大值，确保 commitTS > 所有可能的读事务的 startTS
- Primary Key 的 lock 中记录 `secondaries` 和 `minCommitTS`
- 后台异步提交所有 key

**好处**：客户端在 Prewrite 完成后即可继续，减少了一个网络往返延迟。

**代价**：

- Prewrite 阶段需要额外收集 `minCommitTS`（一次额外的读操作）
- 崩溃恢复更复杂：需要检查所有 Secondary Key 状态来计算 commitTS
- 不适合大事务（Secondary Key 列表很大）

#### 6.1.3 1PC（One-Phase Commit）

**适用条件**：事务的所有写操作都在**同一个 Region** 内（单分片事务）。

**原理**：单 Region 内的多个写操作可以通过 Raft 一次提交（Raft 本身保证原子性），无需两阶段提交。

**实现**：

- 检测到单 Region 事务后，跳过 Prewrite 阶段
- 直接写 Write CF 和 Data CF，不写 Lock CF
- 一次 Raft 提交完成所有操作

**好处**：消除 2PC 开销，延迟降低约 50%，吞吐量显著提升。

**局限**：仅适用于单 Region 事务，跨 Region 仍需 2PC。

#### 6.1.4 Pipelined Locking（流水线加锁）

**动机**：大事务（写入数千个 key）的 Prewrite 阶段需要串行等待所有 key 的锁写入确认，延迟随事务规模线性增长。

**原理**：不等待前一个 key 的锁写入确认，立即开始下一个 key 的锁写入（流水线化）。

**好处**：减少大事务的端到端延迟。

**代价**：更复杂的错误处理（部分锁写入失败时回滚更复杂）。

### 6.2 Google Spanner

#### 6.2.1 TrueTime API

Spanner 的核心创新是 **TrueTime**，一套基于 GPS 天线和原子钟的时钟服务：

```
TrueTime 返回的不是单一时间点，而是一个区间：
TT.now() = [earliest, latest]
其中 |latest - earliest| ≤ 2ε（ε 通常为 1-7 毫秒）
```

时间戳保证：

- 如果事件 A 在事件 B 开始之前结束，则 `TT.after(B.start) > TT.before(A.end)`
- 即 B 的时间戳严格大于 A 的时间戳

#### 6.2.2 Commit-Wait 机制

Spanner 的提交等待机制确保外部一致性：

```
提交流程：
1. 获取提交时间戳 s = TT.now().latest
2. 执行 Prepare（等待所有参与者就绪）
3. 等待（commit-wait）：等到 TT.now().earliest > s（即确保 s 已过去）
4. 执行 Commit
```

**意义**：等待 `2ε` 时间后，整个地球上所有节点的时钟一定都已经超过了 s，保证了外部一致性（后续任何事务的 startTS > s）。

#### 6.2.3 Spanner vs Percolator

| 对比项     | Spanner                | Percolator/TiKV         |
| ---------- | ---------------------- | ----------------------- |
| 时钟方案   | TrueTime（GPS+原子钟） | TSO（中心化时间戳服务） |
| 一致性级别 | 外部一致性             | 快照隔离（SI）          |
| 协调者     | 有（基于 Paxos Group） | 无（Primary Key）       |
| 硬件依赖   | 专用 GPS/原子钟        | 无特殊硬件              |
| 跨地域延迟 | commit-wait 增加延迟   | TSO 往返延迟            |
| 适用场景   | 全球分布式数据库       | 通用分布式数据库        |

### 6.3 CockroachDB

#### 6.3.1 混合逻辑时钟（HLC）

**问题**：CockroachDB 希望无需 GPS/原子钟，也不依赖中心化 TSO，但仍要保证全局时序。

**HLC（Hybrid Logical Clock）**：将物理时钟（NTP 同步）与逻辑时钟结合：

```
HLC 时钟 = (physical, logical)
规则：
  发送消息时：HLC = max(本地HLC, 消息HLC) + 1
  本地事件时：HLC = max(本地HLC, NTP时钟) + 1
```

**特性**：

- 物理分量接近 NTP 时间（有界误差）
- 逻辑分量处理同一物理时刻的多个事件
- 不需要专用硬件，不需要中心化 TSO

#### 6.3.2 事务不确定区间

由于时钟不确定性，CockroachDB 在读操作中使用**不确定区间（uncertainty window）**：

```
如果读事务的 maxTS = readTS + uncertaintyInterval
遇到 commitTS ∈ (readTS, maxTS) 的写操作 → 不确定，需要重启事务
```

这保证了即使时钟略有偏差，也不会读到"未来"写的数据。

#### 6.3.3 Transaction Record

CockroachDB 中的 **Transaction Record** 类似 Percolator 的 Primary Key：

- 存储事务的最终状态（PENDING / COMMITTED / ABORTED）
- 崩溃恢复时通过 Transaction Record 确定事务结果
- 与数据存储在同一节点（减少网络往返）

#### 6.3.4 Contention Resolution（冲突解决）

CockroachDB 遇到锁冲突时，不是简单等待，而是尝试**推进（push）**持锁事务：

```
如果高优先级事务 T2 遇到低优先级事务 T1 的锁：
  T2 可以强制推进 T1：
  - 如果 T1 的优先级较低 → T1 的 commitTS 被推高（晚于 T2）
  - T2 继续执行，T1 在提交时需要重新校验
```

这避免了长时间锁等待，提高了高优先级事务的响应性。

### 6.4 MySQL XA

#### 6.4.1 XA 标准接口

XA 是 The Open Group 制定的分布式事务标准，MySQL InnoDB 实现了 XA 接口：

```sql
-- 开始 XA 事务
XA START 'transaction_id';
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
XA END 'transaction_id';

-- Prepare 阶段
XA PREPARE 'transaction_id';

-- Commit 或 Rollback
XA COMMIT 'transaction_id';
-- 或
XA ROLLBACK 'transaction_id';
```

**查询未完成的 XA 事务**：

```sql
XA RECOVER;  -- 返回所有 PREPARED 状态的事务
```

#### 6.4.2 XA 的限制

- **性能差**：每个 XA 事务需要两次磁盘 fsync（prepare + commit），持锁时间长
- **协调者单点**：应用程序作为协调者，崩溃后 PREPARED 的事务需要手动处理
- **实现不完整**：早期 MySQL XA 实现存在 binlog 一致性问题（已在 5.7 修复）
- **适用场景**：跨数据库实例的少量事务，不适合高吞吐场景

### 6.5 Saga 长事务方案

#### 6.5.1 Saga 的设计动机

在微服务架构中，一个业务流程可能涉及多个服务（订单服务、库存服务、支付服务），且每个服务拥有独立的数据库。传统 2PC 在此场景下的问题：

- 需要跨服务加锁，锁持有时间可能很长（分钟级）
- 服务不同技术栈，XA 支持困难
- 2PC 的阻塞问题在网络较慢的微服务场景下更加严重

Saga 的思路：接受最终一致性，通过补偿事务实现回滚。

#### 6.5.2 Saga 实现示例

**电商下单 Saga**：

```
正向事务序列：
T1: 创建订单（订单服务）
T2: 扣减库存（库存服务）
T3: 扣款（支付服务）
T4: 更新积分（积分服务）

补偿事务序列（逆序执行）：
C4: 撤销积分更新
C3: 退款
C2: 恢复库存
C1: 取消订单

失败场景（T3 失败）：
T1 → T2 → T3（失败）→ C2 → C1
```

#### 6.5.3 Orchestration vs Choreography

**Orchestration（编排）**：

- 有一个 Saga Orchestrator（如 workflow engine）
- Orchestrator 按顺序调用各服务，处理失败和补偿
- 优点：流程集中，易于监控和调试
- 缺点：Orchestrator 是单点，流程耦合

**Choreography（编舞）**：

- 每个服务完成本地事务后发布领域事件
- 其他服务监听事件并响应
- 优点：服务解耦，高可用
- 缺点：流程分散，难以追踪和调试

#### 6.5.4 Saga 的局限

- **无隔离性**：中间状态对外可见，可能导致"脏读"（其他事务看到未完成的 Saga）
- **补偿逻辑复杂**：每个操作都需要实现补偿，业务代码量倍增
- **补偿不总能实现**：如果操作不可逆（已发货），补偿只能是近似恢复

### 6.6 综合对比表

| 方案            | 一致性         | 隔离性                   | 协调者            | 侵入性 | 性能                      | 适用场景         |
| --------------- | -------------- | ------------------------ | ----------------- | ------ | ------------------------- | ---------------- |
| **XA/2PC**      | 强一致         | 可串行化                 | 有（单点）        | 低     | 差                        | 少量跨库操作     |
| **Percolator**  | SI（快照隔离） | SI                       | 无（Primary Key） | 中     | 中                        | 通用 KV 事务     |
| **Spanner**     | 外部一致       | 可串行化                 | 有（Paxos）       | 低     | 中（受 commit-wait 影响） | 全球数据库       |
| **CockroachDB** | 可串行化       | 可串行化                 | 无（HLC）         | 低     | 中                        | 多云/无 GPS 环境 |
| **TCC**         | 强一致         | 最终一致（中间状态存在） | 有                | 极高   | 好                        | 金融核心交易     |
| **Saga**        | 最终一致       | 无                       | 可选              | 高     | 好                        | 微服务长事务     |
| **本地消息表**  | 最终一致       | 无                       | 无                | 中     | 好                        | 异步通知场景     |

---

## 7. 现有实现的问题与优化空间

### 7.1 TinyKV 实现的性能局限

#### 7.1.1 两次 TSO 请求的延迟

标准 Percolator 需要两次从 TSO 获取时间戳：

- Prewrite 前获取 `startTS`
- Commit 前获取 `commitTS`

每次 TSO 请求都是一次网络往返（取决于客户端到 PD 的延迟），在高频小事务场景下，这两次额外的网络延迟显著影响吞吐量。

**优化方向**：

- Async Commit：减少一次 TSO 请求（commitTS 由 minCommitTS 计算）
- TSO 批处理：客户端批量预取时间戳，摊薄网络开销

#### 7.1.2 两次写入放大

每个 key 的写入涉及：

1. Prewrite：写 CfDefault + 写 CfLock（2 次写）
2. Commit：写 CfWrite + 删 CfLock（1写1删）

总计：每个 key 4 次存储操作（相比单机事务的 1 次写 + 1 次 WAL）。

**优化方向**：

- 1PC：对单 Region 事务，跳过 CfLock，直接写 CfWrite + CfDefault（减少 2 次操作）
- 合并小写入：在存储层批量合并写操作

#### 7.1.3 锁等待与重试开销

Percolator 是乐观并发控制，遇到冲突时需要回滚整个事务并重试：

- 重试包含重新获取 startTS、重新 Prewrite 所有 key
- 在高冲突场景下，重试次数可能很多（称为"活锁"问题）

**优化方向**：

- 死锁检测：主动检测死锁并强制中止优先级低的事务
- 优先级调度：高优先级事务优先获取锁（类似 CockroachDB 的 Contention Resolution）

### 7.2 TinyKV 功能局限

#### 7.2.1 无 Async Commit 支持

TinyKV 实现中，所有事务必须完整执行两阶段提交：

```
客户端等待 Prewrite 完成 → 等待 Commit 完成 → 返回结果
```

工业级 TiKV 中，Async Commit 允许：

```
客户端等待 Prewrite 完成 → 立即返回（Commit 在后台异步进行）
```

这减少了客户端感知的延迟，但 TinyKV 为简化实现未包含此功能。

#### 7.2.2 无 1PC 支持

TinyKV 所有写事务，即使只涉及单个 key，也执行完整 2PC：

```
单 key 写：PrewriteRequest → CommitRequest（2 次 RPC）
```

1PC 优化将此简化为：

```
单 key 写：PrewriteRequest（含 commit 信息）（1 次 RPC）
```

TiKV 在检测到单 Region 事务时自动使用 1PC。

#### 7.2.3 无写入管道化

TinyKV 的 KvPrewrite 串行处理每个 mutation：

- 检测冲突 → 写 Default CF → 写 Lock CF
- 等待第 N 个 key 完成后，才处理第 N+1 个 key

工业级实现通常并行处理多个 key 的加锁操作（Pipelined Locking）。

#### 7.2.4 Scanner 不处理锁等待

TinyKV 的 Scanner 在遇到锁冲突时直接将错误记录到结果中返回：

```go
// 现有实现：锁冲突直接返回错误
if err, ok := err.(*mvcc.KeyError); ok {
    pairs = append(pairs, &kvrpcpb.KvPair{Error: err.Err})
}
```

工业级实现应该等待锁释放或协助清理 Stale Lock，然后继续扫描，对客户端透明。

### 7.3 隔离级别问题

#### 7.3.1 TinyKV 的隔离级别

TinyKV 实现达到的隔离级别是**快照隔离（Snapshot Isolation, SI）**：

**能防止**：

- 脏读 ✓（通过 CfWrite 只读已提交数据）
- 不可重复读 ✓（通过 startTS 固定快照）
- 大多数幻读 ✓（快照读不受并发插入影响）

**不能防止**：

- **写偏斜（Write Skew）** ✗

#### 7.3.2 写偏斜示例

```
场景：医院值班系统，约束：至少 1 名医生在岗
当前状态：医生 A（on_call=true），医生 B（on_call=true）

T1（医生 A 申请下班）:
  startTS=100
  Read: SELECT count(*) FROM doctors WHERE on_call=true → 2
  决策: 2 > 1，可以下班
  Write: UPDATE doctors SET on_call=false WHERE id='A'

T2（医生 B 申请下班）:
  startTS=100（与 T1 相同快照）
  Read: SELECT count(*) FROM doctors WHERE on_call=true → 2
  决策: 2 > 1，可以下班
  Write: UPDATE doctors SET on_call=false WHERE id='B'

两个事务都提交成功（无冲突，因为写的是不同 key）
最终结果：A.on_call=false, B.on_call=false → 0 名医生在岗！违反约束
```

**原因**：T1 和 T2 都读了对方的写（`on_call` 字段），但写的是不同的 key（`A` 和 `B`），MVCC 无法检测到这种读-写依赖。

**解决方案**：

- **悲观锁**：T1 和 T2 各自对读到的所有行加 SELECT FOR UPDATE，T2 会阻塞等待 T1
- **SSI（串行化快照隔离）**：检测读写冲突环，自动中止一个事务
- **应用层控制**：在应用层显式加锁（如 Redis 分布式锁）

#### 7.3.3 MVCC GC 问题

随着时间推移，MVCC 数据不断积累历史版本，会占用大量存储空间并降低读性能（需要扫描更多历史版本）。

**TinyKV 现状**：没有实现 MVCC GC，历史版本会无限积累。

**工业级 TiKV 的 GC 机制**：

- 定期从 PD 获取 **GC Safe Point**（所有活跃事务的最小 startTS）
- 对 GC Safe Point 之前的所有版本进行清理
- 只保留每个 key 在 GC Safe Point 时刻可见的最新版本

```
Safe Point = 100
CfDefault 中 startTS < 100 的旧版本 → 可以删除
CfWrite 中 commitTS < 100 的旧提交记录（保留每个 key 的最新一条）→ 部分删除
```

---

## 8. 关键设计思考与深度分析

### 8.1 为什么 Percolator 能去掉独立的协调者

传统 2PC 依赖独立协调者的原因：需要一个中立的节点来做全局决策（Commit or Abort），且该决策必须持久化。

Percolator 的洞察：

1. **单行原子性**：底层存储（BigTable/RocksDB）支持对单个 key 的原子读写
2. **Primary Key 作为决策点**：对 Primary Key 的 Write CF 写入是原子的，写入成功即代表"事务已提交"这一事实被持久化
3. **无需集中协调**：所有参与者只需查看 Primary Key 的状态，就能确定事务结果

**本质上**，Percolator 将"协调者的决策"从一个独立进程变成了"Primary Key 行上的一次原子写操作"，避免了独立协调者的单点故障。

**代价**：

- 读操作可能遇到进行中的事务的锁，需要额外逻辑（CheckTxnStatus）来处理
- 崩溃恢复需要检查 Primary Key 状态，比独立协调者的 WAL 恢复更复杂

### 8.2 为什么时间戳要倒序存储

**核心原因**：MVCC 的读操作需要找到"最新的、commitTS <= startTS 的版本"。

如果时间戳正序存储：

```
CfWrite（正序）:
  Alice/TS=85   （commitTS=85）
  Alice/TS=100  （commitTS=100）
  Alice/TS=200  （commitTS=200）

读操作（startTS=150）：
  Seek(Alice, TS=150)
  → 找到第一个 >= TS=150 的记录：Alice/TS=200
  → 但 200 > 150，不可见！
  → 需要向后扫，找上一个记录 Alice/TS=100 ✓
```

正序存储需要 Seek + 向前回退，实现复杂（LSM-Tree 的迭代器通常只支持向后扫描）。

如果时间戳倒序存储：

```
CfWrite（倒序，大 TS 排前面）:
  Alice/~TS=85   （对应 commitTS=85，字节值最大）
  Alice/~TS=100  （对应 commitTS=100）
  Alice/~TS=200  （对应 commitTS=200，字节值最小）

实际存储顺序（按字节值升序）:
  Alice/~TS=200  （字节值最小，排最前）
  Alice/~TS=100
  Alice/~TS=85

读操作（startTS=150）：
  Seek(EncodeKey("Alice", 150)) = Seek(Alice/~TS=150)
  → 第一条 >= Alice/~150 的记录：Alice/~100（因为 ~100 > ~150）
  → 对应 commitTS=100 ≤ 150，直接可用！✓
```

倒序存储使 Seek 直接定位到目标版本，向后遍历即可，实现简洁高效。

### 8.3 MVCC GC 的必要性与安全性

#### 8.3.1 为什么必须 GC

**存储膨胀**：每次写操作都产生新版本，删除操作也只是标记（WriteKindDelete），历史版本持续积累。

**读性能下降**：GetValue 中的 Seek 虽然能直接定位，但如果历史版本很多，会增加 LSM-Tree 的文件数量，降低扫描性能。

#### 8.3.2 Safe Point 的确定

**Safe Point** = 所有活跃长事务的最小 startTS。Safe Point 之前的所有版本，已经没有任何活跃事务会读取。

```
活跃事务: T1(startTS=90), T2(startTS=95), T3(startTS=110)
Safe Point = min(90, 95, 110) = 90

Safe Point = 90 意味着：
  - commitTS < 90 的历史版本：没有任何活跃事务的 startTS <= 90 能读到这些版本
  - 可以安全删除（保留每个 key 在 TS=90 时刻可见的最新版本）
```

#### 8.3.3 GC 的安全性保证

GC 需要保证：执行 GC 后，所有活跃事务（startTS >= Safe Point）的读操作结果不变。

证明：

- 对于 startTS >= Safe Point 的读事务，其快照版本 >= Safe Point
- GC 只删除 commitTS < Safe Point 的旧版本，保留最新可见版本
- 最新可见版本 = commitTS 最接近 Safe Point 的版本，保留后仍能被这些读事务读到

### 8.4 锁的 TTL 设计权衡

#### 8.4.1 TTL 太短的风险

```
场景：
大事务（写入 10000 个 key，需要 5 秒）
TTL = 3 秒

时序：
t=0: 开始 Prewrite
t=3: TTL 到期，其他事务清理了部分锁
t=5: 事务继续 Commit → 发现锁已被清理 → 事务失败，需要重试

结果：大事务被误杀，需要重试，浪费资源
```

#### 8.4.2 TTL 太长的风险

```
场景：
事务崩溃（客户端宕机），遗留了锁
TTL = 10 分钟

影响：
读取该 key 的其他事务等待长达 10 分钟（或需要主动发起 CheckTxnStatus）
系统吞吐量大幅下降
```

#### 8.4.3 自适应 TTL

TiKV 实现中，TTL 根据事务大小动态计算：

```
base_ttl = 3000ms（3 秒）
size_factor = 1ms per KB

TTL = base_ttl + size_factor × txn_size_kb
    = 3000 + 1 × 10000  （10MB 事务）
    = 13000ms（13 秒）
```

这样大事务有更长的 TTL，小事务快速清理，平衡了两种风险。

### 8.5 Primary Key 的性能影响

#### 8.5.1 热点问题

如果业务逻辑导致大量事务的 Primary Key 集中在同一个 key（如全局计数器、自增序列），该 key 会成为热点：

- 所有事务的 CheckTxnStatus 都需要读写这个 key
- 并发 Commit 时会产生锁竞争

**缓解方案**：

- 避免使用单一热点 key 作为 Primary
- 使用分布式序列（如 TiDB 的 `AUTO_RANDOM`）代替自增主键

#### 8.5.2 Primary Key 选择策略

实践中，Primary Key 通常选择事务写入的**第一个 key**，原因：

- 简单，不需要特殊处理
- 对于有序写入，第一个 key 是最早写入的，崩溃恢复时最先被检查
- TiKV 的实现也采用此策略

### 8.6 Read-Your-Own-Writes 一致性

**问题**：在分布式系统中，写入某个节点后立即读取，可能读不到（因为副本同步有延迟）。

**Percolator 的保证**：

- 同一事务内的写入，通过写缓冲（MvccTxn.writes）在客户端内存中维护
- 事务内读取自己写的 key 时，先查 write buffer，再查存储
- 事务提交后，其他客户端可能因副本延迟暂时读不到（但最终一致）

**注意**：TinyKV 的实现中，读请求创建新的 MvccTxn，不共享写缓冲，同一"会话"的读写实际上是独立的事务。真正的 Read-Your-Own-Writes 需要客户端维护更多状态。

---

## 9. 常见问题与注意事项（代码层面）

### 9.1 GetValue 的 Seek 越界问题

**问题**：GetValue 中 Seek 后可能扫到另一个 key 的版本。

```go
// 错误写法（未验证 user key）
iter.Seek(EncodeKey(key, txn.StartTS))
if iter.Valid() {
    item := iter.Item()
    value, _ := item.Value()
    write, _ := ParseWrite(value)
    return txn.Reader.GetCF(CfDefault, EncodeKey(key, write.StartTs))
    // ↑ 如果 Seek 到了下一个 key 的版本，write.StartTs 是另一个 key 的 TS！
}
```

```go
// 正确写法（必须验证 user key）
iter.Seek(EncodeKey(key, txn.StartTS))
if iter.Valid() {
    item := iter.Item()
    userKey := DecodeUserKey(item.Key())
    if !bytes.Equal(userKey, key) {
        return nil, nil  // 已超出此 key 的范围
    }
    // ... 继续处理
}
```

**原因**：CfWrite 按 (key ASC, ts DESC) 排序，当某个 key 没有 commitTS <= startTS 的版本时，Seek 可能跳到下一个 key 的版本。

### 9.2 CurrentWrite 的扫描终止条件

**问题**：CurrentWrite 从 TsMax 向小扫描，需要正确判断何时停止。

```go
for iter.Seek(EncodeKey(key, TsMax)); iter.Valid(); iter.Next() {
    item := iter.Item()
    userKey := DecodeUserKey(item.Key())

    if !bytes.Equal(userKey, key) {
        break  // 已扫完此 key 的所有版本，进入下一个 key
    }

    // 处理...
}
```

如果不检查 `!bytes.Equal(userKey, key)`，扫描会继续到其他 key 的版本，导致错误结果或无限循环。

### 9.3 KvCommit 的 Retryable 与 Abort 区分

```
情况1：lock.ts != startVersion（锁属于另一个事务）
  → Retryable: 当前事务的锁已被清理，客户端应重新开始事务

情况2：write.kind == WriteKindRollback（已回滚）
  → Abort: 不应该再提交（返回 Retryable: "already rolled back"）
  → 注意：有些实现在这里返回非 Retryable 错误，需根据测试要求调整

情况3：lock == nil（锁不存在）
  → Retryable: 同情况1，事务可能已超时被回滚
```

**关键**：区分 Retryable 和非 Retryable 对客户端行为影响很大：

- Retryable：客户端重试整个事务（重新获取 startTS）
- 非 Retryable（Abort）：客户端报错给用户，不重试

### 9.4 KvBatchRollback 的幂等性

```go
// 已 Rollback 的 key 直接跳过（幂等）
if write != nil && write.Kind == WriteKindRollback {
    continue  // 不重复写 Rollback
}

// 已提交的 key 报错
if write != nil && write.Kind != WriteKindRollback {
    return AbortError("already committed")
}
```

**为什么需要幂等**：客户端可能因为网络重试多次发送 BatchRollback 请求，对已处理的 key 再次 Rollback 应该静默成功，而不是报错。

### 9.5 KvCheckTxnStatus 必须在无锁时写 Rollback

**错误做法**：

```go
lock = GetLock(primaryKey)
if lock == nil {
    return {Action: LockNotExistRollback}  // 只返回，不写 Rollback 标记
}
```

**问题时序**：

```
t=1: T1 Prewrite（获得 startTS=100）
t=2: T1 的 Prewrite 网络延迟，锁还未写入
t=3: T2 读取 primaryKey，发现无锁 → CheckTxnStatus 返回 LockNotExistRollback（但未写标记）
t=4: T1 的 Prewrite 到达 → 锁成功写入！
t=5: T1 尝试 Commit → 发现锁存在 → 成功提交！
t=6: 数据不一致（T2 已经认为 T1 不存在，但 T1 实际提交了）
```

**正确做法**：写 Rollback 标记，使后续 Commit 失败：

```go
if lock == nil {
    txn.PutWrite(primaryKey, lockTS, WriteKindRollback)
    server.storage.Write(...)
    return {Action: LockNotExistRollback}
}
```

这样 T1 的 Commit 在 `CurrentWrite` 时发现 Rollback 记录，拒绝提交 ✓

### 9.6 AllLocksForTxn 的性能考量

```go
// 扫描所有 CfLock，找属于 startTS 的锁
func AllLocksForTxn(txn *MvccTxn) ([]KlPair, error) {
    iter := txn.Reader.IterCF(engine_util.CfLock)
    defer iter.Close()

    var locks []KlPair
    for iter.Seek(nil); iter.Valid(); iter.Next() {
        item := iter.Item()
        value, _ := item.Value()
        lock, _ := ParseLock(value)
        if lock.Ts == txn.StartTS {
            locks = append(locks, KlPair{Key: item.Key(), Lock: lock})
        }
    }
    return locks, nil
}
```

**性能问题**：需要扫描**所有** CfLock，如果系统中有大量锁（大量并发事务），这个操作的时间复杂度为 O(n)，n 是总锁数量。

**工业级优化**：

- 记录事务涉及的所有 key（在 Primary Key 的 lock 中存储 secondary keys 列表）
- ResolveLock 时直接读取 secondary keys 列表，不需要全表扫描
- 这也是 Async Commit 中 `secondaries` 字段的设计动机

### 9.7 PutWrite 的参数顺序

```go
// TinyKV 中 PutWrite 参数：key, ts（commitTS）, write
txn.PutWrite(key, req.CommitVersion, &mvcc.Write{StartTs: req.StartVersion, Kind: lock.Kind})
```

**注意**：

- `ts` 参数是 **commitTS**（用作 CfWrite 的键的时间戳部分）
- `write.StartTs` 是 **startTS**（存储在值中，用于从 CfDefault 读取实际数据）
- 两者含义不同，混淆会导致读操作找不到正确版本

---

## 11. 代码索引与附录

### 11.1 核心代码文件

| 文件                                                         | 内容             | 关键函数                                                     |
| ------------------------------------------------------------ | ---------------- | ------------------------------------------------------------ |
| [kv/transaction/mvcc/transaction.go](kv/transaction/mvcc/transaction.go) | MvccTxn 核心实现 | `GetLock`, `PutLock`, `GetValue`, `PutValue`, `CurrentWrite`, `MostRecentWrite`, `EncodeKey` |
| [kv/transaction/mvcc/scanner.go](kv/transaction/mvcc/scanner.go) | 范围扫描实现     | `NewScanner`, `Next`, `Close`                                |
| [kv/server/server.go](kv/server/server.go)                   | RPC 处理器       | `KvGet`, `KvPrewrite`, `KvCommit`, `KvScan`, `KvCheckTxnStatus`, `KvBatchRollback`, `KvResolveLock` |
| [kv/transaction/mvcc/lock.go](kv/transaction/mvcc/lock.go)   | Lock 结构体      | `ParseLock`, `Lock.ToBytes`, `Lock.Info`                     |
| [kv/transaction/mvcc/write.go](kv/transaction/mvcc/write.go) | Write 结构体     | `ParseWrite`, `Write.ToBytes`                                |

### 11.2 关键常量与类型

```go
// Column Families
engine_util.CfDefault  = "default"  // 存储实际值
engine_util.CfLock     = "lock"     // 存储锁
engine_util.CfWrite    = "write"    // 存储提交记录

// Write Kinds
mvcc.WriteKindPut      // 写入操作
mvcc.WriteKindDelete   // 删除操作
mvcc.WriteKindRollback // 回滚标记

// Lock Kinds（与 Write 相同类型）
mvcc.LockKindPut
mvcc.LockKindDelete

// 特殊时间戳
mvcc.TsMax = math.MaxUint64  // 最大时间戳，用于 Seek 最新版本
```

### 11.3 Percolator 事务流程速查

```
事务生命周期：

1. 客户端获取 startTS（TSO）

2. Prewrite 阶段（FOR 每个 mutation）:
   a. MostRecentWrite(key) → commitTS >= startTS? → WriteConflict
   b. GetLock(key) → lock != nil? → LockConflict
   c. PutValue(key, value) + PutLock(key, Lock{primary, startTS, TTL})
   d. storage.Write（批量原子提交）

3. 客户端获取 commitTS（TSO）

4. Commit 阶段（FOR 每个 key）:
   a. CurrentWrite(key) → 已有记录? → 幂等成功/已回滚报错
   b. GetLock(key) → 锁不存在或 ts 不匹配? → Retryable Error
   c. PutWrite(key, commitTS, Write{startTS, kind}) + DeleteLock(key)
   d. storage.Write（批量原子提交）

5. 崩溃恢复（遇到 Stale Lock）:
   a. CheckTxnStatus(primaryKey, lockTS) → 确认事务状态
   b. ResolveLock(startTS, commitTS) → 批量解决所有锁
```

### 11.4 隔离级别与 Percolator 对应关系

| 隔离级别            | 单机 InnoDB 实现         | Percolator 对应                            |
| ------------------- | ------------------------ | ------------------------------------------ |
| READ UNCOMMITTED    | 无 Read View             | 不适用（Percolator 始终读已提交）          |
| READ COMMITTED      | 每次读创建 Read View     | 理论上可通过每次读获取新 startTS 实现      |
| **REPEATABLE READ** | 事务开始时创建 Read View | **TinyKV 实现（startTS 固定快照）**        |
| SERIALIZABLE        | 2PL + Next-Key Lock      | 需要 SSI 或显式锁（Percolator 本身不支持） |

TinyKV 的实现达到快照隔离（SI），近似于可重复读（RR），但不等同于 SQL 标准的 SERIALIZABLE。

### 11.5 参考资料

1. **Percolator 论文**：Google, "Large-scale Incremental Processing Using Distributed Transactions and Notifications", OSDI 2010
2. **Spanner 论文**：Google, "Spanner: Google's Globally Distributed Database", TOCS 2013
3. **CockroachDB HLC**：Cockroach Labs, "Living Without Atomic Clocks", 2016
4. **TiKV 事务文档**：https://tikv.org/deep-dive/distributed-transaction/introduction/
5. **Async Commit 设计**：TiKV, "Async Commit, the Accelerator for Transaction Commit in TiKV 5.0", 2021
