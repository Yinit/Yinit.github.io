---
title: "TinyKV Raft Review"
date: 2026-04-05T07:53:47Z
lastMod: 2026-04-05T07:53:47Z
draft: false # 是否为草稿
author: ["tkk"]

categories: ["TinyKV", "Raft"]

tags: ["TinyKV", "Raft"]

keywords: ["TinyKV", "Raft"]

description: "深入解析 TinyKV Project2 Raft 协议实现，涵盖选举、日志复制、快照机制与 etcd 工程化设计细节" # 文章描述，与搜索优化相关
summary: "深入解析 TinyKV Raft 实现，涵盖 Leader 选举、日志复制、安全性保证与 etcd 工程化设计" # 文章简单描述，会展示在主页
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

## 1. 概述与背景

### 1.1 Project2 整体目标

Project2 要求在一个分布式 KV 存储系统中实现基于 Raft 协议的强一致性复制。整个 Project 分为三个递进的子部分：

| 子部分        | 目标                                 | 核心文件                                                     |
| ------------- | ------------------------------------ | ------------------------------------------------------------ |
| **Project2A** | 实现基础 Raft 算法（选举、日志复制） | `raft/raft.go`, `raft/log.go`, `raft/rawnode.go`             |
| **Project2B** | 在 Raft 之上构建容错 KV 服务         | `kv/raftstore/peer_msg_handler.go`, `kv/raftstore/peer_storage.go` |
| **Project2C** | 添加日志 GC 和快照支持               | 同上，扩展快照处理                                           |

### 1.2 整体架构分层

```
┌─────────────────────────────────────────────┐
│              Client                         │
└─────────────────┬───────────────────────────┘
                  │ RaftCmdRequest (gRPC)
┌─────────────────▼───────────────────────────┐
│          RaftStore / PeerMsgHandler         │  ← peer 层（Project2B）
│  proposeRaftCommand()  HandleRaftReady()    │
└─────────────────┬───────────────────────────┘
                  │ Propose / Ready
┌─────────────────▼───────────────────────────┐
│              RawNode                        │  ← raft 层接口（Project2A）
│    Tick()  Step()  Ready()  Advance()       │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│            Raft 状态机                      │  ← raft 核心（Project2A）
│  选举  日志复制  角色转换  消息处理          │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│             RaftLog                         │  ← 日志内存缓存（Project2A）
│  entries[]  committed  applied  stabled     │
└─────────────────┬───────────────────────────┘
                  │
┌─────────────────▼───────────────────────────┐
│    PeerStorage（Storage 接口实现）           │  ← 持久化层（Project2B）
│         raftDB（badger）                    │
│         kvDB（badger）                      │
└─────────────────────────────────────────────┘
```

### 1.3 关键设计思想

TinyKV 的 Raft 实现深度参考了 etcd 的 Raft 库，核心设计思想如下：

1. **状态机化**：整个 Raft 被设计为一个纯函数状态机，从一端输入消息（Step），从另一端输出结果（Ready），全程线性无并发。
2. **关注点分离**：Raft 模块只负责逻辑判断，持久化、网络收发全部交给上层（RawNode + PeerStorage）处理。
3. **Pull 模式输出**：Raft 不主动推送结果给上层，而是由上层定期轮询 `HasReady()` 拉取 Ready。

---

## 2. Raft 算法核心知识点

### 2.1 基础概念

#### 2.1.1 节点角色

Raft 集群中每个节点处于以下三种角色之一：

| 角色          | 说明                                                 |
| ------------- | ---------------------------------------------------- |
| **Leader**    | 集群唯一领导者，负责接收客户端请求、复制日志         |
| **Follower**  | 被动响应，不发起请求，只处理 Leader/Candidate 的 RPC |
| **Candidate** | 候选人，Leader 不可用时发起选举                      |

状态转换规则（对应 `raft.go` 中的 `becomeFollower/becomeCandidate/becomeLeader`）：

- 启动时默认 Follower
- Follower 选举超时 → Candidate
- Candidate 获得多数票 → Leader
- Candidate/Leader 发现更高 term → Follower

#### 2.1.2 任期（Term）

- 任期是严格单调递增的整数，逻辑时钟意义
- 每次选举任期加 1
- 节点收到更高 term 的消息时立即更新自己的 term 并转为 Follower
- Term 用于识别过期信息：任何携带旧 term 的消息都会被拒绝

```go
// raft.go - becomeFollower
func (r *Raft) becomeFollower(term uint64, lead uint64) {
    r.State = StateFollower
    r.Term = term
    r.Vote = lead
    r.Lead = lead
    r.electionElapsed = 0
    r.heartbeatElapsed = 0
    r.leadTransferee = None
}
```

注意：`becomeFollower` 中 `r.Vote = lead`，这是 TinyKV 的实现细节。将 Vote 设置为 lead，意味着在新 term 中如果该 Leader 发来 RequestVote（在 Leader Transfer 场景中），可以直接投票。

#### 2.1.3 日志结构

每条日志条目（Log Entry）包含三个字段：

```protobuf
message Entry {
    EntryType entry_type = 1;
    uint64 term = 2;     // 生成该日志时 Leader 的任期
    uint64 index = 3;    // 日志在序列中的位置（全局单调递增）
    bytes data = 4;      // 实际数据（序列化的命令）
}
```

**关键约束**：

- 如果两个节点日志中某位置 `<index, term>` 相同，则该位置及之前的所有日志完全相同（Log Matching 属性）
- 这个约束由 AppendEntries 中的 prevLogIndex/prevLogTerm 校验来保证

#### 2.1.4 重要的日志索引指针

所有节点维护：

- `committed`：已被集群多数确认的最高日志 index
- `applied`：已经应用到状态机的最高日志 index
- 不变式：`applied <= committed`

Leader 额外维护（`Prs map[uint64]*Progress`）：

- `nextIndex[i]`：下次发给节点 i 的日志起始 index，初始化为 leader 最后日志 index + 1
- `matchIndex[i]`：已确认节点 i 已复制的最高 index，初始化为 0

### 2.2 Leader 选举

#### 2.2.1 选举触发

Follower 维护一个 `electionElapsed` 计数器，每次 `tick()` 加 1。当 `electionElapsed >= electionTimeout` 时（且未收到 Leader 心跳），触发选举。

**随机化选举超时**：为避免多个节点同时选举导致分票，每次重置时 `electionTimeout` 是随机的：

```go
// util.go
func randElectionTimeout(timeout int) int {
    n, _ := rand.Int(rand.Reader, big.NewInt(int64(timeout)))
    return int(n.Int64()) + timeout
}
```

即超时范围是 `[ElectionTick, 2*ElectionTick)`，有效降低分票概率。

#### 2.2.2 选举流程

1. Follower → Candidate：`becomeCandidate()` 将 term +1，给自己投票，重置计时
2. 广播 `MsgRequestVote` 给所有其他节点
3. 收集投票结果，存储于 `r.votes map[uint64]bool`
4. 获得多数票 → `becomeLeader()`；被多数拒绝 → `becomeFollower()`

```go
// raft.go - becomeCandidate
func (r *Raft) becomeCandidate() {
    r.State = StateCandidate
    r.Term += 1
    r.Vote = r.id      // 给自己投票
    r.Lead = None
    r.votes = make(map[uint64]bool)
    r.votes[r.id] = true

    if len(r.Prs) == 1 {
        r.becomeLeader()  // 单节点直接成为 Leader
        return
    }
}
```

#### 2.2.3 投票规则

节点 B 给 Candidate A 投票的条件（同时满足）：

1. A 的 term >= B 的 term
2. B 在当前 term 还没有投过票（或已投给 A）
3. **A 的日志不比 B 的日志旧**（选举限制）

日志新旧比较规则：

- 先比较最后一条日志的 term，term 更大的更新
- term 相同时，index 更大（日志更长）的更新

```go
// raft.go - handleRequestVote
} else if r.Vote == None {
    lastIndex := r.RaftLog.LastIndex()
    lastTerm, _ := r.RaftLog.Term(lastIndex)
    if m.LogTerm > lastTerm || m.LogTerm == lastTerm && m.Index >= lastIndex {
        reject = false
        r.Vote = m.From
    }
}
```

**选举限制的意义**：确保每个当选的 Leader 都拥有当前集群中最完整的已提交日志，防止已提交的日志被覆盖。

#### 2.2.4 成为 Leader 的初始化

```go
// raft.go - becomeLeader
func (r *Raft) becomeLeader() {
    r.State = StateLeader
    r.Lead = r.id

    // 初始化所有节点的进度
    for id := range r.Prs {
        if id == r.id {
            r.Prs[id].Match = r.RaftLog.LastIndex()
        } else {
            r.Prs[id].Match = 0
        }
        r.Prs[id].Next = r.RaftLog.LastIndex() + 1
    }

    r.startPropose()  // 提交一条空日志（noop entry）
}
```

**重要**：Leader 在成为 Leader 后必须立即提交一条 **noop entry（空日志）**。原因：

- Raft 论文要求：Leader 只能通过当前任期的日志来推进 commit
- 通过提交一条当前任期的空日志，可以间接提交之前任期的所有未提交日志
- 这是 Leader Completeness 属性的工程化实现

### 2.3 日志复制

#### 2.3.1 正常流程

1. 客户端将请求发给 Leader（通过 `MsgPropose`）
2. Leader 将其追加到本地日志
3. Leader 广播 `MsgAppend` 给所有 Follower
4. Follower 检查 prevLogIndex/prevLogTerm，成功则追加日志，返回 `MsgAppendResponse`（Reject=false）
5. Leader 收到多数成功响应后推进 commitIndex
6. Leader 在下次心跳或 AppendEntries 中携带新的 commitIndex 告知 Follower
7. Follower 更新自己的 commitIndex

#### 2.3.2 AppendEntries 一致性校验

```
MsgAppend 字段：
- Index:   prevLogIndex（要追加日志的前一条日志的 index）
- LogTerm: prevLogTerm（要追加日志的前一条日志的 term）
- Entries: 要追加的日志条目
- Commit:  Leader 的 commitIndex
```

Follower 收到 AppendEntries 时的校验逻辑：

