---
title: "TinyKV Multi-Raft 实现与复习"
date: 2026-04-07T06:50:10Z
lastMod: 2026-04-07T06:50:10Z
draft: false # 是否为草稿
author: ["Yinit"]

categories: ["数据库"]

tags: ["TinyKV", "Multi-Raft", "分布式", "Raft"]

keywords: ["TinyKV", "Multi-Raft", "Region Split", "ConfChange", "Leader Transfer", "分布式KV", "调度器"]

description: "深入解析 TinyKV Project3 Multi-Raft 实现，涵盖 Leader Transfer、ConfChange、Region Split 与 Scheduler 调度全流程" # 文章描述，与搜索优化相关
summary: "深入解析 TinyKV Multi-Raft 架构，涵盖 Leader Transfer、成员变更、Region 分裂与全局调度器实现" # 文章简单描述，会展示在主页
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

### 1.1 为什么需要 Multi-Raft

在 Project2 中，我们实现了基于单个 Raft Group 的 KV 存储服务。这个架构存在两个根本性的瓶颈：

**瓶颈一：可扩展性受限**

单个 Raft Group 意味着所有数据都由同一组节点管理，无法横向扩展。当数据量增长时，只能垂直扩容（更大的机器），无法通过增加节点来线性提升容量。

**瓶颈二：并发处理能力有限**

所有写请求都必须经过同一个 Raft Group 串行提交，即使拥有多台机器，也无法并行处理不同 key 范围的请求，吞吐量受限于单个 Raft Group 的处理速度。

**Multi-Raft 的解法**

Project3 引入了 Multi-Raft 架构：将整个 key 空间划分为多个 **Region**，每个 Region 负责一段连续的 key 范围，由独立的 Raft Group 管理。不同 Region 的请求可以并行处理，从而突破单 Raft Group 的性能瓶颈。

```
Project2（单 Raft）：
[key: 0 ～ 100] → 一个 Raft Group → 串行处理所有请求

Project3（Multi-Raft）：
[key: 0 ～ 50]  → Raft Group A → 并行处理
[key: 50 ～ 100] → Raft Group B → 并行处理
```

### 1.2 三个子部分

| 子部分        | 内容                                               | 难度  |
| ------------- | -------------------------------------------------- | ----- |
| **Project3A** | 在 Raft 层实现 Leader Transfer 和 ConfChange       | ★★☆   |
| **Project3B** | 在 RaftStore 层实现 ConfChange 应用和 Region Split | ★★★★★ |
| **Project3C** | 实现调度器（Scheduler）：心跳收集 + Region Balance | ★★★   |

Project3A 是 3B 的底层支撑，3C 相对独立，是调度系统的实现。整个 Project3 中最难的是 3B，涉及大量并发、网络分区、异常恢复场景的处理。

### 1.3 Project2 → Project3 的演进

Project2 建立了以下基础：

- Raft 状态机（log.go / raft.go / rawnode.go）
- RaftStore 消息处理（peer_msg_handler.go）
- 持久化存储（peer_storage.go）
- 日志压缩与快照（Project2C）

Project3 在此基础上新增：

- **Leader Transfer**：主动将 Leader 职责转移给指定节点
- **ConfChange**：动态增删集群成员（AddNode / RemoveNode）
- **Region Split**：将一个 Region 一分为二，支持数据分片
- **Scheduler**：全局调度器，监控集群状态并指挥 Region 迁移

---

## 2. TinyKV 整体架构深度解析

### 2.1 宏观架构

```
┌─────────────┐
│   Client    │  (读写请求)
└──────┬──────┘
       │ gRPC
┌──────▼──────────────────────────────────────────┐
│                  TinyKV Server                  │
│  ┌──────────────────────────────────────────┐   │
│  │              RaftStore                   │   │
│  │  ┌──────────┐  ┌──────────┐  ┌────────┐ │   │
│  │  │ Region 1 │  │ Region 2 │  │ ...    │ │   │
│  │  │ RaftGroup│  │ RaftGroup│  │        │ │   │
│  │  └──────────┘  └──────────┘  └────────┘ │   │
│  └──────────────────────────────────────────┘   │
│  ┌─────────────────┐  ┌──────────────────────┐  │
│  │    raftDB       │  │       kvDB           │  │
│  │  (Raft 元数据)  │  │   (KV 业务数据)      │  │
│  └─────────────────┘  └──────────────────────┘  │
└──────────┬──────────────────────────────────────┘
           │ 心跳/调度指令
┌──────────▼──────┐
│  TinyScheduler  │  (对应 TiDB PD，全局调度)
│  (Project3C)    │
└─────────────────┘
```

**关键组件说明：**

- **RaftStore**：管理本物理节点（Store）上所有 Region 的 Raft Group
- **Region**：一个 key 范围的逻辑分片，由独立 Raft Group 管理
- **TinyScheduler**：类似 TiDB 的 Placement Driver（PD），负责全局调度
- **raftDB**：存储 Raft 元数据（HardState、RaftApplyState、RegionLocalState 等）
- **kvDB**：存储业务 KV 数据（由各 Region apply 写入）

### 2.2 Store / Peer / Region 三层概念

理解这三个概念是理解 Multi-Raft 的基础：

```
物理层：
Store（物理机器/进程）
├── 运行多个 Peer（每个 Peer 属于不同 Region 的副本）
├── 共享 raftDB 和 kvDB
└── 运行 storeWorker、raftWorker 等

逻辑层：
Region（逻辑分片）
├── 管理一段 key 范围 [StartKey, EndKey)
├── 由 3 个（或更多）Peer 组成（分布在不同 Store 上）
└── 每个 Region 有独立的 Raft Group

对应关系：
Region 1: Peer{StoreId=1, PeerId=1}, Peer{StoreId=2, PeerId=2}, Peer{StoreId=3, PeerId=3}
Region 2: Peer{StoreId=1, PeerId=4}, Peer{StoreId=2, PeerId=5}, Peer{StoreId=3, PeerId=6}

同一 Store 上可以有多个 Region 的 Peer：
Store 1: [Peer 1 (Region 1 副本), Peer 4 (Region 2 副本), ...]
```

**RegionEpoch**

每个 Region 维护一个版本信息 `RegionEpoch`：

```protobuf
message RegionEpoch {
    uint64 conf_ver = 1;  // ConfChange 版本：AddNode/RemoveNode 时递增
    uint64 version  = 2;  // Split/Merge 版本：Region 分裂或合并时递增
}
```

RegionEpoch 用于判断消息的新旧：

- 网络分区后，两个 Leader 可能同时存在，RegionEpoch 较旧的消息应被丢弃
- 每次 ConfChange 后 ConfVer++，每次 Split 后 Version++

### 2.3 Router 消息路由机制

TinyKV 通过 `router` 结构（`kv/raftstore/router.go`）将消息分发到对应的 Region Peer：

```go
// router routes a message to a peer.
type router struct {
    peers       sync.Map         // regionID -> peerState
    peerSender  chan message.Msg  // 发给 peer 的消息队列（容量 40960）
    storeSender chan<- message.Msg // 发给 store 的消息队列
}
```

**关键方法：**

| 方法                  | 说明                                        |
| --------------------- | ------------------------------------------- |
| `register(peer)`      | 注册新的 Peer（Split 创建新 Region 时调用） |
| `close(regionID)`     | 关闭 Region（destroyPeer 时调用）           |
| `send(regionID, msg)` | 向指定 Region 发送消息                      |
| `sendStore(msg)`      | 向 Store 级别发送消息                       |

消息的分发路径：

1. 外部请求（Client、其他节点的 Raft 消息）→ peerSender 队列
2. raftWorker 从 peerSender 取出消息 → 找到对应 Region 的 peerMsgHandler → 处理

### 2.4 raftWorker 驱动循环

`raftWorker`（`kv/raftstore/raft_worker.go`）是 TinyKV 的核心驱动组件：

```
raftWorker.run() 循环：
    ├── 从 router.peerSender 批量取消息
    ├── 对每个消息，找到对应 peer 的 peerMsgHandler
    ├── 调用 handler.HandleMsg() 处理消息
    └── 调用 handler.HandleRaftReady() 处理 Ready
```

**关键设计：批量处理**

raftWorker 采用批量消息处理策略：每次循环尽量多取消息（最多 4096 条），减少 HandleRaftReady 的调用次数，提升整体吞吐量。

```go
// 批量取消息
var msgs []message.Msg
for {
    msgs = append(msgs, msg)
    if len(msgs) >= RaftWorkerMaxRecvMsgCnt {
        break
    }
    // 尝试继续取，取不到就停止
    select {
    case msg = <-rw.receiver:
    default:
        goto DONE
    }
}
```

### 2.5 storeWorker 职责

`storeWorker`（`kv/raftstore/store_worker.go`）负责 Store 级别的操作：

**主要职责：**

1. **处理 Raft 消息路由**（`onRaftMessage`）：收到其他节点发来的 Raft 消息，找到对应 peer 转发，如果 peer 不存在则尝试创建（`maybeCreatePeer`）
2. **定时 Store 心跳**（`onSchedulerStoreHeartbeatTick`）：向 Scheduler 汇报本 Store 的状态（Region 数量、存储容量等）
3. **快照 GC**（`onSnapMgrGC`）：清理过期快照文件

**maybeCreatePeer 的重要性：**

```go
func (d *storeWorker) onRaftMessage(msg *rspb.RaftMessage) error {
    // ...各种检查...
    created, err := d.maybeCreatePeer(regionID, msg)
    // ...
}
```

当一个新节点被 AddNode 后，Leader 会向其发送心跳。此心跳会经过 storeWorker 的 `onRaftMessage`，如果目标 Peer 不存在，则调用 `maybeCreatePeer` 创建。创建条件由 `IsInitialMsg` 判断：

