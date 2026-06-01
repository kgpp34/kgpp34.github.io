# Raft Leader Election 原理解析

这篇文章主要想系统梳理一下 Raft Leader Election（Leader 选举）的核心原理。

在分布式系统领域，[Raft](https://raft.github.io/raft.pdf "In Search of an Understandable Consensus Algorithm(Extended Version)") 是一个非常经典的共识算法。相比 Paxos，Raft 最大的特点是“可理解性（Understandability）”，它将复杂的共识问题拆分为了几个相对独立的子问题：

* Leader Election（Leader 选举）
* Log Replication（日志复制）
* Safety（安全性保证）
* Membership Change（成员变更）

其中，Leader Election 是整个 Raft 的起点，因为在 Raft 中，整个集群只能存在一个节点负责“做决定”，这个节点就是 Leader。

---

# 1. 为什么需要 Leader

一个分布式系统想要对外提供服务，往往需要由多个节点组成一个分布式集群。而一旦系统进入分布式环境后，就必须面对一个核心问题：如何保证多个节点之间的数据一致性。

例如：

```text
Client A: set x = 1
Client B: set x = 2
```

如果多个节点都能同时处理写请求，那么网络延迟、消息乱序、节点宕机、网络分区等问题，都可能导致各节点最终状态出现分歧，这就是经典的分布式一致性（Consensus）问题。

Raft 对这个问题的解决方案是使用复制状态机（Replicated State Machine）模型。它并不直接同步“状态”，而是同步操作日志（Replicated Log）：

```text
set x = 1
set y = 2
delete z
```

只要所有节点：

* 拥有相同日志
* 按相同顺序执行日志

那么最终状态一定一致。

但这里有一个关键问题：谁来决定日志顺序？

如果多个节点都能同时向日志中写入数据，就会产生冲突。因此，Raft 做了一个非常重要的设计：整个集群只允许一个节点负责管理日志，这个节点就是 Leader。

所有客户端写请求都必须先发送给 Leader，再由 Leader 统一协调日志复制。Follower 不允许主动写日志，只负责复制 Leader 的日志。这样做的好处是：

* 日志顺序唯一
* 不存在并发写冲突
* 集群行为更容易推导
* 一致性模型更清晰

---

# 2. Leader 的核心职责

Leader 本质上是整个集群的日志协调者，其核心职责包括：

* 接收客户端请求
* 将日志追加到本地 WAL（Write Ahead Log）
* 向 Follower 复制日志（AppendEntries RPC）
* 推进 Commit Index
* 将日志应用到状态机
* 定期发送 Heartbeat

其中，Heartbeat 本质上也是一种特殊的 AppendEntries RPC，只不过：

```go
entries = []
```

即不携带实际日志。

Heartbeat 的作用本质上是维持 Leadership。Follower 只要持续收到 Heartbeat，就不会触发 Election Timeout，也不会发起新的选举。

---

# 3. 为什么需要 Leader Election

Leader 是整个集群唯一的决策中心，因此其存活状态直接决定系统是否还能继续处理写请求。

一旦 Leader 崩溃：

* Follower 无法继续提交新日志
* 集群无法处理新的写请求
* 系统进入不可写状态

因此，Raft 必须能够在 Leader 故障后快速重新选出新的 Leader，这也是 Leader Election 存在的根本原因。

---

# 4. 节点角色与核心概念

Raft 中共有三种角色：

| 角色        | 说明           |
| --------- | ------------ |
| Follower  | 被动状态，只响应 RPC |
| Candidate | 发起选举时的过渡状态   |
| Leader    | 负责日志复制与集群协调  |

节点状态转换如下图所示：

![Raft角色状态转换](https://files.mdnice.com/user/203138/67ab8994-db6b-4cc5-921b-da91fb369e44.png)

除了角色之外，Raft 还引入了一个非常核心的概念：Term（任期）。

Term 本质上是一个逻辑时钟（Logical Clock），Raft 使用 Term 来区分不同轮次的 Leader Election，并判断消息是否过期。

其核心规则如下：

* Term 单调递增
* 每次发起新选举时 term++
* 每个节点都会持久化 currentTerm
* 所有 RPC 都会携带 term

例如：

```go
RequestVote(term=5)
AppendEntries(term=5)
```

如果某个节点收到：

```go
term > currentTerm
```

那么它会立刻：

* 更新 currentTerm
* 退回 Follower 状态

反之，如果收到：

```go
term < currentTerm
```

则直接拒绝该 RPC。

---

# 5. Raft 中的两种核心 RPC

Raft Leader Election 主要依赖两种 RPC：

| RPC           | 发起方       | 作用              |
| ------------- | --------- | --------------- |
| RequestVote   | Candidate | 请求投票            |
| AppendEntries | Leader    | 日志复制与 Heartbeat |

其中：

* RequestVote 用于发起选举
* AppendEntries 用于复制日志以及维持 Leadership

---

# 6. Leader Election 流程

### 6.1 Election Timeout

每个 Follower 都会维护一个 Election Timeout。只要 Follower 能持续收到 Leader 的 Heartbeat：

```go
AppendEntries(entries=[])
```

它就会不断重置 Election Timeout。

但如果长时间收不到 Heartbeat，Follower 就会认为当前 Leader 已失联，并进入 Candidate 状态：

```text
Follower -> Candidate
```

随后发起新一轮选举。

Raft 论文推荐的 Election Timeout 范围为：

```text
150ms ~ 300ms
```

并要求满足：

```
broadcastTime < electionTimeout << MTBF
```

即：

* Election Timeout 必须远大于网络广播时间
* 同时远小于节点平均故障时间（MTBF）

否则：

* Timeout 过小容易误触发选举
* Timeout 过大则会导致 Leader 故障恢复过慢

---

### 6.2 发起选举

当 Follower 超时后，会执行以下操作：

```go
currentTerm++
state = Candidate
voteFor = self
resetElectionTimeout()
```

随后，Candidate 会向其他所有节点发送：

```go
RequestVote RPC
```

请求其他节点给自己投票。

---

### 6.3 投票规则

收到 RequestVote 的节点（Voter）会根据以下规则决定是否投票。

首先，如果：

```go
candidate.term < currentTerm
```

说明 Candidate 已经过期，直接拒绝。

否则，还需要同时满足以下条件：

#### 条件一：当前 Term 尚未投票

即：

```go
votedFor == nil
```

或者：

```go
votedFor == candidate
```

Raft 要求每个节点在同一个 Term 内最多只能投一票，这是 Election Safety 的基础。

#### 条件二：Candidate 的日志至少和自己一样新

Raft 并不是“谁先发起选举谁就能成为 Leader”。

Follower 只会投票给日志至少不落后于自己的 Candidate，因为新 Leader 必须拥有所有已提交日志。这也是 Raft Log Safety 的关键基础。

---

# 7. Leader Election 的三种结果

### 7.1 赢得选举

如果 Candidate 获得多数派（Majority）节点支持：

```
⌊N/2⌋ + 1
```

那么：

```text
Candidate -> Leader
```

随后，Leader 会立刻向所有节点广播 Heartbeat：

```go
AppendEntries(entries=[])
```

以阻止其他节点继续发起选举。

---

### 7.2 发现合法 Leader

如果 Candidate 在等待投票期间收到了某个 Leader 发来的 AppendEntries，并且：

```go
leader.term >= currentTerm
```

则说明当前集群已经存在合法 Leader。

于是：

```text
Candidate -> Follower
```

停止当前选举。

---

### 7.3 Split Vote（选票分裂）

Split Vote 是 Raft Leader Election 中一个非常经典的问题。

假设多个节点几乎同时超时：

```text
Node A timeout
Node B timeout
```

那么 A 和 B 都会同时进入 Candidate 状态：

* A 给自己投票
* B 给自己投票
* 其他节点投票被瓜分

最终：

```text
没有任何 Candidate 获得 Majority
```

整个集群进入 Split Vote 状态。

---

# 8. Raft 如何解决 Split Vote

Raft 的解决方案非常经典：Randomized Election Timeout（随机化选举超时）。

即每个节点的 Election Timeout 并不固定，而是在一个区间内随机选择：

```text
150ms ~ 300ms
```

这样大多数情况下，总会有一个节点更早超时，从而：

* 更早发起选举
* 更早获得多数票
* 更快成为 Leader

因此，Split Vote 的概率会显著下降。

---

# 9. Leader Election 的安全性保证

Raft Leader Election 的安全性主要体现在两个方面。

### Election Safety

即每个 Term 最多只能存在一个 Leader。

这是通过：

* Majority 投票
* 每个节点每个 Term 最多投一票

共同保证的。

因为两个不同 Candidate 不可能同时获得多数票。

### Leader Completeness

即新 Leader 必然拥有所有已提交日志。

这是通过“日志新旧检查”实现的。由于：

* 已提交日志一定存在于多数派节点
* Leader Election 同样需要多数派支持

因此两个 Majority 必然存在交集，而这个交集节点只会投票给日志不落后的 Candidate，因此新 Leader 不会丢失已提交日志。

---

# 10. 总结

Raft Leader Election 的核心思想并不复杂，其本质是通过 Majority 投票机制，在任意时刻只允许一个节点成为 Leader。

整个过程主要依赖：

* Election Timeout
* RequestVote RPC
* Term
* Majority

共同完成。

不过需要注意的是，论文中的 Raft Leader Election 更多是一个“理想化模型”。在真实工程环境中，网络分区、GC Pause、时钟漂移、磁盘 IO 抖动以及 Slow Node 等问题，都会对选举过程产生影响。

因此，今天工业界中的 etcd、TiKV、Consul 等系统，实际上都对原始 Raft 做了大量工程增强。

下一篇文章中，我们将进一步分析：

> 为什么一个已经失联很久的节点，重新加入集群后，反而可能触发整个 Raft 集群重新选举。

以及：

> PreVote 为什么会成为工业级 Raft 的标配。