1. 如果 `prevLogIndex > lastIndex`：reject，表示缺少日志
2. 如果 `term(prevLogIndex) != prevLogTerm`：reject，表示日志冲突

```go
// log.go - Append
func (l *RaftLog) Append(preIndex, preTerm uint64, entries []*pb.Entry) bool {
    if term, err := l.Term(preIndex); err != nil || term != preTerm {
        return false
    }
    // 去除重复项（跳过已经一致的条目）
    offset := l.entries[0].Index
    for len(entries) > 0 {
        firstIndex := entries[0].Index
        if firstIndex > l.LastIndex() || l.entries[firstIndex-offset].Term != entries[0].Term {
            break
        }
        entries = entries[1:]
    }
    // 截断冲突条目并追加
    if len(entries) > 0 {
        l.entries = l.entries[:entries[0].Index-offset]
        l.stabled = min(l.stabled, l.LastIndex())
        for len(entries) > 0 {
            l.entries = append(l.entries, *entries[0])
            entries = entries[1:]
        }
    }
    return true
}
```

#### 2.3.3 日志回退优化

当 Follower 拒绝后，Leader 需要找到与 Follower 日志一致的最后位置（nextIndex 回退）。

**朴素方法**：每次 reject 后 nextIndex--，逐步回退（O(n) 次 RPC）

**优化方法（TinyKV 实现）**：Follower 在 AppendResponse 中携带 `Commit` 字段（当前 committedIndex），Leader 直接将 nextIndex 设置为 `Commit + 1`：

```go
// raft.go - handleAppendResponse
if m.Reject {
    r.Prs[m.From].Next = m.Commit + 1
    r.Prs[m.From].Match = m.Commit
    r.sendAppend(m.From)
}
```

这里使用 `committed` 作为回退点，因为已提交的日志一定是一致的，这样可以快速找到一致点。

#### 2.3.4 Commit 推进算法

Leader 收到多数成功 AppendResponse 后需要推进 commitIndex。

**条件**：存在 N > commitIndex，使得多数节点的 matchIndex[i] >= N，且 log[N].term == currentTerm

TinyKV 中使用**差分数组**加速：

```go
// raft.go - checkCommit
func (r *Raft) checkCommit() {
    if _, exist := r.Prs[r.id]; !exist {
        return
    }
    arr := make([]uint64, r.Prs[r.id].Next-r.RaftLog.committed)
    for _, p := range r.Prs {
        if p.Match >= r.RaftLog.committed {
            arr[p.Match-r.RaftLog.committed]++
        }
    }
    for i, n := len(arr)-1, 0; i > 0; i-- {
        n += int(arr[i])
        if n > len(r.Prs)/2 {
            r.RaftLog.committed = r.RaftLog.committed + uint64(i)
            r.sendAllAppend()
            break
        }
    }
}
```

算法思路：

- `arr[i]` 表示 matchIndex 恰好为 `committed + i` 的节点数量
- 从右往左累加，找到第一个累计数超过半数的位置
- 这个位置就是新的 commitIndex（前提是该位置的日志 term 必须等于 currentTerm）

**注意**：必须检查 `log[N].term == currentTerm`。这是安全性要求，防止提交"前任期日志"带来的问题（详见 2.4.2）。

### 2.4 安全性

#### 2.4.1 选举安全性

**保证**：每个任期内最多只有一个 Leader。

**实现机制**：

1. 每个节点在一个 term 内只能投一票（`r.Vote` 记录）
2. 需要获得多数票（`> len(Prs)/2`）才能成为 Leader
3. 由于多数集合有交集，两个不同的 Candidate 不可能同时获得多数票

#### 2.4.2 为何不能直接提交前任期日志

考虑如下场景（论文 Figure 8）：

```
时刻 a: S1 是 Leader，index=2 写入 term=2 的日志，复制到 S2
时刻 b: S1 崩溃，S5 成为 Leader(term=3)，在 index=2 写入 term=3 的日志
时刻 c: S5 崩溃，S1 重新成为 Leader(term=4)，将 term=2 的日志复制到多数节点
时刻 d1: S1 崩溃，S5 成为 Leader，覆盖了多数节点的 index=2 日志
时刻 d2: S1 不崩溃，通过 term=4 的日志间接提交了 index=2 的日志
```

结论：即使 term=2 的日志已经复制到多数节点（时刻 c），仍然可能被覆盖（时刻 d1）。因此 **不能仅凭"复制到多数节点"就提交前任期的日志**。

**正确做法**：只通过当前任期的日志来推进 commit。一旦当前任期的日志提交了，其之前的所有日志（包括前任期）也随之提交。

这正是 `becomeLeader` 中提交 noop entry 的原因。

#### 2.4.3 Leader 完备性（Leader Completeness）

**保证**：如果一条日志在某任期被提交，则所有更高任期的 Leader 都包含这条日志。

**实现**：通过选举限制（投票时检查日志新旧）来保证。只有日志不比多数节点旧的 Candidate 才能当选，而已提交的日志必然存在于多数节点中，因此新 Leader 一定包含所有已提交的日志。

### 2.5 集群成员变更

TinyKV 采用论文推荐的**单步成员变更**（每次只添加或删除一个节点），确保任何时刻不存在两个独立的多数集合。

在 TinyKV Project3A 中通过 `ConfChange` 消息实现：

- `ConfChangeType_AddNode`：调用 `r.addNode(id)`
- `ConfChangeType_RemoveNode`：调用 `r.removeNode(id)`

删除节点后，如果是当前节点自身被删除，触发 `destroyPeer()`。

### 2.6 日志压缩与快照

随着时间增长，日志会无限增大。快照（Snapshot）将状态机的某个时刻状态压缩存储，并丢弃该时刻之前的日志。

**快照包含**：

- 最后一条被压缩的日志的 `<index, term>`（用于 AppendEntries 的前缀检查）
- 状态机在该时刻的完整状态（KV 数据）
- 集群配置信息（ConfState）

**触发时机**：当 Leader 发现 Follower 所需的日志已经被压缩时（nextIndex < firstIndex），发送快照而非日志。

---

## 3. etcd Raft 工程化设计

### 3.1 核心设计：与应用层的解耦

etcd Raft 库（TinyKV 的参考实现）将 Raft 协议逻辑与应用层完全解耦。Raft 库本身不进行：

- 网络通信（不发送 RPC）
- 磁盘写入（不持久化）
- 状态机执行（不应用日志）

所有这些操作都通过 **Ready 结构体** 交给应用层处理。

### 3.2 Ready 结构体设计

```go
// rawnode.go
type Ready struct {
    *SoftState              // 易失性状态（Lead, RaftState），仅用于通知上层状态变化
    pb.HardState            // 持久性状态（Term, Vote, Commit），需要持久化
    Entries []pb.Entry      // 需要持久化的日志条目（unstable entries）
    Snapshot pb.Snapshot    // 需要应用的快照
    CommittedEntries []pb.Entry  // 可以应用到状态机的日志
    Messages []pb.Message   // 需要发送给其他节点的消息
}
```

**设计哲学**：

- `HardState` 和 `Entries` 必须在发送 `Messages` **之前**持久化，否则崩溃重启后可能不一致
- `CommittedEntries` 是已经被持久化过的日志，可以安全应用
- `SoftState` 不需要持久化，只是通知上层有状态变化

**上层处理 Ready 的标准顺序**（非常重要）：

1. 如果有 `Snapshot`，先应用快照
2. 将 `Entries` 写入持久化存储（WAL）
3. 将 `HardState` 写入持久化存储
4. 发送 `Messages` 给其他节点
5. 应用 `CommittedEntries` 到状态机
6. 调用 `Advance()` 通知 Raft 库处理完毕

### 3.3 Storage 接口抽象

```go
// storage.go - Storage 接口
type Storage interface {
    InitialState() (pb.HardState, pb.ConfState, error)
    Entries(lo, hi uint64) ([]pb.Entry, error)
    Term(i uint64) (uint64, error)
    LastIndex() (uint64, error)
    FirstIndex() (uint64, error)
    Snapshot() (pb.Snapshot, error)
}
```

Storage 代表已经持久化到磁盘的数据视图。Raft 库通过 Storage 接口读取已持久化的日志，但不通过 Storage 写入（写入由上层负责）。

在 TinyKV 中：

- 测试时使用 `MemoryStorage`（内存实现）
- 生产中由 `PeerStorage` 实现（基于 badger）

### 3.4 三层数据管理

etcd Raft 的数据分三层管理：

```
┌─────────────────────────────────────┐
│  unstable（raft 库内存中）           │
│  - 尚未持久化的日志                 │
│  - 待应用的快照                     │
└───────────────┬─────────────────────┘
                │ 持久化后
┌───────────────▼─────────────────────┐
│  Storage（应用层管理的持久化数据）   │
│  - 已持久化但未压缩的日志            │
│  - 最新的快照                       │
└─────────────────────────────────────┘
```

TinyKV 简化了这个设计，将 unstable 和 stable 的日志都存在 `RaftLog.entries[]` 中，用 `stabled` 指针区分。

### 3.5 Pipeline 异步设计

etcd Raft 支持 Pipeline：不等待上一批日志提交就继续发送下一批。这是通过 nextIndex 的乐观更新实现的：Leader 发送日志后立即更新 nextIndex，不等待响应就发送下一批。

TinyKV 实现中也有类似思路：`sendAllAppend()` 在 commit 推进后立即广播新的 commitIndex，无需等待。

### 3.6 与 TinyKV 实现的差异

| 特性                 | etcd Raft                     | TinyKV                          |
| -------------------- | ----------------------------- | ------------------------------- |
| Heartbeat            | 与 AppendEntries 合并         | 独立的 `MsgHeartbeat`           |
| unstable/stable 日志 | 分开存储                      | 混合存储（用 stabled 指针区分） |
| Progress 状态        | Probe/Replicate/Snapshot 三态 | 简化，只有 Match/Next           |
| ReadIndex            | 完整实现                      | 未实现（可选优化）              |
| PreVote              | 支持                          | 未实现（2A 测试不允许）         |

---

## 4. TinyKV Raft 层实现分析（Project2A）