```go
func IsInitialMsg(msg *eraftpb.Message) bool {
    return msg.MsgType == eraftpb.MessageType_MsgRequestVote ||
        // MsgHeartbeat 且 Commit == 0（RaftInvalidIndex）才触发创建
        (msg.MsgType == eraftpb.MessageType_MsgHeartbeat && msg.Commit == RaftInvalidIndex)
}
```

**关键细节**：Leader 发送给新节点的第一条心跳，其 `msg.Commit` 字段必须是 `RaftInvalidIndex`（即 0），才能触发 `maybeCreatePeer`。如果实现时直接复用普通心跳（commit 字段有值），新节点就永远无法被创建！

### 2.6 GlobalContext 与 StoreMeta

`GlobalContext` 包含整个 Store 共享的全局状态：

```go
type GlobalContext struct {
    cfg                *config.Config
    engine             *engine_util.Engines
    store              *metapb.Store
    storeMeta          *storeMeta      // 全局 Region 元数据
    snapManager        *snap.SnapManager
    router             *router
    trans              Transport
    schedulerTaskSender chan<- worker.Task
    regionTaskSender   chan<- worker.Task
    // ...
}
```

`storeMeta` 维护了 Store 级别的 Region 信息：

```go
type storeMeta struct {
    sync.RWMutex
    regions      map[uint64]*metapb.Region  // regionID -> Region 元数据
    regionRanges *btree.BTree               // 按 key 范围索引的 Region（用于路由）
}
```

`regionRanges` 是一个 B-tree，支持按 key 快速查找对应的 Region（用于请求路由）。**修改 storeMeta 必须加锁**，这是 3B 中经常遗忘的细节。

### 2.7 心跳双重机制

TinyKV 中存在两种完全不同的心跳：

| 心跳类型           | 发送方      | 接收方        | 目的                                                     |
| ------------------ | ----------- | ------------- | -------------------------------------------------------- |
| **Raft 心跳**      | Leader Peer | Follower Peer | 维持 Raft Leader 地位，防止 Follower 发起选举            |
| **Scheduler 心跳** | Leader Peer | TinyScheduler | 汇报 Region 状态（大小、Peers、Leader 信息），供调度决策 |

Scheduler 心跳触发时机：

- `peer_msg_handler.go` 的 `onHeartbeatSchedulerTick()` 定期调用
- ConfChange apply 后主动调用（`notifyHeartbeatScheduler`）
- Split apply 后主动调用

### 2.8 调度器工作原理

TinyScheduler 收到 Region 心跳后，检查是否需要下发调度指令：

```
Region 发送 RegionHeartbeatRequest
    └── TinyScheduler 接收
            ├── processRegionHeartbeat：更新本地 Region 信息
            └── 检查是否有 pending operator
                    ├── 有：通过 RegionHeartbeatResponse 下发（ChangePeer / TransferLeader）
                    └── 无：检查是否需要 balance，生成新 operator
```

调度指令通过心跳响应（`RegionHeartbeatResponse`）下发，最终通过 `sendAdminRequest` 封装成 `RaftCmdRequest` 发给 Leader。

---

## 3. Raft 层扩展实现（Project3A）

### 3.1 Leader Transfer 总体设计

Leader Transfer 允许当前 Leader 主动将 Leader 权限转移给指定节点，常用于：

- Region Balance 调度（先 Transfer，再 RemovePeer 原 Leader）
- 计划内维护（graceful shutdown 前先迁走 Leader）

**引入的两种消息类型：**

| 消息类型            | 语义                                                         |
| ------------------- | ------------------------------------------------------------ |
| `MsgTransferLeader` | 请求 Leader 将权限转移给指定节点（`msg.From` 为转移目标）。本地消息，不通过网络传播 |
| `MsgTimeoutNow`     | Leader 发给转移目标，命令其立即发起选举，不等待选举超时      |

**关键字段：**

```go
type Raft struct {
    // ...
    leadTransferee  uint64  // 当前 Leader Transfer 的目标节点 ID（0 表示没有进行中的 Transfer）
    transferElapsed int     // Transfer 已经持续的 tick 数（超过 electionTimeout 则放弃）
}
```

### 3.2 Leader Transfer 完整流程

**Step 1：上层调用 RawNode.TransferLeader()**

```go
func (rn *RawNode) TransferLeader(transferee uint64) {
    _ = rn.Raft.Step(pb.Message{
        MsgType: pb.MessageType_MsgTransferLeader,
        From:    transferee,  // From 字段存储转移目标
    })
}
```

注意：`MsgTransferLeader` 中 `From` 是转移目标（不是发送方），这与其他消息的语义不同。

**Step 2：Leader 处理 MsgTransferLeader**

```go
func (r *Raft) handleTransferLeader(m pb.Message) {
    // 1. 检查转移目标是否在集群中
    if _, ok := r.Prs[m.From]; !ok {
        return
    }
    // 2. 如果转移目标就是自己，忽略
    if m.From == r.id {
        return
    }
    // 3. 处理正在进行中的 Transfer
    if r.leadTransferee != None {
        if r.leadTransferee == m.From {
            return  // 相同目标，忽略重复请求
        }
        r.leadTransferee = None  // 不同目标，强制覆盖
    }
    // 4. 设置转移目标，开始计时
    r.leadTransferee = m.From
    r.transferElapsed = 0
    // 5. 判断目标日志是否足够新
    if r.Prs[m.From].Match == r.RaftLog.LastIndex() {
        // 日志已同步，直接发 TimeoutNow
        r.sendTimeoutNow(m.From)
    } else {
        // 日志落后，先帮目标同步日志
        r.sendAppend(m.From)
    }
}
```

**Step 3：等待日志同步（如需）**

当 Leader 收到转移目标的 `MsgAppendResponse` 时，检查是否同步完成：

```go
// 在处理 MsgAppendResponse 时
if r.leadTransferee != None && r.leadTransferee == m.From {
    if r.Prs[m.From].Match == r.RaftLog.LastIndex() {
        r.sendTimeoutNow(m.From)  // 同步完成，发 TimeoutNow
    }
}
```

**Step 4：转移目标收到 MsgTimeoutNow**

```go
func (r *Raft) handleTimeoutNow(m pb.Message) {
    // 检查自己是否在集群中（可能已被 RemoveNode）
    if _, ok := r.Prs[r.id]; !ok {
        return
    }
    // 立即发起选举，不等待 electionTimeout
    r.Step(pb.Message{MsgType: pb.MessageType_MsgHup})
}
```

目标节点收到 TimeoutNow 后立即调用 `MsgHup` 开始选举，由于其日志至少和原 Leader 一样新，在 term+1 的新一轮选举中必定能赢得多数票，成为新 Leader。

**Step 5：超时放弃**

```go
// 在 tickHeartbeat() 中
if r.leadTransferee != None {
    r.transferElapsed++
    if r.transferElapsed >= r.electionTimeout {
        // 超过一个选举超时，放弃 Transfer
        r.leadTransferee = None
    }
}
```

**Step 6：Propose 阻塞**

Transfer 进行中，Leader 拒绝新的 propose：

```go
func (r *Raft) handlePropose(m pb.Message) {
    if r.leadTransferee != None {
        return  // 正在 Transfer，拒绝新 propose
    }
    // ...正常处理
}
```

**Step 7：非 Leader 收到 MsgTransferLeader 转发**

```go
// Follower 收到 MsgTransferLeader
case pb.MessageType_MsgTransferLeader:
    if r.Lead != None {
        m.To = r.Lead
        r.msgs = append(r.msgs, m)  // 转发给当前 Leader
    }
```

虽然文档说 MsgTransferLeader 是本地消息，但测试中会将其发给 Follower，Follower 需要将其转发给 Leader。

### 3.3 Leader Transfer 成功的根本原因

LeaderTransfer 之所以可靠，原因在于：

1. 转移前确保目标节点日志与 Leader 完全一致（Match == LastIndex）
2. 目标节点收到 TimeoutNow 后以 `term+1`（比当前 Leader 的 term 大）发起选举
3. 其他节点收到更高 term 的 RequestVote，且目标节点日志最新，会投票支持
4. 原 Leader 收到更高 term 的消息后退化为 Follower

整个过程在一次选举周期内完成（通常很快），对外几乎无感知。

### 3.4 成员变更（ConfChange）总体设计

ConfChange 允许动态增删集群成员。TinyKV 实现的是**单步变更**（每次只增加或删除一个节点），而非 Raft 论文中的 Joint Consensus 算法。

**为什么用单步变更？**

Joint Consensus 允许一次性变更多个节点，但实现复杂。单步变更足够简单安全：每次只变更一个节点，不会产生两个独立的多数派，避免脑裂。

**PendingConfIndex 字段：**

```go
type Raft struct {
    // ...
    PendingConfIndex uint64  // 上一个 ConfChange Entry 的 Index（未 apply 前不允许新 ConfChange）
}
```

`PendingConfIndex` 保证同一时刻最多只有一个未 apply 的 ConfChange，防止并发成员变更导致不一致。

### 3.5 ConfChange 在 Raft 层的实现

**ProposeConfChange（通过 RawNode 调用）：**

```go
func (rn *RawNode) ProposeConfChange(cc pb.ConfChange) error {
    data, err := cc.Marshal()
    if err != nil {
        return err
    }
    ent := pb.Entry{
        EntryType: pb.EntryType_EntryConfChange,  // 特殊 Entry 类型
        Data:      data,
    }
    return rn.Raft.Step(pb.Message{
        MsgType: pb.MessageType_MsgPropose,
        Entries: []*pb.Entry{&ent},
    })
}
```