### 4.1 RaftLog（log.go）

RaftLog 是 Raft 模块内部的日志内存缓存，负责管理所有未被 compact 的日志条目。

#### 4.1.1 数据结构

```go
// log.go
type RaftLog struct {
    storage Storage       // 已持久化数据的视图（只读）
    committed uint64      // 已被多数节点确认的最高 index
    applied   uint64      // 已应用到状态机的最高 index
    stabled   uint64      // 已持久化到 storage 的最高 index
    entries   []pb.Entry  // 所有未被 compact 的日志（含持久化和非持久化）
    pendingSnapshot *pb.Snapshot  // 待处理的快照（2C 使用）
}
```

**核心不变式**：

```
snapshot.Index <= firstIndex - 1 <= applied <= committed <= stabled <= lastIndex
```

实际上在 TinyKV 中，stabled 和 committed 的关系比较微妙：stabled 可能大于或小于 committed，因为持久化和提交是独立操作。

#### 4.1.2 哨兵节点设计

`entries[0]` 是一个**虚拟哨兵节点**（dummy entry），其 Index 和 Term 来自最后一次快照（或初始值 0）。

**为何需要哨兵节点**：

- AppendEntries 校验需要检查 prevLog（`entries[prevIndex]`），当 prevIndex 是最早可用的日志时，需要有一个"前一条"日志
- 哨兵节点的 Index 表示 "compact 截止位置"，firstIndex = entries[0].Index + 1
- 通过哨兵，`Term(i)` 方法可以均匀处理所有有效 index，包括 truncatedIndex

```go
// log.go - newLog
func newLog(storage Storage) *RaftLog {
    firstIndex, _ := storage.FirstIndex()
    lastIndex, _ := storage.LastIndex()
    term, _ := storage.Term(firstIndex - 1)  // 获取 truncatedTerm

    entries := make([]pb.Entry, 1)
    entries[0].Index = firstIndex - 1  // 哨兵节点的 Index = truncatedIndex
    entries[0].Term = term             // 哨兵节点的 Term = truncatedTerm
    ents, _ := storage.Entries(firstIndex, lastIndex+1)
    entries = append(entries, ents...)

    return &RaftLog{
        storage:   storage,
        stabled:   lastIndex,
        entries:   entries,
        committed: firstIndex - 1,
        applied:   firstIndex - 1,
    }
}
```

#### 4.1.3 关键方法实现

**LastIndex()**：返回最后一条日志的 index

```go
func (l *RaftLog) LastIndex() uint64 {
    return l.entries[0].Index + uint64(len(l.entries)) - 1
    // 哨兵的 Index + 总条数 - 1（哨兵本身）
}
```

**FirstIndex()**：返回第一条有效日志的 index

```go
func (l *RaftLog) FirstIndex() uint64 {
    return l.entries[0].Index + 1  // 哨兵 Index + 1
}
```

**Term(i)**：返回 index i 的日志 term

```go
func (l *RaftLog) Term(i uint64) (uint64, error) {
    offset := l.entries[0].Index
    if i < offset {
        return 0, ErrCompacted
    }
    if int(i-offset) >= len(l.entries) {
        return 0, ErrUnavailable
    }
    return l.entries[i-offset].Term, nil
}
```

**unstableEntries()**：返回需要持久化的日志

```go
func (l *RaftLog) unstableEntries() []pb.Entry {
    offset := l.entries[0].Index
    return l.entries[l.stabled+1-offset:]  // stabled+1 之后的都是未持久化的
}
```

**nextEnts()**：返回已提交但未应用的日志

```go
func (l *RaftLog) nextEnts() (ents []pb.Entry) {
    offset := l.entries[0].Index
    lo := l.applied + 1 - offset
    hi := l.committed + 1 - offset
    return l.entries[lo:hi]
}
```

#### 4.1.4 maybeCompact：日志 GC

当上层（PeerStorage）完成 snapshot 后，storage.FirstIndex() 会增大，RaftLog 中旧的条目可以删除：

```go
// log.go - maybeCompact
func (l *RaftLog) maybeCompact() {
    if FirstIndex, _ := l.storage.FirstIndex(); l.FirstIndex() < FirstIndex {
        l.entries = l.entries[FirstIndex-l.FirstIndex():]
        l.pendingSnapshot = nil
    }
}
```

**调用时机**：在 `tick()` 中每次调用，确保内存中的 entries 不会无限增长。

#### 4.1.5 ApplySnapshot：应用快照

```go
// log.go - ApplySnapshot
func (l *RaftLog) ApplySnapshot(snap *pb.Snapshot) {
    l.pendingSnapshot = snap
    l.entries = make([]pb.Entry, 1)
    l.entries[0].Index = snap.Metadata.Index  // 新的哨兵
    l.entries[0].Term = snap.Metadata.Term
    l.applied = snap.Metadata.Index
    l.committed = snap.Metadata.Index
    l.stabled = snap.Metadata.Index
}
```

应用快照后，所有旧的日志条目被清空，以快照的 index 作为新的起点，三个指针都推进到快照的 index。

### 4.2 Raft 核心模块（raft.go）

#### 4.2.1 Raft 结构体

```go
type Raft struct {
    id   uint64
    Term uint64        // 当前任期（需持久化）
    Vote uint64        // 当前任期投票给谁（需持久化）

    RaftLog *RaftLog   // 日志模块
    Prs  map[uint64]*Progress  // 每个节点的复制进度（仅 Leader 使用）
    State StateType    // 当前角色
    votes map[uint64]bool  // 选票记录（Candidate 使用）
    msgs  []pb.Message     // 待发送的消息
    Lead  uint64           // 当前 Leader ID

    heartbeatTimeout int   // 心跳超时（固定，来自 Config.HeartbeatTick）
    electionTimeout  int   // 选举超时（随机化）
    heartbeatElapsed int   // 心跳计时
    electionElapsed  int   // 选举计时

    leadTransferee  uint64  // Leader 迁移目标（3A）
    PendingConfIndex uint64 // 待应用的配置变更（3A）
    active  map[uint64]bool // 存活节点集合（用于孤岛检测）
    electionTick int        // ElectionTick 基准值（用于随机化）
}

type Progress struct {
    Match, Next uint64
}
```

#### 4.2.2 tick() 逻辑时钟

`tick()` 是 Raft 的逻辑时钟推进函数，上层每隔固定时间调用一次（通过 `RawNode.Tick()`）。

```go
// raft.go - tick
func (r *Raft) tick() {
    r.electionElapsed++
    r.RaftLog.maybeCompact()  // 顺便检查日志压缩

    switch r.State {
    case StateFollower:
        if r.electionElapsed >= r.electionTimeout {
            r.startElection()
        }
    case StateCandidate:
        if r.electionElapsed >= r.electionTimeout {
            r.startElection()  // 超时后重新选举
        }
    case StateLeader:
        if r.electionElapsed >= r.electionTimeout {
            r.electionElapsed = 0
            r.leadTransferee = None
            // 检查是否收到足够心跳响应（孤岛检测）
            activeNum := len(r.active)
            if activeNum*2 <= len(r.Prs) {
                r.becomeFollower(r.Term, None)
                r.RaftLog.pendingSnapshot = nil
                return
            }
            r.active = make(map[uint64]bool)
            r.active[r.id] = true
        }
        r.heartbeatElapsed++
        if r.heartbeatElapsed >= r.heartbeatTimeout {
            r.startBeat()
        }
    }
}
```

**Leader 孤岛检测**（TinyKV 特有设计）：

- Leader 维护一个 `active` map，记录在一个 electionTimeout 周期内收到响应（心跳响应或日志响应）的节点
- 每个 electionTimeout 周期检查：如果 active 节点数 <= 半数，说明可能被网络隔离（孤岛 Leader）
- 孤岛 Leader 应立即 becomeFollower，避免在网络分区中继续接受写请求

这解决了网络分区后旧 Leader 持续占用的问题。

#### 4.2.3 Step() 消息分发

```go
// raft.go - Step
func (r *Raft) Step(m pb.Message) error {
    switch r.State {
    case StateFollower:
        r.FollowerStep(m)
    case StateCandidate:
        r.CandidateStep(m)
    case StateLeader:
        r.LeaderStep(m)
    }
    return nil
}
```

注意：TinyKV 的 Step 没有做 term 的统一预处理（etcd 原版会先统一处理 term），而是在每个具体 handler 中分别处理。这需要在每个 handler 中仔细处理 term 比较。

#### 4.2.4 12 种 MessageType 详解

**① MsgHup**（Local Msg，触发选举）

| 发送方    | 接收方 | 触发条件 |
| --------- | ------ | -------- |
| 上层/tick | 自身   | 选举超时 |

```go
case pb.MessageType_MsgHup:
    r.becomeCandidate()
    r.sendRequestVote()  // 广播投票请求
```

特殊处理：Follower 收到 MsgHup 才发起选举；Leader 收到 MsgHup 忽略（已经是 Leader）。

---

**② MsgBeat**（Local Msg，触发心跳）

| 发送方 | 接收方         | 触发条件 |
| ------ | -------------- | -------- |
| tick   | 自身（Leader） | 心跳超时 |

```go
case pb.MessageType_MsgBeat:
    r.sendHeartbeat()  // 向所有 Follower 发送心跳
```

只有 Leader 处理 MsgBeat；Follower/Candidate 忽略。

---

**③ MsgPropose**（Local Msg，提交新日志）

| 发送方             | 接收方 | 触发条件     |
| ------------------ | ------ | ------------ |
| 上层（客户端请求） | Leader | 客户端写请求 |

```go
// raft.go - handlePropose
func (r *Raft) handlePropose(m pb.Message) error {
    if r.leadTransferee != None {
        return ErrProposalDropped  // 正在迁移 Leader，拒绝
    }
    // 设置 entry 的 index 和 term
    NextIndex := r.Prs[r.id].Next
    for i, entry := range m.Entries {
        entry.Index = NextIndex + uint64(i)
        entry.Term = r.Term
    }
    // 追加到本地日志
    lastIndex := r.RaftLog.LastIndex()
    term, _ := r.RaftLog.Term(lastIndex)
    r.RaftLog.Append(lastIndex, term, m.Entries)
    // 更新自身进度
    r.Prs[r.id].Next = r.RaftLog.LastIndex() + 1
    r.Prs[r.id].Match = r.RaftLog.LastIndex()
    // 单节点直接提交
    if len(r.Prs) == 1 {
        r.RaftLog.committed = r.RaftLog.LastIndex()
    }
    // 广播 AppendEntries
    r.sendAllAppend()
    return nil
}
```

非 Leader 节点收到 MsgPropose 时直接返回 `ErrProposalDropped`（或转发给 Leader）。

---

**④ MsgAppend**（Common Msg，日志复制）

字段说明：

- `Index`: prevLogIndex
- `LogTerm`: prevLogTerm
- `Entries`: 待追加的日志
- `Commit`: Leader 的 commitIndex

发送：

```go
// raft.go - sendAppend
func (r *Raft) sendAppend(to uint64) bool {
    index := r.Prs[to].Next - 1
    term, err := r.RaftLog.Term(index)
    if err == ErrCompacted {
        r.sendSnapshot(to)  // 需要的日志已被压缩，发快照
    } else {
        entries, _ := r.RaftLog.Entries(r.Prs[to].Next, r.RaftLog.LastIndex()+1)
        r.msgs = append(r.msgs, pb.Message{
            MsgType: pb.MessageType_MsgAppend,
            Term:    r.Term, From: r.id, To: to,
            Commit:  r.RaftLog.committed,
            Index:   index, LogTerm: term,
            Entries: entries,
        })
    }
    return true
}
```

接收处理：

```go
// raft.go - handleAppendEntries
func (r *Raft) handleAppendEntries(m pb.Message) {
    // 旧 Leader 的消息，拒绝
    if m.Term < r.Term || r.State == StateLeader && m.Term == r.Term {
        r.sendAppendResponse(m.From, true)
        return
    }
    r.becomeFollower(m.Term, m.From)  // 更新 term，确认 Leader
    reject := !r.RaftLog.Append(m.Index, m.LogTerm, m.Entries)
    if !reject {
        // 更新 committed
        if len(m.Entries) > 0 {
            r.RaftLog.committed = max(r.RaftLog.committed,
                min(m.Commit, m.Entries[len(m.Entries)-1].Index))
        } else {
            r.RaftLog.committed = max(r.RaftLog.committed, min(m.Commit, m.Index))
        }
    }
    r.sendAppendResponse(m.From, reject)
}
```

**committed 更新逻辑**：取 `min(m.Commit, 最新追加的日志 Index)`，防止 committed 超过自己实际拥有的日志范围。

---

**⑤ MsgAppendResponse**（Common Msg，日志复制响应）

字段说明：

- `Reject`: 是否拒绝
- `Index`: 成功时为最后追加的 index；拒绝时辅助 Leader 快速定位
- `Commit`: 节点当前 committedIndex（用于 reject 时回退）

处理：

```go
// raft.go - handleAppendResponse
func (r *Raft) handleAppendResponse(m pb.Message) {
    if m.Term > r.Term {
        r.becomeFollower(m.Term, None)
        return
    }
    r.active[m.From] = true  // 记录该节点活跃

    if m.Reject {
        r.Prs[m.From].Next = m.Commit + 1  // 回退到对方 committed 之后
        r.Prs[m.From].Match = m.Commit
        r.sendAppend(m.From)               // 重新发送
    } else {
        r.Prs[m.From].Next = m.Index + 1
        r.Prs[m.From].Match = r.Prs[m.From].Next - 1
        // 检查是否可以推进 commit
        if term, _ := r.RaftLog.Term(m.Index); term == r.Term && m.Index > r.RaftLog.committed {
            r.checkCommit()
        }
        // Leader Transfer 完成检查
        if r.leadTransferee == m.From && r.Prs[m.From].Match == r.RaftLog.LastIndex() {
            r.sendTimeoutNow(m.From)
        }
    }
}
```

---

**⑥ MsgRequestVote**（Common Msg，请求投票）

字段：

- `Term`: Candidate 的 term
- `Index`: Candidate 最后一条日志的 index
- `LogTerm`: Candidate 最后一条日志的 term

处理：

```go
// raft.go - handleRequestVote
func (r *Raft) handleRequestVote(m pb.Message) {
    if m.Term > r.Term {
        r.becomeFollower(m.Term, None)
    }
    reject := true
    if m.Term < r.Term || r.State != StateFollower && m.Term == r.Term {
        reject = true  // term 小或非 Follower 状态
    } else if r.Vote == m.From {
        reject = false  // 已经投给他了
    } else if r.Vote == None {
        // 检查日志新旧
        lastIndex := r.RaftLog.LastIndex()
        lastTerm, _ := r.RaftLog.Term(lastIndex)
        if m.LogTerm > lastTerm || m.LogTerm == lastTerm && m.Index >= lastIndex {
            reject = false
            r.Vote = m.From
        }
    }
    r.sendRequestVoteResponse(m.From, reject)
}
```

---

**⑦ MsgRequestVoteResponse**（Common Msg，投票结果）

```go
// raft.go - handleRequestVoteResponse
func (r *Raft) handleRequestVoteResponse(m pb.Message) {
    if m.Term > r.Term {
        r.becomeFollower(m.Term, None)
        return
    }
    if _, exists := r.votes[m.From]; exists {
        return  // 已处理过该节点的响应
    }
    r.votes[m.From] = !m.Reject

    if len(r.votes)*2 <= len(r.Prs) {
        return  // 票数还不够，继续等待
    }
    // 统计票数
    argNum, denNum := 0, 0
    for _, v := range r.votes {
        if v { argNum++ } else { denNum++ }
    }
    if argNum*2 > len(r.Prs) {
        r.becomeLeader()
    }
    if denNum*2 > len(r.Prs) {
        r.becomeFollower(m.Term, None)
    }
}
```

**早退出优化**：只有当 `votes` 数量 > 一半时才进行统计，避免频繁统计。

---

**⑧ MsgHeartbeat**（Common Msg，心跳）

字段：

- `Term`: Leader 的 term
- `Commit`：在 TinyKV 实现中，心跳携带 `min(matchIndex, committed)`，让 Follower 推进 committed

处理：

```go
// raft.go - handleHeartbeat
func (r *Raft) handleHeartbeat(m pb.Message) {
    if m.Term < r.Term || r.State == StateLeader && m.Term == r.Term {
        r.sendHeartbeatResponse(m.From)
        return
    }
    r.becomeFollower(m.Term, m.From)
    r.sendHeartbeatResponse(m.From)
}
```

Follower 收到心跳后：重置 `electionElapsed`（隐含在 `becomeFollower` 中），回复心跳响应。

---

**⑨ MsgHeartbeatResponse**（Common Msg，心跳响应）

```go
// raft.go - handleHeartbeatResponse
func (r *Raft) handleHeartbeatResponse(m pb.Message) {
    if m.Term > r.Term {
        r.becomeFollower(m.Term, None)
        return
    }
    r.active[m.From] = true  // 记录活跃（用于孤岛检测）
    // 如果 Follower 落后，触发日志同步
    if r.RaftLog.committed > m.Commit {
        r.sendAppend(m.From)
    }
}
```

Leader 通过检查 `r.RaftLog.committed > m.Commit` 来判断 Follower 是否落后，若落后则主动触发 AppendEntries。这解决了网络恢复后 Follower 长时间落后的问题。

---

**⑩ MsgSnapshot**（Common Msg，快照）

只有 Leader 会发送快照（当 Follower 需要的日志已被压缩时）。

```go
// raft.go - sendSnapshot
func (r *Raft) sendSnapshot(to uint64) {
    if r.RaftLog.pendingSnapshot == nil {
        snapshot, err := r.RaftLog.storage.Snapshot()
        if err != nil {
            return  // 快照尚未生成好，等待下次
        }
        r.RaftLog.pendingSnapshot = &snapshot
    }
    r.msgs = append(r.msgs, pb.Message{
        MsgType:  pb.MessageType_MsgSnapshot,
        Term:     r.Term, From: r.id, To: to,
        Snapshot: r.RaftLog.pendingSnapshot,
    })
}
```

Follower 接收快照：

```go
// raft.go - handleSnapshot
func (r *Raft) handleSnapshot(m pb.Message) {
    if m.Term < r.Term || r.State == StateLeader && m.Term == r.Term {
        r.sendAppendResponse(m.From, true)
        return
    }
    r.becomeFollower(m.Term, m.From)
    reject := false
    if m.Snapshot == nil || m.Snapshot.Metadata.Index <= r.RaftLog.committed {
        reject = true  // 快照比自己旧
    } else {
        // 更新 Prs（集群配置可能变化）
        if m.Snapshot.Metadata.ConfState != nil {
            r.Prs = make(map[uint64]*Progress, len(m.Snapshot.Metadata.ConfState.Nodes))
            for _, id := range m.Snapshot.Metadata.ConfState.Nodes {
                r.Prs[id] = &Progress{}
            }
        }
        r.RaftLog.ApplySnapshot(m.Snapshot)
    }
    r.sendAppendResponse(m.From, reject)
}
```

---

**⑪ MsgTransferLeader**（Local Msg，Leader 迁移，Project3）

Leader 收到后：

1. 设置 `leadTransferee = m.From`
2. 检查目标节点日志是否最新，若不是则先同步
3. 同步完成后发送 `MsgTimeoutNow`

---

**⑫ MsgTimeoutNow**（Local Msg，强制立即选举，Project3）

目标节点收到后立即发起选举（忽略 electionElapsed）：

```go
func (r *Raft) handleTimeoutNow() {
    if _, ok := r.Prs[r.id]; !ok {
        return  // 自己已被移出集群
    }
    r.startElection()
}
```

#### 4.2.5 Progress 与 sendAllAppend

`sendAllAppend()` 在 Leader 写入新日志或推进 commit 后调用，向所有 Follower 广播：

```go
func (r *Raft) sendAllAppend() {
    r.heartbeatElapsed = 0  // 重置心跳计时（AppendEntries 兼有心跳作用）
    for id := range r.Prs {
        if id == r.id {
            continue
        }
        r.sendAppend(id)
    }
}
```