ConfChange Entry 与普通 Entry 的区别在于 `EntryType` 为 `EntryType_EntryConfChange`，上层 apply 时需要特殊处理。

**在 Propose 阶段更新 PendingConfIndex：**

```go
// 在 handlePropose 中，当 entry.EntryType == EntryConfChange 时
r.PendingConfIndex = r.RaftLog.LastIndex() + uint64(i) + 1
```

**addNode 实现：**

```go
func (r *Raft) addNode(id uint64) {
    if _, ok := r.Prs[id]; !ok {
        r.Prs[id] = &Progress{
            Next:  r.RaftLog.LastIndex() + 1,
            Match: 0,
        }
    }
    r.PendingConfIndex = None  // 清除 PendingConfIndex
}
```

**removeNode 实现：**

```go
func (r *Raft) removeNode(id uint64) {
    delete(r.Prs, id)
    r.PendingConfIndex = None  // 清除 PendingConfIndex

    // 关键：删除节点后 quorum 变小，可能有新的 entry 可以 commit
    if r.State == StateLeader {
        r.maybeCommit()
        // 需要广播 append，通知其他节点更新 commit
        r.bcastAppend()
    }

    // 如果被删的正是 leadTransferee，终止 Transfer
    if r.leadTransferee == id {
        r.leadTransferee = None
    }
}
```

**为什么 removeNode 后必须重新计算 commit？**

考虑以下场景（TestCommitAfterRemoveNode3A）：

```
初始状态：3 节点集群 [1, 2, 3]，节点 1 是 Leader
1. Leader 发送 entry3 给节点 2 和 3
2. 节点 2 返回了 AppendResponse（Match 更新为 3）
3. 节点 3 在返回 AppendResponse 之前被 removeNode 了
4. 此时集群只剩 [1, 2]，quorum = 2
5. entry3 已经在节点 1 和 2 上，满足多数派条件，应该 commit
6. 但 Leader 还没有收到节点 3 的 AppendResponse，不会主动计算 commit
7. 需要 removeNode 后立即重新计算 commit，推进 committedIndex
```

---

## 4. RaftStore 层实现（Project3B）

### 4.1 代码改动范围

Project3B 的主要改动集中在两个文件：

**`kv/raftstore/peer_msg_handler.go`**

| 函数                        | 改动内容                                                   |
| --------------------------- | ---------------------------------------------------------- |
| `proposeRaftCommand`        | 新增对 TransferLeader、ChangePeer、Split 的处理            |
| `HandleRaftReady`           | 新增 processCommittedEntry 对 ConfChange 和 Split 的 apply |
| `processConfChange`（新增） | 处理 ConfChange Entry 的 apply                             |
| `processSplit`（新增）      | 处理 Split Entry 的 apply                                  |

整体请求处理框架：

```go
func (d *peerMsgHandler) proposeRaftCommand(msg *raft_cmdpb.RaftCmdRequest, cb *message.Callback) {
    if msg.AdminRequest != nil {
        switch msg.AdminRequest.CmdType {
        case raft_cmdpb.AdminCmdType_CompactLog:     // 2C 已实现，propose 到 Raft
        case raft_cmdpb.AdminCmdType_TransferLeader: // 3B 新增，不 propose，直接执行
        case raft_cmdpb.AdminCmdType_ChangePeer:     // 3B 新增，ProposeConfChange
        case raft_cmdpb.AdminCmdType_Split:          // 3B 新增，普通 Propose
        }
    } else {
        // 普通 KV 请求，直接 Propose
    }
}
```

### 4.2 LeaderTransfer 上层处理

TransferLeader 是四种 AdminCmd 中最简单的，因为它**不需要经过 Raft 日志同步**：

```go
case raft_cmdpb.AdminCmdType_TransferLeader:
    // 直接调用 TransferLeader，不走 Propose
    d.RaftGroup.TransferLeader(msg.AdminRequest.TransferLeader.Peer.Id)
    // 立即返回 Response
    adminResp := &raft_cmdpb.AdminResponse{
        CmdType:        raft_cmdpb.AdminCmdType_TransferLeader,
        TransferLeader: &raft_cmdpb.TransferLeaderResponse{},
    }
    cb.Done(&raft_cmdpb.RaftCmdResponse{
        Header:        &raft_cmdpb.RaftResponseHeader{},
        AdminResponse: adminResp,
    })
```

**为什么不需要 Propose？**

Leader Transfer 不修改数据或集群成员，是一个单边操作（Leader 单方面决定转移），不需要其他节点确认，因此无需走 Raft 日志同步。

### 4.3 ConfChange Propose 阶段

```go
case raft_cmdpb.AdminCmdType_ChangePeer:
    // 1. 单步变更保证：只有上一个 ConfChange 已 apply 才能发起新的
    if d.peerStorage.AppliedIndex() >= d.RaftGroup.Raft.PendingConfIndex {
        // 2. 特殊情况：两节点且要 Remove Leader，先 Transfer
        if len(d.Region().Peers) == 2 &&
            msg.AdminRequest.ChangePeer.ChangeType == pb.ConfChangeType_RemoveNode &&
            msg.AdminRequest.ChangePeer.Peer.Id == d.PeerId() {
            for _, p := range d.Region().Peers {
                if p.Id != d.PeerId() {
                    d.RaftGroup.TransferLeader(p.Id)
                    break
                }
            }
            // 注意：这里不 return，让 client 重试（或者 return 也可以）
        }
        // 3. 创建 proposal
        d.proposals = append(d.proposals, &proposal{
            index: d.nextProposalIndex(),
            term:  d.Term(),
            cb:    cb,
        })
        // 4. 将 RaftCmdRequest 序列化后放入 ConfChange.Context
        context, _ := msg.Marshal()
        d.RaftGroup.ProposeConfChange(pb.ConfChange{
            ChangeType: msg.AdminRequest.ChangePeer.ChangeType,
            NodeId:     msg.AdminRequest.ChangePeer.Peer.Id,
            Context:    context,
        })
    }
```

**关键点：**

- `AppliedIndex >= PendingConfIndex` 才允许新的 ConfChange（单步保证）
- 两节点 Remove Leader 的特殊处理（否则会导致集群陷入死循环，详见第8节）
- `ProposeConfChange` 而非 `Propose`（这样 Entry 的 EntryType 会被设为 EntryConfChange）
- Context 字段存原始 RaftCmdRequest（apply 时反序列化获取 Peer 信息）

**ProposeConfChange vs Propose 的区别：**

```go
// Propose 产生普通 Entry
func (rn *RawNode) Propose(data []byte) error {
    ent := pb.Entry{EntryType: pb.EntryType_EntryNormal, Data: data}
    // ...
}

// ProposeConfChange 产生 ConfChange Entry
func (rn *RawNode) ProposeConfChange(cc pb.ConfChange) error {
    data, _ := cc.Marshal()
    ent := pb.Entry{EntryType: pb.EntryType_EntryConfChange, Data: data}
    // ...
}
```

HandleRaftReady 在 apply 时通过 `entry.EntryType` 区分两者的处理路径。

### 4.4 ConfChange Apply 阶段

ConfChange Entry commit 后，在 `HandleRaftReady` 的 apply 阶段处理：

```go
func (d *peerMsgHandler) processCommittedEntry(entry *pb.Entry, kvWB *engine_util.WriteBatch) *engine_util.WriteBatch {
    if entry.EntryType == pb.EntryType_EntryConfChange {
        var cc pb.ConfChange
        cc.Unmarshal(entry.Data)
        return d.processConfChange(entry, &cc, kvWB)
    }
    // ...普通 Entry 处理
}
```

**processConfChange 完整流程：**

```go
func (d *peerMsgHandler) processConfChange(entry *pb.Entry, cc *pb.ConfChange, kvWB *engine_util.WriteBatch) *engine_util.WriteBatch {
    // 1. 反序列化 Context 获取原始 RaftCmdRequest
    msg := &raft_cmdpb.RaftCmdRequest{}
    msg.Unmarshal(cc.Context)

    region := d.Region()

    // 2. 检查 RegionEpoch 是否过期（防止重复执行同一 ConfChange）
    // 测试会多次提交同一 ConfChange 直到 apply，RegionEpoch 检查能过滤重复
    if err := util.CheckRegionEpoch(msg, region, true); err != nil {
        if errEpochNotMatch, ok := err.(*util.ErrEpochNotMatch); ok {
            d.handleProposal(entry, ErrResp(errEpochNotMatch))
            return kvWB
        }
    }

    switch cc.ChangeType {
    case pb.ConfChangeType_AddNode:
        // 3a. 检查待添加节点是否已存在
        if d.searchPeerWithId(cc.NodeId) == len(region.Peers) {
            // 不存在，执行添加
            region.Peers = append(region.Peers, msg.AdminRequest.ChangePeer.Peer)
            region.RegionEpoch.ConfVer++
            // 写入 kvDB
            meta.WriteRegionState(kvWB, region, rspb.PeerState_Normal)
            // 更新 storeMeta（需加锁）
            d.updateStoreMeta(region)
            // 更新 peerCache（消息发送时需要知道对方的 StoreId）
            d.insertPeerCache(msg.AdminRequest.ChangePeer.Peer)
        }

    case pb.ConfChangeType_RemoveNode:
        // 3b. 如果删除目标是自己，直接销毁
        if cc.NodeId == d.PeerId() {
            d.destroyPeer()
            return kvWB  // 必须 return，后面不能再执行任何操作
        }
        // 不是自己，检查待删节点是否存在
        n := d.searchPeerWithId(cc.NodeId)
        if n != len(region.Peers) {
            region.Peers = append(region.Peers[:n], region.Peers[n+1:]...)
            region.RegionEpoch.ConfVer++
            meta.WriteRegionState(kvWB, region, rspb.PeerState_Normal)
            d.updateStoreMeta(region)
            d.removePeerCache(cc.NodeId)
        }
    }

    // 4. 更新 Raft 层的成员配置（调用 3A 实现的 addNode/removeNode）
    d.RaftGroup.ApplyConfChange(*cc)

    // 5. 处理 proposal 回调
    d.handleProposal(entry, &raft_cmdpb.RaftCmdResponse{
        Header: &raft_cmdpb.RaftResponseHeader{},
        AdminResponse: &raft_cmdpb.AdminResponse{
            CmdType:    raft_cmdpb.AdminCmdType_ChangePeer,
            ChangePeer: &raft_cmdpb.ChangePeerResponse{Region: region},
        },
    })

    // 6. 如果是 Leader，通知 Scheduler 更新 Region 缓存
    if d.IsLeader() {
        d.notifyHeartbeatScheduler(region, d.peer)
    }

    return kvWB
}
```