### 4.3 RawNode（rawnode.go）

RawNode 是 Raft 模块暴露给上层的接口层，负责：

1. 向 Raft 传入外部消息（`Step`）和逻辑时钟推进（`Tick`）
2. 从 Raft 收集处理结果并打包成 `Ready` 供上层消费

#### 4.3.1 状态记录

```go
type RawNode struct {
    Raft      *Raft
    softState SoftState      // 上次生成 Ready 时的 SoftState 快照
    hardState pb.HardState   // 上次生成 Ready 时的 HardState 快照
}
```

RawNode 需要缓存上一次的状态，用于 `HasReady()` 检测是否有变化。

#### 4.3.2 HasReady()

```go
func (rn *RawNode) HasReady() bool {
    return len(rn.Raft.msgs) > 0 ||
        rn.Raft.RaftLog.stabled != rn.Raft.RaftLog.LastIndex() ||
        rn.Raft.RaftLog.applied != rn.Raft.RaftLog.committed ||
        rn.softState != rn.Raft.getSoftState() ||
        !isHardStateEqual(rn.hardState, rn.Raft.GetHardState()) ||
        rn.Raft.State != StateLeader && rn.Raft.RaftLog.pendingSnapshot != nil
}
```

判断条件：

1. 有待发送的消息
2. 有未持久化的日志（`stabled != lastIndex`）
3. 有未应用的日志（`applied != committed`）
4. SoftState 有变化（Leader 或 RaftState 变化）
5. HardState 有变化（Term/Vote/Commit 变化）
6. 有待应用的快照（非 Leader 且有 pendingSnapshot）

#### 4.3.3 Ready()

```go
func (rn *RawNode) Ready() Ready {
    ready := Ready{
        Entries:          rn.Raft.RaftLog.unstableEntries(),
        CommittedEntries: rn.Raft.RaftLog.nextEnts(),
        Messages:         rn.Raft.msgs,
    }
    if softState := rn.Raft.getSoftState(); rn.softState != softState {
        ready.SoftState = &softState
    }
    if hardState := rn.Raft.GetHardState(); !isHardStateEqual(rn.hardState, hardState) {
        ready.HardState = hardState
    }
    if rn.Raft.State != StateLeader && rn.Raft.RaftLog.pendingSnapshot != nil {
        ready.Snapshot = *rn.Raft.RaftLog.pendingSnapshot
    }
    return ready
}
```

**注意**：非 Leader 才能携带 Snapshot。Leader 上的 `pendingSnapshot` 是用于发送给 Follower 的快照缓存，不需要自己应用。

#### 4.3.4 Advance()

```go
func (rn *RawNode) Advance(rd Ready) {
    rn.Raft.msgs = nil  // 清空已发送的消息
    if len(rd.Entries) > 0 {
        rn.Raft.RaftLog.stabled = rd.Entries[len(rd.Entries)-1].Index
    }
    if len(rd.CommittedEntries) > 0 {
        rn.Raft.RaftLog.applied = rd.CommittedEntries[len(rd.CommittedEntries)-1].Index
    }
    if rd.SoftState != nil {
        rn.softState = *rd.SoftState
    }
    if !IsEmptyHardState(rd.HardState) {
        rn.hardState = rd.HardState
    }
    if !IsEmptySnap(&rd.Snapshot) {
        rn.Raft.RaftLog.pendingSnapshot = nil
    }
}
```

`Advance` 在上层调用时更新 RawNode 的状态快照，确保下次 `HasReady()` 能正确反映新状态。

---

## 5. TinyKV 上层应用层实现（Project2B）

### 5.1 架构概览

#### 5.1.1 Store / Peer / Region 三个核心概念

```
┌─────────────────────────────────────────────────────────┐
│                     RaftStore（节点）                    │
│  ┌──────────────────────┐  ┌──────────────────────────┐ │
│  │  Region A (Peer A1)  │  │  Region B (Peer B1)      │ │
│  │  [key_start, split)  │  │  [split, key_end)        │ │
│  └──────────────────────┘  └──────────────────────────┘ │
│          共用同一个 badger 实例                          │
└─────────────────────────────────────────────────────────┘
```

- **RaftStore（Store）**：每个物理节点一个，管理该节点上所有的 Region 和对应的 Peer
- **Region**：一个 Raft 组，负责某个 key 范围的数据（如 `[start, end)`）。多个 Region 的 Peer 散布在不同的 Store 上
- **Peer**：Region 在某个 Store 上的实例，包含一个 `RawNode`（Raft 状态机）和 `PeerStorage`

**关键约束**：一个 Region 在一个 Store 上最多只有一个 Peer（同一 Raft 组的副本无需在同一节点上多份存储）。

#### 5.1.2 消息流向

```
外部 RPC → RaftStore → router → Peer 的 mailbox → raftWorker
                                                       ↓
                                               HandleMsg(msg)
                                                       ↓
                                               HandleRaftReady()
```

### 5.2 raftWorker 驱动循环

```go
// raft_worker.go - run
func (rw *raftWorker) run(closeCh <-chan struct{}, wg *sync.WaitGroup) {
    defer wg.Done()
    var msgs []message.Msg
    for {
        msgs = msgs[:0]
        select {
        case <-closeCh:
            return
        case msg := <-rw.raftCh:  // 阻塞等待消息
            msgs = append(msgs, msg)
        }
        // 批量取出所有待处理消息
        pending := len(rw.raftCh)
        for i := 0; i < pending; i++ {
            msgs = append(msgs, <-rw.raftCh)
        }
        // 分发处理
        peerStateMap := make(map[uint64]*peerState)
        for _, msg := range msgs {
            peerState := rw.getPeerState(peerStateMap, msg.RegionID)
            if peerState == nil {
                continue
            }
            newPeerMsgHandler(peerState.peer, rw.ctx).HandleMsg(msg)
        }
        // 处理所有涉及到的 Peer 的 Ready
        for _, peerState := range peerStateMap {
            newPeerMsgHandler(peerState.peer, rw.ctx).HandleRaftReady()
        }
    }
}
```

**关键设计**：

1. **批量处理**：先批量取出所有消息再统一处理，减少循环开销
2. **先 HandleMsg 后 HandleRaftReady**：所有消息处理完后再统一检查 Ready，避免重复处理
3. **PeerMsgHandler 用完即丢**：每次都 new 一个，不持久化状态（状态在 peer 对象中）

### 5.3 PeerMsgHandler

#### 5.3.1 proposal 机制

`proposal` 是 peer 层记录"待回调"请求的数据结构：

```go
// peer.go
type proposal struct {
    index uint64          // 该请求在 Raft 日志中的 index
    term  uint64          // 请求时的 Leader term
    cb    *message.Callback  // 回调函数，用于通知客户端
}
```

**工作原理**：

1. 客户端发来 `RaftCmdRequest`，peer 将其 propose 到 Raft
2. 同时创建一个 `proposal`（记录 index/term/callback）
3. 当 Ready 中出现对应的 `CommittedEntry` 时，找到匹配的 proposal 调用 callback
4. 客户端通过 callback 获得执行结果

#### 5.3.2 proposeRaftCommand()

```go
// peer_msg_handler.go
func (d *peerMsgHandler) proposeRaftCommand(msg *raft_cmdpb.RaftCmdRequest, cb *message.Callback) {
    // 前置检查：是否是 Leader，term 是否正确，epoch 是否匹配
    err := d.preProposeRaftCommand(msg)
    if err != nil {
        if cb != nil { cb.Done(ErrResp(err)) }
        return
    }
    // 序列化请求
    data, err := msg.Marshal()
    if err != nil {
        if cb != nil { cb.Done(ErrResp(err)) }
        return
    }
    // 提交给 Raft
    err = d.RaftGroup.Propose(data)
    if err != nil {
        if cb != nil { cb.Done(ErrResp(err)) }
        return
    }
    // 记录 proposal（等待回调）
    if cb != nil {
        d.proposals = append(d.proposals, &proposal{
            index: d.RaftGroup.Raft.RaftLog.LastIndex(),
            term:  d.Term(),
            cb:    cb,
        })
    }
}
```

**前置检查 `preProposeRaftCommand`**：

1. 检查 storeID 是否正确
2. 检查当前节点是否是 Leader（非 Leader 返回 `ErrNotLeader`）
3. 检查 peerID 是否匹配
4. 检查 term 是否一致
5. 检查 RegionEpoch 是否过期

#### 5.3.3 HandleRaftReady()

```go
// peer_msg_handler.go
func (d *peerMsgHandler) HandleRaftReady() {
    if d.stopped { return }
    if !d.RaftGroup.HasReady() { return }

    ready := d.RaftGroup.Ready()
    d.Send(d.ctx.trans, ready.Messages)  // 先发消息

    // Apply CommittedEntries（含回调处理）
    if d.apply(&ready) {
        return  // 自身被删除，停止处理
    }

    // 持久化 Ready 中的数据
    if res, err := d.peerStorage.SaveReadyState(&ready); err != nil {
        log.Errorf(...)
    } else if res != nil {
        d.updateMeta(res.Region)  // 快照应用后更新 Region 元数据
    }

    d.RaftGroup.Advance(ready)  // 推进 RawNode
}
```

**注意 TinyKV 的处理顺序**（与标准顺序略有不同）：

1. 发送消息
2. Apply CommittedEntries
3. 持久化 HardState + Entries
4. Advance

标准的 etcd 顺序是先持久化再发送消息，TinyKV 简化了这个要求（因为测试不检查崩溃恢复场景的消息顺序）。

#### 5.3.4 apply() 与 doResp()

```go
// peer_msg_handler.go - apply
func (d *peerMsgHandler) apply(ready *raft.Ready) bool {
    if len(ready.CommittedEntries) > 0 {
        for _, entry := range ready.CommittedEntries {
            if stop, resp := d.newCmdResp(entry); stop {
                return true
            } else {
                d.doResp(resp, &entry)
            }
        }
    }
    return false
}
```

**doResp() 的 proposal 匹配逻辑**（这是一个重要的正确性保证）：