**notifyHeartbeatScheduler 的必要性：**

```go
func (d *peerMsgHandler) notifyHeartbeatScheduler(region *metapb.Region, peer *peer) {
    clonedRegion := new(metapb.Region)
    util.CloneMsg(region, clonedRegion)
    d.ctx.schedulerTaskSender <- &runner.SchedulerRegionHeartbeatTask{
        Region:          clonedRegion,
        Peer:            peer.Meta,
        PendingPeers:    peer.CollectPendingPeers(),
        ApproximateSize: peer.ApproximateSize,
    }
}
```

ConfChange 修改了 Region 的成员，需要立即通知 Scheduler 更新其缓存。否则 Scheduler 可能基于旧信息下发错误的调度指令，或者 Client 请求时 Scheduler 返回"no region"错误。

### 4.5 节点增删完整流程梳理

理解完整的增删节点链路，对面试至关重要。

**RemoveNode 完整链路：**

```
1. 测试/调度器 调用 schedulerClient.RemovePeer(regionID, peer)
   └── 设置 operators[regionID] = RemovePeerOperator

2. 定期心跳：某 Region 的 Leader 发送 RegionHeartbeatRequest
   └── MockSchedulerClient.RegionHeartbeat() 处理
       ├── 发现 operators[regionID] 有待执行的 RemoveNode
       └── 调用 makeRegionHeartbeatResponse 生成响应（ChangePeer: RemoveNode）

3. heartbeatResponseHandler 处理响应
   └── onRegionHeartbeatResponse() 调用 sendAdminRequest
       └── 封装 AdminCmdType_ChangePeer 请求发给 Leader

4. Leader 的 peerMsgHandler 收到 HandleMsg
   └── proposeRaftCommand → ProposeConfChange
       └── Raft 层写入 EntryConfChange Entry

5. Entry 复制到多数节点后 commit
   └── HandleRaftReady → processCommittedEntry → processConfChange
       ├── 所有节点执行：更新 region.Peers，ConfVer++，持久化
       └── 被删节点：调用 destroyPeer()，从集群中移除自己
```

**AddNode 完整链路：**

```
1. 调度器调用 AddPeer → 通过心跳响应下发 AdminCmdType_ChangePeer(AddNode)

2. Leader proposeRaftCommand → ProposeConfChange
   └── 所有节点 apply：region.Peers 中添加新节点 F，ConfVer++，持久化

3. 此时 F 还不存在，Leader 的 Prs 中有 F，会给 F 发心跳
   └── 心跳进入 storeWorker.onRaftMessage()
       └── F 不存在 → maybeCreatePeer()
           ├── 条件：IsInitialMsg（心跳 + Commit == RaftInvalidIndex）
           └── 成立 → replicatePeer() 创建空的 Peer F
               └── 注册到 router，发送 MsgTypeStart 启动

4. F 启动后，Leader 发现其日志落后（LastIndex=0）
   └── 直接发 Snapshot 给 F

5. F 收到 Snapshot，通过 HandleRaftReady → ApplySnapshot
   └── 更新 F.Region() 信息，同步 storeMeta
   └── F 正式加入集群，可以服务请求
```

### 4.6 Region Split 详解

Region Split 是 Multi-Raft 的核心功能，将一个大 Region 分裂成两个小 Region，使系统可以用两个 Raft Group 并行处理请求。

**分裂不迁移数据的原理：**

分裂后两个 Region 仍然共用同一个 Store 的 badger 实例（kvDB），数据在 kvDB 中的 key 本身没有变化。分裂只是修改了元数据（StartKey/EndKey），告知系统哪个 Region 负责哪段 key。

```
分裂前：Region A [0, 100) 在 Store 1 上
kvDB: {key1: v1, key2: v2, ..., key100: v100}（全在一个 badger 中）

分裂后：
Region A [0, 50)  → 负责 key1~key49
Region B [50, 100) → 负责 key50~key100
共用同一 kvDB，没有数据搬移
```

数据真正的迁移发生在 **Region Balance** 时：Scheduler 通过 MovePeer 操作，将某个 Region 的 Peer 从一个 Store 移到另一个 Store，这才会通过 Snapshot 同步数据。

#### 4.6.1 Split 触发流程

```
Step 1: peer_msg_handler.go onTick()
    └── onSplitRegionCheckTick()
        └── 生成 SplitCheckTask，发给 split_checker.go

Step 2: split_checker.go Handle()
    └── 遍历 kvDB 中该 Region 的 key，按字典序找中位 key（SplitKey）
    └── 如果 Region 大小超过阈值（cfg.RegionMaxSize），发送 MsgTypeSplitRegion

Step 3: peer_msg_handler.go HandleMsg()
    └── 收到 MsgTypeSplitRegion
        └── onPrepareSplitRegion()
            └── 发送 SchedulerAskSplitTask 给 scheduler_task.go
                └── 向 Scheduler 申请新的 RegionId 和 PeerIds（保证全局唯一）

Step 4: scheduler_task.go onAskSplit()
    └── 收到 Scheduler 分配的 NewRegionId 和 NewPeerIds
        └── 发起 AdminCmdType_Split 的 RaftCmdRequest 给 Leader

Step 5: 正常 propose → Raft 同步 → apply 流程
```

**SplitKey 的选取：**

`split_checker.go` 遍历 Region 内所有 key，找到能将数据等分的中位 key。遍历是按照 badger 的字典序（不是 key 的数值顺序），所以 SplitKey 按字典序将 Region 一分为二。

#### 4.6.2 Split 应用流程

Split Entry commit 后，在 apply 阶段执行：

```go
case raft_cmdpb.AdminCmdType_Split:
    // Step 1: 检查请求合法性
    if requests.Header.RegionId != d.regionId {
        // ErrRegionNotFound
        d.handleProposal(entry, ErrResp(&util.ErrRegionNotFound{...}))
        return kvWB
    }
    if err := util.CheckRegionEpoch(requests, d.Region(), true); err != nil {
        // RegionEpoch 不匹配（Region 已经分裂过了）
        d.handleProposal(entry, ErrResp(err))
        return kvWB
    }
    if err := util.CheckKeyInRegion(adminReq.Split.SplitKey, d.Region()); err != nil {
        // SplitKey 不在当前 Region 范围内
        d.handleProposal(entry, ErrResp(err))
        return kvWB
    }
    if len(d.Region().Peers) != len(adminReq.Split.NewPeerIds) {
        // NewPeerIds 数量不一致，拒绝（Scheduler 信息过时）
        d.handleProposal(entry, ErrRespStaleCommand(d.Term()))
        return kvWB
    }

    // Step 2: 创建新 Region
    oldRegion := d.Region()
    split := adminReq.Split

    // 为新 Region 创建 Peers（复制 oldRegion 的 Peers，修改 PeerId）
    newPeers := make([]*metapb.Peer, len(oldRegion.Peers))
    for i, peer := range oldRegion.Peers {
        newPeers[i] = &metapb.Peer{
            Id:      split.NewPeerIds[i],
            StoreId: peer.StoreId,
        }
    }
    newRegion := &metapb.Region{
        Id:       split.NewRegionId,
        StartKey: split.SplitKey,           // 新 Region 从 SplitKey 开始
        EndKey:   oldRegion.EndKey,          // 到原 Region 结束
        RegionEpoch: &metapb.RegionEpoch{
            ConfVer: 1,
            Version: 1,
        },
        Peers: newPeers,
    }

    // Step 3: 更新 RegionEpoch
    oldRegion.RegionEpoch.Version++
    newRegion.RegionEpoch.Version = oldRegion.RegionEpoch.Version  // 同步版本号

    // Step 4: 更新 storeMeta（必须加锁！）
    storeMeta := d.ctx.storeMeta
    storeMeta.Lock()
    storeMeta.regionRanges.Delete(&regionItem{region: oldRegion})       // 先删旧范围
    oldRegion.EndKey = split.SplitKey                                    // 修改 oldRegion 范围
    storeMeta.regionRanges.ReplaceOrInsert(&regionItem{region: oldRegion}) // 重新插入
    storeMeta.regionRanges.ReplaceOrInsert(&regionItem{region: newRegion}) // 插入新 Region
    storeMeta.regions[newRegion.Id] = newRegion
    storeMeta.Unlock()

    // Step 5: 持久化
    meta.WriteRegionState(kvWB, oldRegion, rspb.PeerState_Normal)
    meta.WriteRegionState(kvWB, newRegion, rspb.PeerState_Normal)

    // Step 6: 创建新 Region 的 Peer，注册到 router 并启动
    newPeer, err := createPeer(d.storeID(), d.ctx.cfg, d.ctx.schedulerTaskSender, d.ctx.engine, newRegion)
    d.ctx.router.register(newPeer)
    d.ctx.router.send(newRegion.Id, message.Msg{Type: message.MsgTypeStart})

    // Step 7: 处理 proposal 回调
    d.handleProposal(entry, &raft_cmdpb.RaftCmdResponse{
        Header: &raft_cmdpb.RaftResponseHeader{},
        AdminResponse: &raft_cmdpb.AdminResponse{
            CmdType: raft_cmdpb.AdminCmdType_Split,
            Split: &raft_cmdpb.SplitResponse{
                Regions: []*metapb.Region{newRegion, oldRegion},
            },
        },
    })

    // Step 8: 通知 Scheduler 更新缓存（防止 no region 问题）
    if d.IsLeader() {
        d.notifyHeartbeatScheduler(oldRegion, d.peer)
        d.notifyHeartbeatScheduler(newRegion, newPeer)  // 也要通知新 Region！
    }
```