```go
// peer_msg_handler.go - doResp
func (d *peerMsgHandler) doResp(resp *raft_cmdpb.RaftCmdResponse, entry *eraftpb.Entry) {
    for len(d.proposals) > 0 {
        p := d.proposals[0]
        if entry.Index < p.index {
            return  // 当前 entry 还没到这个 proposal 的 index，跳过
        }
        if entry.Index > p.index {
            p.cb.Done(ErrRespStaleCommand(p.term))  // proposal 对应的 entry 已经被覆盖
            d.proposals = d.proposals[1:]
            continue
        }
        // entry.Index == p.index
        if entry.Term < p.term {
            return  // term 不匹配，等待
        }
        if entry.Term > p.term {
            p.cb.Done(ErrRespStaleCommand(p.term))  // term 不匹配，stale
            d.proposals = d.proposals[1:]
            continue
        }
        // index 和 term 都匹配，正常回调
        d.proposals = d.proposals[1:]
        p.cb.Txn = d.ctx.engine.Kv.NewTransaction(false)
        p.cb.Done(resp)
    }
}
```

**为何需要同时检查 index 和 term**？
在 Leader 切换时，同一个 index 可能被不同 term 的 entry 占用（旧的 entry 被覆盖）。如果 entry 的 term 与 proposal 记录的 term 不同，说明这是一个"新"的 entry 占用了旧 proposal 的 index，旧 proposal 应该以 `StaleCommand` 错误回调。

#### 5.3.5 HandleMsg() 消息处理

```go
// peer_msg_handler.go - HandleMsg
func (d *peerMsgHandler) HandleMsg(msg message.Msg) {
    switch msg.Type {
    case message.MsgTypeRaftMessage:
        // 外部 Raft 消息（其他节点发来的）
        d.onRaftMsg(msg.Data.(*rspb.RaftMessage))
    case message.MsgTypeRaftCmd:
        // 客户端请求（需要 propose）
        raftCMD := msg.Data.(*message.MsgRaftCmd)
        d.proposeRaftCommand(raftCMD.Request, raftCMD.Callback)
    case message.MsgTypeTick:
        d.onTick()  // 触发 RawNode.Tick()
    case message.MsgTypeSplitRegion:
        // Region Split（Project3B）
    case message.MsgTypeGcSnap:
        d.onGCSnap(...)  // 清理已应用的快照文件
    case message.MsgTypeStart:
        d.startTicker()  // 启动定时器（新 Peer 创建时）
    }
}
```

**onRaftMsg** 会在将消息传入 `RaftGroup.Step()` 之前做一系列验证：

- store ID 检查
- region epoch 检查（stale 消息过滤）
- 快照冲突检查

### 5.4 PeerStorage

PeerStorage 实现了 `raft.Storage` 接口，是 Raft 模块与持久化层之间的桥梁。

#### 5.4.1 存储布局

**raftDB（badger 实例 1）**：

- Raft 日志条目：key = `RaftLogKey(regionId, logIndex)`
- RaftLocalState：key = `RaftStateKey(regionId)`，内容包括 `HardState` + `LastIndex` + `LastTerm`

**kvDB（badger 实例 2）**：

- KV 数据（用户数据）
- RaftApplyState：key = `ApplyStateKey(regionId)`，内容包括 `AppliedIndex` + `TruncatedState`
- RegionLocalState：key = `RegionStateKey(regionId)`，内容包括 `Region` + `PeerState`

**为何分两个 DB**：

- raftDB 只用于 Raft 协议本身（日志 + 状态），用完即删（compact 后可删除）
- kvDB 存储业务数据，生命周期更长
- 分开存储方便独立的 GC 和快照策略

#### 5.4.2 SaveReadyState()

```go
// peer_storage.go - SaveReadyState
func (ps *PeerStorage) SaveReadyState(ready *raft.Ready) (*ApplySnapResult, error) {
    kvWB := new(engine_util.WriteBatch)
    raftWB := new(engine_util.WriteBatch)
    var result *ApplySnapResult

    // 1. 更新 HardState（如果有变化）
    if !raft.IsEmptyHardState(ready.HardState) {
        ps.raftState.HardState = &ready.HardState
    }

    // 2. 更新 AppliedIndex
    if len(ready.CommittedEntries) > 0 {
        ps.applyState.AppliedIndex = ready.CommittedEntries[len(ready.CommittedEntries)-1].Index
    }

    // 3. 持久化日志条目
    ps.Append(ready.Entries, raftWB)

    // 4. 处理快照（如果有）
    if !raft.IsEmptySnap(&ready.Snapshot) {
        result, err = ps.ApplySnapshot(&ready.Snapshot, kvWB, raftWB)
    } else {
        // 写入 RaftLocalState 和 ApplyState
        raftWB.SetMeta(meta.RaftStateKey(ps.region.Id), ps.raftState)
        kvWB.SetMeta(meta.ApplyStateKey(ps.region.Id), ps.applyState)
        meta.WriteRegionState(kvWB, ps.region, rspb.PeerState_Normal)
    }

    // 5. 批量写入磁盘
    raftWB.WriteToDB(ps.Engines.Raft)
    kvWB.WriteToDB(ps.Engines.Kv)
    return result, nil
}
```

**WriteBatch 优化**：将多次写操作批量化，通过 `WriteToDB` 一次性提交，减少磁盘 I/O 次数。

#### 5.4.3 Append()

```go
// peer_storage.go - Append
func (ps *PeerStorage) Append(entries []eraftpb.Entry, raftWB *engine_util.WriteBatch) error {
    if len(entries) == 0 {
        return nil
    }
    lastIndex := ps.raftState.LastIndex
    // 删除可能冲突的旧条目（index 在新条目范围内的旧条目）
    if entries[0].Index > lastIndex+1 {
        return errors.Errorf("missing log entry")
    }
    for i := entries[0].Index; i <= lastIndex; i++ {
        raftWB.DeleteMeta(meta.RaftLogKey(ps.region.Id, i))
    }
    // 写入新条目
    for _, entry := range entries {
        raftWB.SetMeta(meta.RaftLogKey(ps.region.Id, entry.Index), &entry)
    }
    // 更新 RaftLocalState
    ps.raftState.LastIndex = entries[len(entries)-1].Index
    ps.raftState.LastTerm = entries[len(entries)-1].Term
    return nil
}
```

**删除冲突条目**：如果已持久化的日志中有与新条目 index 重叠的部分（可能是旧 Leader 的日志），需要先删除，再写入新的。

#### 5.4.4 Storage 接口实现

```go
// peer_storage.go - InitialState
func (ps *PeerStorage) InitialState() (eraftpb.HardState, eraftpb.ConfState, error) {
    raftState := ps.raftState
    if raft.IsEmptyHardState(*raftState.HardState) {
        return eraftpb.HardState{}, eraftpb.ConfState{}, nil
    }
    return *raftState.HardState, util.ConfStateFromRegion(ps.region), nil
}

func (ps *PeerStorage) FirstIndex() (uint64, error) {
    return ps.truncatedIndex() + 1, nil
}

func (ps *PeerStorage) LastIndex() (uint64, error) {
    return ps.raftState.LastIndex, nil
}

func (ps *PeerStorage) Term(idx uint64) (uint64, error) {
    if idx == ps.truncatedIndex() {
        return ps.truncatedTerm(), nil
    }
    // ... 从 raftDB 读取
}
```

`truncatedIndex()` 是最后一次被压缩的日志的 index，`FirstIndex() = truncatedIndex() + 1`。

---

## 6. 日志压缩与快照（Project2C）

### 6.1 整体流程概览

```
触发条件：appliedIdx - firstIdx >= RaftLogGcCountLimit
           ↓
      onRaftGCLogTick()
           ↓
  proposeRaftCommand(CompactLogRequest)
           ↓
     Raft 集群同步该请求
           ↓
  HandleRaftReady -> apply CompactLogRequest
           ↓
  ScheduleCompactLog()（异步删除 raftDB 日志）
           ↓
  truncatedIndex 更新，FirstIndex 增大
           ↓
  Leader sendAppend 时发现 nextIndex < firstIndex
           ↓
  storage.Snapshot() 生成快照（异步）
           ↓
  Leader 发送 MsgSnapshot 给落后 Follower
           ↓
  Follower handleSnapshot → ApplySnapshot（RaftLog 层）
           ↓
  HandleRaftReady -> SaveReadyState -> ApplySnapshot（PeerStorage 层）
           ↓
  RegionTaskApply → 异步写入 kvDB
```

### 6.2 日志 GC 触发

```go
// peer_msg_handler.go - onRaftGCLogTick
func (d *peerMsgHandler) onRaftGCLogTick() {
    d.ticker.schedule(PeerTickRaftLogGC)
    if !d.IsLeader() { return }

    appliedIdx := d.peerStorage.AppliedIndex()
    firstIdx, _ := d.peerStorage.FirstIndex()
    var compactIdx uint64
    if appliedIdx > firstIdx && appliedIdx-firstIdx >= d.ctx.cfg.RaftLogGcCountLimit {
        compactIdx = appliedIdx
    } else {
        return
    }

    compactIdx -= 1  // 保留最后一条 applied 日志
    if compactIdx < firstIdx { return }

    term, err := d.RaftGroup.Raft.RaftLog.Term(compactIdx)
    // 构造 CompactLogRequest 并提交
    request := newCompactLogRequest(regionID, d.Meta, compactIdx, term)
    d.proposeRaftCommand(request, nil)  // callback 为 nil（系统内部请求）
}
```

**重要细节**：`proposeRaftCommand` 时 callback 为 nil，因为这是系统内部产生的请求，不需要回调给客户端。在 `doResp()` 中需要处理 `cb == nil` 的情况。

### 6.3 CompactLog 的 Apply

```go
// peer_msg_handler.go - handleAdminReq
case raft_cmdpb.AdminCmdType_CompactLog:
    compactLogReq := req.CompactLog
    if compactLogReq.CompactIndex > d.LastCompactedIdx &&
       compactLogReq.CompactIndex > d.peerStorage.truncatedIndex() {
        d.ScheduleCompactLog(compactLogReq.CompactIndex)  // 异步删除旧日志
        d.peerStorage.applyState.TruncatedState.Index = compactLogReq.CompactIndex
        d.peerStorage.applyState.TruncatedState.Term = compactLogReq.CompactTerm
    }
    resp.CompactLog = &raft_cmdpb.CompactLogResponse{}
```

**过期判断**：`compactLogReq.CompactIndex > d.LastCompactedIdx`，确保不会重复处理旧的 CompactLog 请求（网络重传或日志重放可能导致重复）。

### 6.4 快照生成（异步）

```go
// peer_storage.go - Snapshot
func (ps *PeerStorage) Snapshot() (eraftpb.Snapshot, error) {
    var snapshot eraftpb.Snapshot
    if ps.snapState.StateType == snap.SnapState_Generating {
        // 检查是否生成完成
        select {
        case s := <-ps.snapState.Receiver:
            if s != nil { snapshot = *s }
        default:
            return snapshot, raft.ErrSnapshotTemporarilyUnavailable
        }
        // 验证快照有效性
        // ...
        return snapshot, nil
    }
    // 发起异步生成任务
    ch := make(chan *eraftpb.Snapshot, 1)
    ps.snapState = snap.SnapState{
        StateType: snap.SnapState_Generating,
        Receiver:  ch,
    }
    ps.regionSched <- &runner.RegionTaskGen{
        RegionId: ps.region.GetId(),
        Notifier: ch,
    }
    return snapshot, raft.ErrSnapshotTemporarilyUnavailable  // 第一次调用总是返回此错误
}
```

**处理流程**：

1. 第一次调用：发起异步生成任务，返回 `ErrSnapshotTemporarilyUnavailable`
2. Leader 的 `sendSnapshot()` 收到此错误后放弃本次快照发送
3. 下次 `sendAppend()` 时再次调用 `sendSnapshot()`，这次可能快照已生成好

```go
// raft.go - sendSnapshot
func (r *Raft) sendSnapshot(to uint64) {
    if r.RaftLog.pendingSnapshot == nil {
        snapshot, err := r.RaftLog.storage.Snapshot()
        if err != nil {
            return  // 快照未就绪，等待下次
        }
        r.RaftLog.pendingSnapshot = &snapshot
    }
    // 发送快照消息
    r.msgs = append(r.msgs, pb.Message{...Snapshot: r.RaftLog.pendingSnapshot})
}
```

**pendingSnapshot 缓存**：快照生成好后缓存在 `pendingSnapshot` 中，避免重复生成。只有成功发送（或被 `maybeCompact` 清除）后才清空。

### 6.5 快照应用（PeerStorage 层）

```go
// peer_storage.go - ApplySnapshot
func (ps *PeerStorage) ApplySnapshot(snapshot *eraftpb.Snapshot, kvWB, raftWB *engine_util.WriteBatch) (*ApplySnapResult, error) {
    snapData := new(rspb.RaftSnapshotData)
    snapData.Unmarshal(snapshot.Data)

    // 清除旧数据（只有已初始化的 peer 才需要清除）
    if ps.isInitialized() {
        ps.clearMeta(kvWB, raftWB)
        ps.clearExtraData(snapData.Region)
    }

    // 更新 Region
    ps.SetRegion(snapData.Region)

    // 更新 raftState
    ps.raftState = &rspb.RaftLocalState{
        HardState: &eraftpb.HardState{
            Term: snapshot.Metadata.Term,
            Commit: snapshot.Metadata.Index,
        },
        LastIndex: snapshot.Metadata.Index,
        LastTerm:  snapshot.Metadata.Term,
    }

    // 更新 applyState
    ps.applyState = &rspb.RaftApplyState{
        AppliedIndex: snapshot.Metadata.Index,
        TruncatedState: &rspb.RaftTruncatedState{
            Index: snapshot.Metadata.Index,
            Term:  snapshot.Metadata.Term,
        },
    }

    // 写入元数据
    kvWB.SetMeta(meta.ApplyStateKey(snapData.Region.Id), ps.applyState)
    raftWB.SetMeta(meta.RaftStateKey(snapData.Region.Id), ps.raftState)
    meta.WriteRegionState(kvWB, snapData.Region, rspb.PeerState_Normal)

    // 异步应用 KV 数据
    ch := make(chan bool, 1)
    ps.regionSched <- &runner.RegionTaskApply{
        RegionId: ps.region.Id,
        Notifier: ch,
        SnapMeta: snapshot.Metadata,
        StartKey: ps.region.StartKey,
        EndKey:   ps.region.EndKey,
    }
    <-ch  // 等待完成（同步等待）

    return &result, nil
}
```

**关键注意点**：

1. 先判断 `ps.isInitialized()`：新建的 Peer（Project3B split 场景）startKey/endKey 为空，不能调用 clearExtraData，否则会清除其他 Peer 的数据
2. 使用 `snapData.Region.Id` 而非 `ps.region.Id` 保存状态（可能发生 Region 变化）
3. KV 数据通过 `RegionTaskApply` 异步写入，但 TinyKV 实现中同步等待（`<-ch`）以简化处理

### 6.6 maybeCompact 与 FirstIndex 的关联

```
compact 后 TruncatedState.Index 增大
    ↓
PeerStorage.FirstIndex() = truncatedIndex() + 1 增大
    ↓
RaftLog.maybeCompact() 发现 storage.FirstIndex() > log.FirstIndex()
    ↓
裁剪 entries（删除已压缩的条目）
    ↓
log.FirstIndex() 更新
    ↓
Leader sendAppend 时：Prs[to].Next-1 < log.FirstIndex()-1 时触发 sendSnapshot
```

---

## 7. 整体设计分析与关键决策

### 7.1 为何 entries 混合存储持久化与非持久化数据

etcd 原版实现将已持久化（stable）和未持久化（unstable）的日志分开存储。TinyKV 为了简化实现，将两者都放在 `RaftLog.entries[]` 中，通过 `stabled` 指针区分。

**优势**：实现简单，不需要维护两个数据结构，日志查询（Term、Entries）统一操作一个切片。

**代价**：内存占用略大（持久化后的数据没有及时从内存释放）；但通过 `maybeCompact()` 可以在快照后清理。

### 7.2 为何采用逻辑时钟

Raft 使用逻辑时钟（tick 计数）而非真实时间，原因：

1. **可测试性**：测试可以精确控制时间推进，不依赖真实时间
2. **确定性**：逻辑时钟不受系统负载、时钟漂移影响
3. **灵活性**：上层可以根据实际情况调整 tick 频率（如负载高时减少 tick）

**服务器性能问题**：在性能不足的机器上，tick 消息可能堆积，导致 `electionElapsed` 快速累积，触发频繁选举。解决方法：增大 `ElectionTick` 配置值。

### 7.3 HandleRaftReady 中的处理顺序

TinyKV 的实际顺序：

```
1. Send Messages（发送消息）
2. Apply CommittedEntries（应用日志）
3. SaveReadyState（持久化）
4. Advance（推进）
```

这与标准 etcd 的顺序（先持久化，再发消息）不同。在 TinyKV 中这是可以接受的，因为：

- TinyKV 的测试不模拟持久化后崩溃的场景
- 实际工程中确实应该先持久化再发消息，以保证崩溃恢复的正确性

**正确顺序的重要性（工程角度）**：

- 如果先发消息，消息到达对方，对方 commit 了，但本地还没持久化就崩溃
- 重启后本地没有这条日志，但集群认为已提交，导致数据不一致

### 7.4 active map 孤岛检测设计

```go
// Leader 在收到来自其他节点的响应时记录
r.active[m.From] = true

// 每个 electionTimeout 周期检查
activeNum := len(r.active)
if activeNum*2 <= len(r.Prs) {
    r.becomeFollower(r.Term, None)  // 孤岛，退出 Leader
}
r.active = make(map[uint64]bool)
r.active[r.id] = true  // 重置，只计自己
```

**为何使用 electionTimeout 作为检查周期**：

- heartbeatTimeout 过短，一次心跳没响应可能是网络抖动
- electionTimeout 足够长，在这个时间内没收到半数响应，说明真的被隔离了
- 这个时间也是 Follower 发起选举的超时，因此孤岛 Leader 退出后，集群会立即产生新 Leader

**孤岛 Leader 的 term 会持续增加吗**？

是的，孤岛 Leader becomeFollower 后 term 不变（保持发现孤岛时的 term），再次 startElection 时 term +1。如果被隔离时间长，term 会持续增加。但这不影响正确性，因为投票看日志，而孤岛 Leader 的日志落后，不会被选为新 Leader。

### 7.5 差分数组加速 commit 推进

传统方法是遍历所有 Prs，排序后取中位数（O(n log n)）。

TinyKV 使用差分数组（O(n)）：

- `arr[i]` = matchIndex 恰好等于 `committed + i` 的节点数
- 从右往左累加，第一个超过半数的位置就是新 commitIndex
- 需要范围：`committed + 1` 到 `Prs[r.id].Next - 1`（最大可能的 commitIndex）

这种方法避免了排序，在集群规模较大时有明显优势。

### 7.6 Heartbeat 与 AppendEntries 的关系

**论文设计**：空的 AppendEntries 作为心跳，减少消息类型。

**TinyKV 实现**：独立的 MsgHeartbeat，只携带 term 和 commit，不携带日志。

**优势**：心跳消息更轻量，网络开销小；缺点：不能通过心跳顺带推进 Follower 的 committed（只能通过下次 AppendEntries）。

---

## 8. 常见问题与注意事项

### 8.1 Project2A 常见 Bug

#### Bug 1：becomeCandidate 与发送投票请求耦合

**错误做法**：在 `becomeCandidate()` 中直接调用 `sendRequestVote()`

**问题**：某些测试用例期望 `becomeCandidate` 和发起选举是分开的。`becomeCandidate` 只是状态转换，`sendRequestVote` 是在处理 `MsgHup` 时才调用。

**正确做法**：

```go
case pb.MessageType_MsgHup:
    r.becomeCandidate()    // 状态转换
    r.sendRequestVote()    // 发送投票请求
```

#### Bug 2：随机选举超时范围