**关键细节：Split Key 顺序问题**

注意更新 storeMeta 时的顺序：

1. 先 `Delete` 旧的 regionRange（删除 oldRegion 的旧范围）
2. 更新 `oldRegion.EndKey = SplitKey`
3. `ReplaceOrInsert` 更新 oldRegion 的新范围
4. `ReplaceOrInsert` 插入 newRegion 的范围

如果先修改 `oldRegion.EndKey` 再 `Delete`，因为 B-tree 的 key 是按 region 的 EndKey 排序的，会导致找不到原来的节点而删除失败，产生 `meta corruption detected` 错误。

### 4.7 Apply 阶段关键注意事项

#### 4.7.1 每个 Entry 执行前检查 RegionEpoch

在 apply 每个 Entry 之前，都应检查 RegionEpoch 是否与当前 Region 匹配：

```go
// 在处理 entry 之前
if err := util.CheckRegionEpoch(req, d.Region(), true); err != nil {
    d.handleProposal(entry, ErrResp(err))
    return kvWB
}
```

**为什么需要这个检查？**

entry 从 propose 到 apply 之间，Region 可能已经发生了 split 或 confchange，导致这个 entry 中的 key 不再属于当前 Region，或者 Region 成员已经变化。尤其是 Snapshot 类型的请求，如果在 Snap entry 执行前 Region 已经 split，那么这个 Snap 覆盖的 key 范围可能已经超出当前 Region 范围，强行执行会导致数据不一致。

#### 4.7.2 SizeDiffHint 更新

```go
case raft_cmdpb.CmdType_Put:
    // Put 时增加 SizeDiffHint
    d.SizeDiffHint += uint64(len(req.Put.Key) + len(req.Put.Value))
    // 执行写入...

case raft_cmdpb.CmdType_Delete:
    // Delete 时减少 SizeDiffHint
    if len(req.Delete.Key) > d.SizeDiffHint {
        d.SizeDiffHint = 0
    } else {
        d.SizeDiffHint -= uint64(len(req.Delete.Key))
    }
    // 执行删除...
```

`SizeDiffHint` 用于触发 split 检查（`onSplitRegionCheckTick`）。如果不更新，Region 大小估算会不准确，导致 split 不被触发，在高并发大量写入的测试中会出现卡死。

#### 4.7.3 processConfChange 后检查 d.stopped

```go
// 处理完 ConfChange Entry 后
d.processConfChange(entry, cc, kvWB)

// 关键：检查是否因为 RemoveNode 自己而停止
if d.stopped {
    return
}

// 继续处理下一个 entry...
```

如果 ConfChange 是 RemoveNode 自己，`destroyPeer()` 会设置 `d.stopped = true`。此时必须立即停止处理后续 entry，否则会在一个已销毁的节点上继续执行操作，产生各种错误。

---

## 5. Scheduler 实现（Project3C）

### 5.1 Scheduler 整体架构

TinyScheduler 对应 TiDB 生态中的 PD（Placement Driver），负责：

1. **全局元数据管理**：维护所有 Region 的位置、大小、Leader 等信息
2. **负载均衡调度**：当 Store 间 Region 数量不均衡时，通过 MovePeer 操作均衡负载
3. **健康状态监控**：通过 Store 心跳监控节点存活状态

Scheduler 通过**心跳**获取集群信息：

- Region 的 Leader 定期发送 `RegionHeartbeatRequest`
- 各 Store 定期发送 `StoreHeartbeatRequest`

Scheduler 通过**心跳响应**下发调度指令：

- `RegionHeartbeatResponse` 中携带 `ChangePeer`（AddNode/RemoveNode）或 `TransferLeader` 指令

### 5.2 processRegionHeartbeat 实现

**函数签名与 RegionInfo 结构：**

```go
// RegionInfo 包含心跳中的所有信息
type RegionInfo struct {
    meta            *metapb.Region   // Region 元数据（ID, 范围, Peers, Epoch）
    learners        []*metapb.Peer
    voters          []*metapb.Peer
    leader          *metapb.Peer     // 当前 Leader
    pendingPeers    []*metapb.Peer   // 待同步的 Peer
    approximateSize int64            // Region 大小估算
}

func (c *RaftCluster) processRegionHeartbeat(region *core.RegionInfo) error {
    // ...
}
```

**过期判断逻辑（两种场景）：**

**场景一：Scheduler 中已有相同 ID 的 Region**

```go
origin := c.core.GetRegion(region.GetID())
if origin != nil {
    // 比较 RegionEpoch：新 epoch 的 Version 或 ConfVer 必须 >= 旧的
    if util.IsEpochStale(region.GetRegionEpoch(), origin.GetRegionEpoch()) {
        return ErrRegionIsStale(region.GetMeta(), origin.GetMeta())
    }
}
```

**场景二：Scheduler 中没有相同 ID 的 Region**

```go
if origin == nil {
    // 扫描所有与该 Region key 范围有 overlap 的 Region
    overlaps := c.core.ScanRegions(region.GetStartKey(), region.GetEndKey(), -1)
    for _, r := range overlaps {
        // 新 Region 必须比所有 overlap Region 都要新
        if util.IsEpochStale(region.GetRegionEpoch(), r.GetRegionEpoch()) {
            return ErrRegionIsStale(region.GetMeta(), r.GetMeta())
        }
    }
}
```

**判断是否需要更新（可跳过条件）：**

以下任何一个条件成立，则不能跳过更新：

```
1. Version 或 ConfVer 大于原来的（Region 发生了 split 或 confchange）
2. Leader 发生了变化
3. 新的或原来的有 pending peers（说明有成员正在追赶日志）
4. ApproximateSize 发生了变化（大小变了，可能触发新的调度）
```

**执行更新：**

```go
// 通过检查后，执行更新
c.core.PutRegion(region)                   // 更新 Region 树
c.core.UpdateStoreStatus(region.GetLeaderStoreId())  // 更新 Store 的负载状态
```

### 5.3 Schedule 实现（balance_region）

**平衡调度的目标：** 将 regionSize 最大的 Store 中的某个 Region，迁移到 regionSize 最小的 Store，使集群负载趋于均衡。

**完整实现流程：**

```go
func (s *balanceRegionScheduler) Schedule(cluster opt.Cluster) *operator.Operator {
    // Step 1: 筛选并排序 suitableStore
    stores := cluster.GetStores()
    var suitableStores []*core.StoreInfo
    for _, store := range stores {
        if store.IsUp() && store.DownTime() < cluster.GetMaxStoreDownTime() {
            suitableStores = append(suitableStores, store)
        }
    }
    if len(suitableStores) < 2 {
        return nil
    }
    // 按 regionSize 降序排列
    sort.Slice(suitableStores, func(i, j int) bool {
        return suitableStores[i].GetRegionSize() > suitableStores[j].GetRegionSize()
    })

    // Step 2: 从 regionSize 最大的 Store 中找待迁移 Region
    var sourceRegion *core.RegionInfo
    var sourceStore *core.StoreInfo
    for _, store := range suitableStores {
        // 按优先级依次查找：pending > follower > leader
        cluster.GetPendingRegionsWithLock(store.GetID(), func(container core.RegionsContainer) {
            sourceRegion = container.RandomRegion(nil, nil)
        })
        if sourceRegion != nil {
            sourceStore = store
            break
        }
        cluster.GetFollowersWithLock(store.GetID(), func(container core.RegionsContainer) {
            sourceRegion = container.RandomRegion(nil, nil)
        })
        if sourceRegion != nil {
            sourceStore = store
            break
        }
        cluster.GetLeadersWithLock(store.GetID(), func(container core.RegionsContainer) {
            sourceRegion = container.RandomRegion(nil, nil)
        })
        if sourceRegion != nil {
            sourceStore = store
            break
        }
    }
    if sourceRegion == nil {
        return nil
    }

    // Step 3: 检查 Region 副本数是否满足最小要求
    if len(sourceRegion.GetPeers()) < cluster.GetMaxReplicas() {
        return nil  // 副本数不足，不能再减少
    }

    // Step 4: 找 regionSize 最小的目标 Store（不能在源 Region 中）
    var targetStore *core.StoreInfo
    sourceRegionStoreIDs := sourceRegion.GetStoreIds()
    for i := len(suitableStores) - 1; i >= 0; i-- {
        store := suitableStores[i]
        if _, ok := sourceRegionStoreIDs[store.GetID()]; !ok {
            targetStore = store
            break
        }
    }
    if targetStore == nil {
        return nil
    }

    // Step 5: 判断迁移是否有价值
    // 差值必须大于 Region 大小的两倍，避免来回迁移
    if sourceStore.GetRegionSize()-targetStore.GetRegionSize() <= 2*sourceRegion.GetApproximateSize() {
        return nil
    }

    // Step 6: 创建 MovePeer Operator
    newPeer, err := cluster.AllocPeer(targetStore.GetID())
    if err != nil {
        return nil
    }
    return operator.CreateMovePeerOperator(
        "balance-region",
        cluster,
        sourceRegion,
        operator.OpBalance,
        sourceStore.GetID(),
        targetStore.GetID(),
        newPeer.GetId(),
    )
}
```