选举超时范围影响测试结果：

- 太小（如 1-10）：选举过快，term 比预期大
- 太大（如固定值）：没有随机性，容易分票

TinyKV 使用 `[ElectionTick, 2*ElectionTick)` 的随机范围（`randElectionTimeout`）。

#### Bug 3：Leader 更新 committed 后必须通知 Follower

```go
// 错误：只更新 Leader 自身的 committed
r.RaftLog.committed = newCommit

// 正确：更新后立即广播给 Follower
r.RaftLog.committed = newCommit
r.sendAllAppend()  // 让 Follower 也推进 committed
```

若不广播，集群 committed 不同步，Follower 无法 apply 最新的日志。

#### Bug 4：MsgRequestVoteResponse 中未处理重复响应

同一个 Follower 可能多次发送投票响应（网络重传）：

```go
// 防止重复处理
if _, exists := r.votes[m.From]; exists {
    return
}
r.votes[m.From] = !m.Reject
```

#### Bug 5：Prs 初始化来源

在测试中，`Config.peers` 非空（测试模式）；在实际运行中，`Config.peers` 为 nil，节点信息来自 `ConfState`：

```go
// raft.go - newRaft
if c.peers != nil {
    prs = make(map[uint64]*Progress, len(c.peers))
    for _, id := range c.peers {
        prs[id] = &Progress{}
    }
} else {
    prs = make(map[uint64]*Progress, len(confState.Nodes))
    for _, id := range confState.Nodes {
        prs[id] = &Progress{}
    }
}
```

### 8.2 Project2B 常见 Bug

#### Bug 1：find no region for key

**原因**：Prs 初始化时使用了 `c.peers`（测试传入），而非从 `confState` 读取。在 Project2B 中，所有的 peer 信息通过 `storage.InitialState()` 中的 `ConfState` 获取。

**排查方法**：检查 `newRaft` 的 Prs 初始化逻辑。

#### Bug 2：p.cb.Txn 未赋值

处理 `CmdType_Snap` 后必须赋值：

```go
case raft_cmdpb.CmdType_Snap:
    resp.Snap = &raft_cmdpb.SnapResponse{Region: d.Region()}
// 在 doResp 中：
p.cb.Txn = d.ctx.engine.Kv.NewTransaction(false)
p.cb.Done(resp)
```

若不赋值，调用方使用 `cb.Txn` 构造迭代器时会 nil pointer panic。

#### Bug 3：proposal 回调时机

proposal 匹配必须同时检查 index 和 term。只检查 index 会在 Leader 切换后产生错误的回调：

```
时刻1: Leader A (term=1) 在 index=5 写入了一个 proposal
时刻2: A 崩溃，Leader B (term=2) 在 index=5 写入不同内容
时刻3: index=5 的条目（term=2）被 apply
→ 必须检测 entry.Term != proposal.term，以 StaleCommand 错误回调
```

#### Bug 4：GC 请求的 callback 为 nil

日志 GC 是 Leader 内部触发的请求，`proposeRaftCommand` 时 `cb = nil`：

```go
d.proposeRaftCommand(request, nil)  // nil callback
```

在 `doResp` 中需要处理 `p.cb` 可能为 nil（实际上 nil callback 时不会创建 proposal，所以问题不在 doResp，而是确保 cb 为 nil 时不调用 `p.cb.Done()`）。

#### Bug 5：Leader 可以直接 becomeCandidate

在处理 `tick()` 时，孤岛 Leader 调用 `becomeFollower` 后会发起选举，成为 Candidate。`becomeCandidate` 不应该拒绝来自 Leader 状态的转换。

### 8.3 Project2C 常见 Bug

#### Bug 1：快照后 lastIndex 异常

**场景**：接收快照后，快照的 `Metadata.Index` 大于 Follower 当前的 `lastIndex`，导致 entries 为空，`lastIndex()` 返回哨兵的 index。

**问题**：`sendAppendResponse` 时的 `Index` 字段不正确，Leader 无法正确更新 matchIndex。

**解决**：在 `handleSnapshot` 后检查 `lastIndex < snapshot.Metadata.Index`，确保 lastIndex 反映快照的位置（`ApplySnapshot` 已经正确处理了这个问题，因为它重置了 entries）。

#### Bug 2：旧 CompactLog 请求过期

**场景**：由于日志重放或网络延迟，可能收到旧的 CompactLog 请求（CompactIndex < 当前 TruncatedState.Index）。

**解决**：

```go
if compactLogReq.CompactIndex > d.LastCompactedIdx &&
   compactLogReq.CompactIndex > d.peerStorage.truncatedIndex() {
    // 只处理新的 CompactLog
}
```

#### Bug 3：pendingSnapshot 语义混淆

- **Leader 的 pendingSnapshot**：是即将发送给 Follower 的快照缓存，可以在 `maybeCompact` 后被清空（快照已过期需要重新生成）
- **Follower 的 pendingSnapshot**：是即将应用的快照（通过 `ApplySnapshot` 设置），会被 Ready 携带到上层应用

混淆这两者会导致：Leader 误将自己的 pendingSnapshot 通过 Ready 发给上层（HasReady 的判断已经通过 `r.State != StateLeader` 过滤）。

#### Bug 4：ApplySnapshot 时的 isInitialized 判断

```go
// 只有已初始化的 peer 才需要清除旧数据
if ps.isInitialized() {
    ps.clearMeta(kvWB, raftWB)
    ps.clearExtraData(snapData.Region)
}
```

新创建的 Peer（startKey/endKey 为空）直接清除会删除不相关的数据。

---

## 9. 性能优化点

### 9.1 Pipeline 异步提交

当前实现中，Leader 发送 AppendEntries 后等待响应才继续。Pipeline 优化：Leader 可以连续发送多批日志，不等待前一批的响应。

**实现思路**：乐观更新 nextIndex（发送后立即增加），收到 reject 后回退。

### 9.2 Batch 写入

TinyKV 已实现 `WriteBatch`，将多个写操作合并为一次磁盘 I/O：

```go
kvWB := new(engine_util.WriteBatch)
raftWB := new(engine_util.WriteBatch)
// ... 多次 SetMeta/DeleteMeta ...
raftWB.WriteToDB(ps.Engines.Raft)
kvWB.WriteToDB(ps.Engines.Kv)
```

可以进一步优化：合并多个 Ready 的 Entries 一次性写入（Batch Ready）。

### 9.3 ReadIndex 优化只读请求

当前 TinyKV 所有读请求都走 Raft 日志（写日志 + 多数确认），性能较低。

**ReadIndex 方案**（未实现）：

```
Leader 收到读请求
  → 记录当前 commitIndex 为 readIndex
  → 广播心跳确认 Leader 身份
  → 等待 applyIndex >= readIndex
  → 执行读操作返回
```

不需要写 WAL，只需一轮心跳 RPC，延迟从 2RTT 降低到 1RTT。

### 9.4 LeaseRead

Leader 维护一个"租约"：收到多数心跳响应后，在 `electionTimeout` 内不可能有新 Leader 产生（因为任何 Follower 发起选举需要等到超时），在此期间可以直接读本地数据。

**缺点**：依赖物理时钟精确性，时钟漂移可能导致 stale read。

### 9.5 流量控制（etcd 的 Probe/Replicate/Snapshot 状态）

etcd 实现中，每个 Follower 的 Progress 有三种状态：

- **Probe**：刚成为 Leader 或发生 reject，每次只发一条日志，等待响应
- **Replicate**：正常同步状态，可以 pipeline 发送
- **Snapshot**：正在发送快照

TinyKV 简化了这个模型，所有节点都使用相同的发送逻辑，没有状态区分。

---

## 附录：关键代码位置索引

| 功能               | 文件                               | 函数/方法                |
| ------------------ | ---------------------------------- | ------------------------ |
| RaftLog 初始化     | `raft/log.go`                      | `newLog()`               |
| 日志追加校验       | `raft/log.go`                      | `Append()`               |
| 未持久化日志获取   | `raft/log.go`                      | `unstableEntries()`      |
| 可应用日志获取     | `raft/log.go`                      | `nextEnts()`             |
| 日志压缩           | `raft/log.go`                      | `maybeCompact()`         |
| 快照应用（Raft层） | `raft/log.go`                      | `ApplySnapshot()`        |
| Raft 初始化        | `raft/raft.go`                     | `newRaft()`              |
| 逻辑时钟           | `raft/raft.go`                     | `tick()`                 |
| 消息分发           | `raft/raft.go`                     | `Step()`                 |
| 日志复制发送       | `raft/raft.go`                     | `sendAppend()`           |
| 日志复制处理       | `raft/raft.go`                     | `handleAppendEntries()`  |
| 响应处理           | `raft/raft.go`                     | `handleAppendResponse()` |
| Commit 推进        | `raft/raft.go`                     | `checkCommit()`          |
| 投票处理           | `raft/raft.go`                     | `handleRequestVote()`    |
| Ready 判断         | `raft/rawnode.go`                  | `HasReady()`             |
| Ready 生成         | `raft/rawnode.go`                  | `Ready()`                |
| 状态推进           | `raft/rawnode.go`                  | `Advance()`              |
| 驱动循环           | `kv/raftstore/raft_worker.go`      | `run()`                  |
| 请求提交           | `kv/raftstore/peer_msg_handler.go` | `proposeRaftCommand()`   |
| Ready 处理         | `kv/raftstore/peer_msg_handler.go` | `HandleRaftReady()`      |
| 日志应用           | `kv/raftstore/peer_msg_handler.go` | `apply()`, `doResp()`    |
| 日志GC触发         | `kv/raftstore/peer_msg_handler.go` | `onRaftGCLogTick()`      |
| 状态持久化         | `kv/raftstore/peer_storage.go`     | `SaveReadyState()`       |
| 日志持久化         | `kv/raftstore/peer_storage.go`     | `Append()`               |
| 快照生成           | `kv/raftstore/peer_storage.go`     | `Snapshot()`             |
| 快照应用（存储层） | `kv/raftstore/peer_storage.go`     | `ApplySnapshot()`        |
| 随机选举超时       | `raft/util.go`                     | `randElectionTimeout()`  |