**MovePeer Operator 的执行步骤：**

`CreateMovePeerOperator` 生成一个包含三步的 Operator：

1. `AddPeer(targetStore, newPeer)`：在目标 Store 添加新 Peer
2. `TransferLeader`（如果源 Store 是 Leader）：迁移 Leadership
3. `RemovePeer(sourceStore)`：删除源 Store 上的 Peer

这三步由 Scheduler 逐步通过心跳响应下发，不是同时执行。

---

## 6. Multi-Raft 关键操作流分析

### 6.1 正常读写请求流

```
Client 发起 Put(key, value) 请求
    ↓
TinyKV Server 收到 gRPC 请求
    ↓
根据 key 路由到对应 Region（查 storeMeta.regionRanges 或询问 Scheduler）
    ↓
将请求发给该 Region 的 Leader（通过 router.send）
    ↓
Leader 的 peerMsgHandler.HandleMsg(MsgTypeRaftCmd) 处理
    ↓
proposeRaftCommand → RaftGroup.Propose(data)
    ↓
Raft 层：Leader 将 Entry 写入 RaftLog，广播 MsgAppend 给 Followers
    ↓
Followers 收到 MsgAppend，持久化 Entry，回复 MsgAppendResponse
    ↓
Leader 收到多数 AppendResponse，推进 committedIndex
    ↓
raftWorker 触发 HandleRaftReady
    ↓
apply CommittedEntries：执行 kvWB.SetCF(key, value)，写入 kvDB
    ↓
处理 proposal 回调：cb.Done(response)
    ↓
响应 Client
```

### 6.2 Region Split 操作全流程

```
[触发阶段]
1. onTick() → onSplitRegionCheckTick()
   └── 检查 Region 大小（通过 SizeDiffHint 估算）
       └── 超过阈值 → 发送 SplitCheckTask

2. split_checker.go Handle(SplitCheckTask)
   └── 扫描 kvDB，按字典序找 SplitKey（中位 key）
       └── 发送 MsgTypeSplitRegion(splitKey)

3. HandleMsg(MsgTypeSplitRegion)
   └── onPrepareSplitRegion()
       └── 发送 SchedulerAskSplitTask（申请新 RegionId 和 PeerIds）

4. scheduler_task.go onAskSplit()
   └── Scheduler 分配 NewRegionId 和 NewPeerIds
       └── 发起 AdminCmdType_Split 的 RaftCmdRequest

[Propose 阶段]
5. proposeRaftCommand(AdminCmdType_Split)
   └── 检查 RegionEpoch 和 SplitKey 合法性
       └── RaftGroup.Propose(splitRequest)

[Raft 同步阶段]
6. Leader 广播 Entry，多数节点 commit

[Apply 阶段]
7. HandleRaftReady → processCommittedEntry → processSplit
   ├── 创建 newRegion（StartKey=SplitKey, EndKey=oldRegion.EndKey）
   ├── 更新 oldRegion.EndKey = SplitKey
   ├── 两个 Region 的 Version++
   ├── 持久化（WriteRegionState）
   ├── 更新 storeMeta.regionRanges（加锁）
   ├── createPeer(newRegion) → router.register → MsgTypeStart
   └── notifyHeartbeatScheduler（两次，分别通知新旧 Region）

[新 Region 选 Leader]
8. 新 Region 的 Peer 启动后，开始 Raft 选举
   └── 选出 Leader 后，开始正常服务请求
```

### 6.3 ConfChange Add 完整流程

```
[调度器下发]
1. Schedule() 决定向某 Store 添加一个 Region 的 Peer
   └── 生成 AddPeer Operator
       └── 通过下一次 Region 心跳响应下发 ChangePeer(AddNode)

[Propose 到 Apply]
2. Leader 收到 AdminCmdType_ChangePeer(AddNode)
   └── 检查 AppliedIndex >= PendingConfIndex
       └── ProposeConfChange(AddNode, newPeerID)

3. Raft 同步后 commit，所有节点 Apply
   └── 每个节点：region.Peers 中添加新 Peer，ConfVer++，持久化
       └── ApplyConfChange → addNode（更新 Prs）

[新节点创建]
4. Leader 向新 PeerID 发送心跳（msg.Commit = RaftInvalidIndex）
   └── storeWorker.onRaftMessage → maybeCreatePeer
       └── IsInitialMsg 判断通过 → replicatePeer() 创建空 Peer
           └── router.register → MsgTypeStart 启动

[数据同步]
5. Leader 发现新节点 Match=0，LastIndex 落后
   └── 直接发 Snapshot 给新节点（跳过逐条 AppendEntry）

6. 新节点 HandleRaftReady 处理 Snapshot
   └── ApplySnapshot → 更新 Region 信息、storeMeta
       └── 新节点正式加入集群，开始参与 Raft 投票
```

### 6.4 ConfChange Remove 完整流程

```
1. Schedule() 决定移除某 Store 上的 Region Peer
   └── 生成 RemovePeer Operator → 心跳响应下发 ChangePeer(RemoveNode)

2. Leader proposeRaftCommand(ChangePeer, RemoveNode)
   └── 特殊检查：若 2 节点且 Remove 的是 Leader
       └── 先 TransferLeader，等 Leader 迁移后再 Remove

3. Entry commit，所有节点 Apply processConfChange
   ├── 非目标节点：region.Peers 中删除该 Peer，ConfVer++，removePeerCache
   └── 目标节点（被删除的）：destroyPeer()
       ├── 清理 raftDB 中的 Region 信息
       ├── 清理 storeMeta（regions, regionRanges）
       ├── 关闭 router 中的 Peer
       └── d.stopped = true

4. destroyPeer 后不再处理任何 entry（检查 d.stopped）
```

### 6.5 Leader Transfer 操作流

```
1. 调度器通过心跳响应下发 TransferLeader 指令

2. Leader 收到 AdminCmdType_TransferLeader
   └── 直接调用 RaftGroup.TransferLeader(targetPeerID)
       └── Step(MsgTransferLeader{From: targetPeerID})

3. Raft 层处理 MsgTransferLeader
   ├── 若目标日志已同步：直接发 MsgTimeoutNow
   └── 若目标日志落后：发 MsgAppend，等 AppendResponse 后再发 TimeoutNow

4. 目标节点收到 MsgTimeoutNow
   └── 立即 Step(MsgHup)，发起选举

5. 新的 Leader 选举成功（term+1，日志最新）
   └── 原 Leader 收到更高 term 的消息，退化为 Follower

6. 新 Leader 就位，上层收到 AdminCmdType_TransferLeader 的 resp
```

### 6.6 Region Balance 操作流

```
[调度决策]
1. Scheduler 定期调用 balanceRegionScheduler.Schedule()
   └── 选出负载最重的 Store（sourceStore）中的一个 Region
       └── 选出负载最轻的 Store（targetStore）

2. 生成 MovePeer Operator：[AddPeer, TransferLeader?, RemovePeer]

[执行 AddPeer]
3. 通过心跳响应下发 ChangePeer(AddNode) 给源 Region 的 Leader
   └── ... 执行 ConfChange AddNode 流程（见 6.3）
       └── 新 Peer 在 targetStore 上创建并同步数据

[执行 TransferLeader（如需）]
4. 若源 Store 是 Leader，下发 TransferLeader 到 targetStore 上的新 Peer
   └── ... 执行 Leader Transfer 流程（见 6.5）

[执行 RemovePeer]
5. 下发 ChangePeer(RemoveNode) 删除 sourceStore 上的 Peer
   └── ... 执行 ConfChange RemoveNode 流程（见 6.4）

[完成]
6. Region 成功从 sourceStore 迁移到 targetStore
   └── 数据通过 Snapshot 完成同步
```

---

## 7. 设计分析与关键决策

### 7.1 为何使用单步变更而非 Joint Consensus

Raft 论文提出了 Joint Consensus 算法，支持一次性变更多个节点，但 TinyKV 选择了更简单的**单步变更**（一次只增删一个节点）。

**Joint Consensus 的问题：**

- 需要两阶段提交（C_old,new 阶段 → C_new 阶段）
- 实现复杂，边界条件多，容易出 Bug
- 对于一般场景（增删单个节点）没有必要

**单步变更的安全性保证：**

考虑 3 节点集群 [A, B, C]，AddNode D 后变成 4 节点：

- 3 节点时 quorum = 2，4 节点时 quorum = 3
- 单步变更保证：在 [A, B, C] 阶段的 quorum（2 票）和 [A, B, C, D] 阶段的 quorum（3 票）之间，任意一个多数派都包含至少一个公共节点
- 因此不会出现两个独立的多数派同时认可不同的 Leader

**为什么需要 PendingConfIndex？**

如果允许多个 ConfChange 并发（比如先 AddNode D，还没 apply 就又 RemoveNode B），可能出现一系列 ConfChange 相互干扰。`PendingConfIndex` 确保：前一个 ConfChange 完全 apply 后（已经修改了集群成员），才允许发起下一个 ConfChange。

### 7.2 RegionEpoch 双版本的设计意图

`RegionEpoch` 有两个独立的版本号：

- `ConfVer`：成员变更版本（AddNode/RemoveNode 时递增）
- `Version`：分片版本（Split/Merge 时递增）

**为什么分开？**

网络分区场景下，可能有两个旧 Leader 分别处理了不同类型的操作：

- 分区 1 的旧 Leader 执行了 ConfChange（ConfVer 更大）
- 分区 2 的旧 Leader 执行了 Split（Version 更大）

如果只用一个版本号，这两种情况的"谁更新"就难以判断。分开后，Scheduler 可以分别比较：

- `Version` 相同但 `ConfVer` 更大 → 成员变更更新
- `ConfVer` 相同但 `Version` 更大 → 分片更新
- 两者都更大才是整体上更新的

判断标准（`IsEpochStale`）：

```go
func IsEpochStale(epoch *metapb.RegionEpoch, checkEpoch *metapb.RegionEpoch) bool {
    return epoch.GetVersion() < checkEpoch.GetVersion() ||
        epoch.GetConfVer() < checkEpoch.GetConfVer()
}
```

只要任一维度落后，就认为是过时的。

### 7.3 心跳双重机制的必要性

**Raft 心跳**（周期约 150ms）：

- 目的：维持 Leader 地位，防止 Follower 超时发起选举
- 只在 Region 内部的 Peer 之间传递
- 不经过 Scheduler

**Scheduler 心跳**（周期约 10s，比 Raft 心跳慢得多）：

- 目的：让 Scheduler 感知集群状态
- 由 Region Leader 定期发给 Scheduler
- 携带 Region 的完整元数据（大小、Peers 列表、Leader 等）

两者解决不同问题，缺一不可：

- 没有 Raft 心跳 → Follower 频繁发起选举，集群不稳定
- 没有 Scheduler 心跳 → Scheduler 无法感知集群状态，无法做负载均衡

### 7.4 Split 不迁移数据的设计

Split 操作只修改 Region 的元数据（StartKey/EndKey/RegionEpoch），不移动 kvDB 中的任何 KV 数据。

**优点：**

- Split 操作极快（毫秒级），只需修改元数据
- 不会因大量数据拷贝而产生 IO 压力
- 可以频繁 Split，精细化管理 Region 粒度

**代价：**

- 同一 Store 上的多个 Region 共用 kvDB 存储，数据并非物理隔离
- 如果某个 key 范围的访问热点在一个 Store 上，即使 Split 了也无法减少该 Store 的 IO 压力（需要配合 balance 操作将 Region 迁移到其他 Store）

**数据真正迁移的时机：**

当 Scheduler 决定将某 Region 的 Peer 从 Store A 迁移到 Store B 时，通过 AddPeer 操作，会触发 Snapshot 传输，将该 Region 的所有数据从 Leader 发给 Store B 上的新 Peer。这才是真正的数据迁移。

### 7.5 storeMeta 的一致性保证

`storeMeta` 是全局共享的 Region 元数据，多个 goroutine（raftWorker、storeWorker 等）会并发读写：

```go
type storeMeta struct {
    sync.RWMutex
    regions      map[uint64]*metapb.Region
    regionRanges *btree.BTree
}
```

**必须加锁的场景：**

- 修改 `regions` map（Split 时插入新 Region）
- 修改 `regionRanges` B-tree（Split 时更新 key 范围）
- 读取 `regions` 或 `regionRanges`（路由请求时）

如果 Split 时忘记加锁，可能导致并发读写 B-tree 产生数据竞争，产生 panic 或数据损坏。

### 7.6 调度器 Operator 模式设计

Scheduler 不直接向 Peer 发送命令，而是生成 **Operator**（操作序列），通过心跳响应下发：

```
Operator = [Step1: AddPeer, Step2: TransferLeader, Step3: RemovePeer]
```

每次 Region 发来心跳时，Scheduler 检查是否有未完成的 Operator，如果有，下发当前步骤。当前步骤完成后（Scheduler 通过下一次心跳确认），再下发下一步。

**这种设计的优点：**

1. **幂等性**：每步操作可以安全重试（心跳可能超时）
2. **可观测性**：每个步骤的完成状态可以通过心跳确认
3. **解耦**：Scheduler 不需要维持与 Peer 的长连接

---

## 8. 常见问题与注意事项

### 8.1 Project3A 常见问题

**问题一：Follower 收到 MsgTransferLeader 不处理**

- **现象**：TestTransferLeaderToUpToDateNode3A 测试超时
- **原因**：文档说 MsgTransferLeader 是本地消息，不通过网络传播，但测试中会将其发给 Follower
- **解决**：Follower 收到 MsgTransferLeader 时，将其转发给自己的 Leader

```go
case pb.MessageType_MsgTransferLeader:
    if r.State != StateLeader {
        if r.Lead != None {
            m.To = r.Lead
            r.msgs = append(r.msgs, m)
        }
        return nil
    }
    // Leader 处理 Transfer...
```

**问题二：addNode 重复节点抛异常**

- **现象**：TestRawNodeProposeAddDuplicateNode3A 测试 panic
- **原因**：添加已存在节点时抛异常
- **解决**：如果节点已存在，直接返回，不报错

```go
func (r *Raft) addNode(id uint64) {
    if _, ok := r.Prs[id]; ok {
        return  // 已存在，忽略
    }
    r.Prs[id] = &Progress{Next: r.RaftLog.LastIndex() + 1}
    r.PendingConfIndex = None
}
```

**问题三：removeNode 后没有重新计算 commit**

- **现象**：TestCommitAfterRemoveNode3A 测试失败，log entry 无法 commit
- **原因**：removeNode 后 quorum 变小，之前因为节点不够而无法 commit 的 entry 现在可以 commit 了
- **解决**：removeNode 后调用 maybeCommit，如果 commit 推进了，需要 bcastAppend 广播

```go
func (r *Raft) removeNode(id uint64) {
    delete(r.Prs, id)
    r.PendingConfIndex = None
    if r.State == StateLeader {
        if r.maybeCommit() {
            r.bcastAppend()
        }
    }
}
```

### 8.2 Project3B 常见问题

**问题一：meta corruption detected**

- **现象**：`panic: [region X] meta corruption detected`
- **原因一**：Split 时更新 storeMeta 顺序错误——先修改了 oldRegion.EndKey 再从 B-tree 删除，导致 B-tree 找不到对应节点
- **解决一**：必须先 Delete 旧节点（基于旧 EndKey），然后再修改 EndKey，再 Insert

```go
storeMeta.regionRanges.Delete(&regionItem{region: oldRegion})  // 先删（此时 EndKey 还是旧值）
oldRegion.EndKey = split.SplitKey                               // 再修改
storeMeta.regionRanges.ReplaceOrInsert(&regionItem{region: oldRegion})  // 再插入
```

- **原因二**：AddNode 时没有更新 regionRanges
- **解决二**：AddNode apply 后需要在 storeMeta 中插入新 Region（如果是新建的话）

```go
// 在 maybeCreatePeer 中，或者 addNode apply 后
meta.Lock()
meta.regionRanges.ReplaceOrInsert(&regionItem{region: peer.Region()})
meta.Unlock()
```

**问题二：Remove Leader 两节点死循环**

- **现象**：两节点集群，要 Remove 的正好是 Leader，unreliable 网络下，另一个节点永远无法完成选举
- **根因**：
  1. Leader apply Remove 自己后调用 destroyPeer，不再存在
  2. 另一个节点因为没收到最后的心跳（commit 没推进），不知道 ConfChange 已经 commit
  3. 另一个节点发起选举，但需要 2 票（quorum），自己只有 1 票，因为它还认为集群有两个节点
  4. 死循环：永远无法获得多数票

- **解决一（Propose 阶段）**：在 Propose 时检测到这种情况，拒绝并先 TransferLeader

```go
if len(d.Region().Peers) == 2 &&
    cc.ChangeType == pb.ConfChangeType_RemoveNode &&
    cc.Peer.Id == d.PeerId() {
    // 先转移 Leader，让 client 重试
    d.RaftGroup.TransferLeader(otherPeerID)
    return  // 不 propose，让 client 重试
}
```

- **解决二（Apply 阶段）**：在 destroyPeer 前重复发送多次心跳给对方，尽量确保对方 commit

```go
func (d *peerMsgHandler) startToDestroyPeer() {
    if len(d.Region().Peers) == 2 && d.IsLeader() {
        // 找到另一个节点
        for _, peer := range d.Region().Peers {
            if peer.Id != d.PeerId() {
                heartbeat := pb.Message{
                    To:      peer.Id,
                    MsgType: pb.MessageType_MsgHeartbeat,
                    Commit:  d.peerStorage.raftState.HardState.Commit,
                }
                for i := 0; i < 10; i++ {
                    d.Send(d.ctx.trans, []pb.Message{heartbeat})
                    time.Sleep(100 * time.Millisecond)
                }
                break
            }
        }
    }
    d.destroyPeer()
}
```

**问题三：d.stopped 判断缺失**

- **现象**：RemoveNode 自己后，继续处理后续 Entry，产生各种空指针或状态错误
- **解决**：processConfChange 执行后立即检查 d.stopped

```go
for _, entry := range committedEntries {
    kvWB = d.processCommittedEntry(&entry, kvWB)
    if d.stopped {
        return  // 已销毁，不再处理
    }
}
```

**问题四：no region 问题**

- **现象**：Client 请求某个 key，Scheduler 返回"no region"
- **原因**：Split 或 ConfChange 后，Scheduler 缓存没有及时更新
- **解决**：在 Split 和 ConfChange apply 后，主动调用 `notifyHeartbeatScheduler`

```go
// ConfChange apply 后
if d.IsLeader() {
    d.notifyHeartbeatScheduler(region, d.peer)
}

// Split apply 后
if d.IsLeader() {
    d.notifyHeartbeatScheduler(oldRegion, d.peer)
    d.notifyHeartbeatScheduler(newRegion, newPeer)  // 新 Region 也要通知！
}
```

**问题五：NewPeerIds 数量与 Region Peers 数量不一致**

- **现象**：Split apply 时，`len(split.NewPeerIds) != len(region.Peers)`
- **根因**：可能 Scheduler 在 Region 成员变更途中（还没完成），就收到了旧 Leader 的 Split 请求，导致 NewPeerIds 的数量基于旧的成员信息
- **解决**：检查数量，不一致则拒绝

```go
if len(d.Region().Peers) != len(adminReq.Split.NewPeerIds) {
    d.handleProposal(entry, ErrRespStaleCommand(d.Term()))
    return kvWB
}
```

**问题六：重复 ConfChange 命令**

- **现象**：`panic: should only one conf change`
- **原因**：测试会多次提交同一 ConfChange 直到被 apply，如果重复执行会导致节点被重复添加/删除
- **解决**：apply 时检查 RegionEpoch 是否过期（已 apply 过的 ConfChange 会更新 ConfVer，重复提交的请求携带旧 ConfVer）

```go
if err := util.CheckRegionEpoch(msg, region, true); err != nil {
    d.handleProposal(entry, ErrResp(err))
    return kvWB
}
```

**问题七：心跳 msg.Commit 必须为 RaftInvalidIndex**

- **现象**：AddNode 后，新节点永远无法被创建
- **原因**：`IsInitialMsg` 要求心跳的 Commit 字段为 `RaftInvalidIndex`（0），才会触发 `maybeCreatePeer`
- **解决**：Leader 向不存在的节点发心跳时，Commit 字段设为 0

```go
// 在 sendHeartbeat 中
if pr.Match == 0 {
    // 对方还没有数据，Commit 设为 RaftInvalidIndex
    m.Commit = 0
}
```

**问题八：storeMeta 加锁缺失**

- **现象**：偶现数据竞争 panic 或数据不一致
- **解决**：所有修改 storeMeta 的操作（regionRanges、regions map）都必须加锁

**问题九：Snap entry 执行前 RegionEpoch 检查**

- **现象**：Scan 操作返回 key 不在 region 的错误
- **原因**：Region Split 后，旧的 Snap entry 中的 key 范围可能已超出当前 Region
- **解决**：执行任何 entry 前都检查 RegionEpoch

**问题十：SizeDiffHint 未更新**

- **现象**：高并发大量写入测试（`nclient>=8 && crash=true && split=true`）卡死
- **原因**：Put/Delete 时没更新 SizeDiffHint，导致 Region 大小估算错误，split 不被触发
- **解决**：Put 时 `SizeDiffHint += len(key) + len(value)`，Delete 时 `SizeDiffHint -= len(key)`

### 8.3 Project3C 常见问题

**问题一：region store 数量未检查**

- **现象**：调度器 panic 或 operator 执行异常
- **原因**：如果 Region 的副本数已经小于 maxReplicas，不能再 RemovePeer
- **解决**：

```go
if len(sourceRegion.GetPeers()) < cluster.GetMaxReplicas() {
    return nil
}
```

**问题二：遍历方向错误**

- **现象**：Schedule 没有效果，或者在 regionSize 差距大时不触发 balance
- **原因**：Source store 应从大到小遍历，target store 应从小到大遍历
- **解决**：确认排序方向正确（降序排列后，前面是大的，后面是小的）

---

## 9. 性能与一致性分析

### 9.1 Multi-Raft 并发带来的性能提升

**单 Raft 的瓶颈：**

- 所有写请求串行提交，吞吐量上限约等于单 Raft Group 的处理速度
- 写操作必须等 entry commit，commit 需要等多数节点确认（涉及网络往返）

**Multi-Raft 的提升：**

- 不同 Region 的写请求并行处理，互不阻塞
- 假设 N 个 Region 均匀分布，理论吞吐量提升 N 倍
- 不同 Region 的 Leader 可以在不同 Store 上，充分利用多机 CPU 和 IO 资源

**限制：**

- 跨 Region 的事务需要 2PC（TiKV 通过 Percolator 模型实现），有额外开销
- Region 数量过多会增加 Scheduler 的心跳处理压力

### 9.2 Region Balance 的负载均衡效果

**不平衡的来源：**

- 初始只有一个 Region，所有写入都由一个 Raft Group 处理
- Split 后，新 Region 仍在同一 Store 上，没有物理分散
- 需要 balance 操作将 Region 均匀分散到各 Store

**Balance 的判断标准：**

```
diff = sourceStore.regionSize - targetStore.regionSize
如果 diff > 2 * region.ApproximateSize，才执行迁移
```

这个 `2x` 的系数是为了防止迁移后又触发反向迁移（乒乓效应）：

```
迁移 Region R（大小 S）之后：
sourceStore 减少 S，targetStore 增加 S
新的 diff = (originalDiff - 2S)

如果 originalDiff > 2S，则新 diff > 0（source 仍然大于 target），不会反向迁移
```

### 9.3 线性一致性保证

Multi-Raft 下，TinyKV 如何保证线性一致性（Linearizability）？

**写请求**：所有写请求必须经过 Raft 日志提交，保证全局有序。同一 Region 内的写请求严格按照 commit 顺序执行，天然线性一致。

**读请求**：如果允许 Follower 直接读，可能读到过期数据（Follower 可能落后）。TinyKV 将读请求也走 Raft 日志（Log Read），确保线性一致，但牺牲了读性能。

**优化方案（TiKV 生产实现）：**

- **ReadIndex**：Leader 收到读请求后，不走日志，而是记录当前 commitIndex，等待 apply 到该 Index 后再响应。需要向多数节点确认自己还是 Leader（防止脑裂）。
- **LeaseRead**：Leader 在租约期内无需确认就可以响应读请求（基于时钟假设：租约内不会有新 Leader 产生）。延迟最低，但依赖时钟精度。

TinyKV 实现中未实现 ReadIndex/LeaseRead 优化，读请求和写请求一样走完整的 Raft 日志提交。

---


## 10. 代码索引附录

### 关键文件与函数

| 文件                                            | 关键函数                   | 功能                             |
| ----------------------------------------------- | -------------------------- | -------------------------------- |
| `raft/raft.go`                                  | `handleTransferLeader`     | 处理 Leader Transfer 请求        |
| `raft/raft.go`                                  | `handleTimeoutNow`         | 目标节点收到 TimeoutNow 立即选举 |
| `raft/raft.go`                                  | `addNode`                  | 添加节点到 Prs                   |
| `raft/raft.go`                                  | `removeNode`               | 删除节点，重算 commit            |
| `raft/rawnode.go`                               | `TransferLeader`           | 发起 Leader Transfer             |
| `raft/rawnode.go`                               | `ProposeConfChange`        | 提交 ConfChange Entry            |
| `raft/rawnode.go`                               | `ApplyConfChange`          | 应用 ConfChange，更新 Prs        |
| `kv/raftstore/peer_msg_handler.go`              | `proposeRaftCommand`       | 处理 4 种 AdminCmd 的 Propose    |
| `kv/raftstore/peer_msg_handler.go`              | `processConfChange`        | Apply ConfChange Entry           |
| `kv/raftstore/peer_msg_handler.go`              | `processSplit`             | Apply Split Entry                |
| `kv/raftstore/peer_msg_handler.go`              | `notifyHeartbeatScheduler` | 通知 Scheduler 更新 Region 缓存  |
| `kv/raftstore/store_worker.go`                  | `onRaftMessage`            | 处理来自其他节点的 Raft 消息     |
| `kv/raftstore/store_worker.go`                  | `maybeCreatePeer`          | 自动创建不存在的 Peer            |
| `kv/raftstore/router.go`                        | `register` / `send`        | 注册/发消息给 Region Peer        |
| `kv/raftstore/runner/split_checker.go`          | `Handle`                   | 扫描 Region 数据找 SplitKey      |
| `kv/raftstore/runner/scheduler_task.go`         | `onAskSplit`               | 向 Scheduler 申请 Split ID       |
| `scheduler/server/cluster.go`                   | `processRegionHeartbeat`   | 处理 Region 心跳，更新调度器缓存 |
| `scheduler/server/schedulers/balance_region.go` | `Schedule`                 | Region Balance 调度算法          |

### 关键常量与阈值

| 常量                      | 值   | 作用                                            |
| ------------------------- | ---- | ----------------------------------------------- |
| `RaftInvalidIndex`        | 0    | 标识无效 Index，心跳触发 maybeCreatePeer 的条件 |
| `cfg.RegionMaxSize`       | 配置 | Region 大小超过此值触发 Split                   |
| `cfg.RaftLogGcCountLimit` | 配置 | Raft 日志条数超过此值触发 CompactLog            |
| `MaxStoreDownTime`        | 配置 | Store 不可用超过此时间不参与调度                |
| `2 × ApproximateSize`     | 动态 | Region balance 有价值的最小 diff 阈值           |

### 关键 Proto 结构

```protobuf
// Region 元信息
message Region {
    uint64 id = 1;
    bytes start_key = 2;
    bytes end_key = 3;
    RegionEpoch region_epoch = 4;
    repeated Peer peers = 5;
}

// Region 版本信息
message RegionEpoch {
    uint64 conf_ver = 1;  // ConfChange 版本
    uint64 version = 2;   // Split 版本
}

// 成员变更
message ConfChange {
    ConfChangeType change_type = 1;  // AddNode / RemoveNode
    uint64 node_id = 2;
    bytes context = 3;  // 序列化的 RaftCmdRequest
}
```

---
