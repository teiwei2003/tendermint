# ADR 068:反向同步

## 变更日志

- 2021 年 4 月 20 日:初稿 (@cmwaters)

## 状态

公认

## 语境

状态同步和区块剪枝的出现为全节点参与共识提供了机会，而无需完整的区块历史。这也带来了证据处理方面的问题。没有在证据时代拥有所有区块的节点无法验证证据，因此如果该证据在链上提交就会停止。

[RFC005](https://github.com/tendermint/spec/blob/master/rfc/005-reverse-sync.md) 是针对此问题发布的，并修改了规范以添加最小区块历史不变量。这主要是为了扩展状态同步，以便它能够获取和存储最后一个“n”高度的“Header”、“Commit”和“ValidatorSet”(本质上是一个“LightBlock”)，其中“n”是基于计算的从证据时代。

此 ADR 着手描述此状态同步扩展的设计以及对轻客户端提供程序的修改和 tm 存储的合并。

## 决定

状态同步反应器将通过引入 2 个新的 P2P 消息(和一个新通道)进行扩展。

```protobuf
message LightBlockRequest {
  uint64 height = 1;
}

message LightBlockResponse {
  tendermint.types.LightBlock light_block = 1;
}
```

这将由“反向同步”协议使用，该协议将获取、验证和存储先前的轻块，以便节点可以安全地参与共识。

此外，这允许新的轻客户端提供程序为 `StateProvider` 提供使用底层 P2P 堆栈而不是 RPC 的能力。

## 详细设计

本节将首先关注作为独立协议的反向同步(这里我们称之为“回填”)机制，然后描述它如何集成到状态同步反应器中以及我们如何定义新的 p2p 轻客户端提供程序。

```go
// Backfill fetches, verifies, and stores necessary history
// to participate in consensus and validate evidence.
func (r *Reactor) backfill(state State) error {}
```

`State` 用于计算返回多远，即我们需要所有具有以下特性的灯块:
- 高度:`h >= state.LastBlockHeight - state.ConsensusParams.Evidence.MaxAgeNumBlocks`
- 时间:`t >= state.LastBlockTime - state.ConsensusParams.Evidence.MaxAgeDuration`

反向同步依赖于两个组件:“Dispatcher”和“BlockQueue”。 `Dispatcher` 是一种取自类似 [PR](https://github.com/tendermint/tendermint/pull/4508) 的模式。它连接到“LightBlockChannel”，并通过在对等点的链接列表中移动来允许并发光块请求。这种抽象具有很好的品质，它也可以用作基于 P2P 的轻客户端的光提供者数组。

“BlockQueue”是一种数据结构，允许多个工作人员获取轻量块，为主线程序列化它们，主线程将它们从队列的末尾挑选出来，验证散列并将它们持久化。

### 与状态同步的集成

反向同步是一个阻塞过程，它在同步状态之后和转换到快速同步或共识之前直接运行。

之前，状态同步服务没有连接到任何数据库，而是将状态传递回节点。对于反向同步，状态同步将被授予访问 `StateStore` 和 `BlockStore` 的权限，以便能够写入 `Header`、`Commit` 和 `ValidatorSet` 并读取它们以服务于其他状态同步对等体。

这也意味着向这些各自的商店添加新方法以保持它们

### P2P 轻客户端提供商

如前所述，“Dispatcher”能够处理对多个对等点的请求。因此，我们可以简单地剥离分配给每个对等方的 `blockProvider` 实例。通过给它链 ID，`blockProvider` 能够在将它返回给客户端之前对光块进行基本的验证。

需要注意的是，由于状态同步无法访问证据通道，因此它无法允许轻客户端报告证据，因此“ReportEvidence”是无效的。这对于反向同步来说不是什么问题，但需要为纯 p2p 轻客户端解决。

### 修剪

最后一个小注意事项是修剪。此 ADR 将引入更改，不允许应用程序修剪证据时代内的块。

## 未来的工作

这个 ADR 试图保持在扩展状态同步的范围内，但是所做的更改为几个需要跟进的领域打开了大门:
- 在轻客户端包中正确集成 p2p 消息传递。这将需要添加证据通道，以便轻客户端能够报告证据。我们可能还需要重新考虑提供者模型(即当前提供者仅在启动时添加)
- 合并和清理薄荷存储(状态、块和证据)。此 ADR 向状态和块存储添加了新方法，用于保存标头、提交和验证器集。这不太适合当前的结构(即只保存了 `BlockMeta`s 而不是 `Header`s)。为了原子性和批处理的机会，我们应该探索合并这一点。还有其他方面的变化，例如我们存储块部件的方式。有关更多上下文，请参阅 [此处](https://github.com/tendermint/tendermint/issues/5383) 和 [此处](https://github.com/tendermint/tendermint/issues/4630)。
- 探索机会反向同步。从技术上讲，如果没有观察到证据，我们不需要反向同步。我已尝试设计协议，以便在我们认为合适的情况下可以将其转移到证据包中。因此，只有在我们没有必要数据的地方看到证据时，我们才执行反向同步。问题在于，假设我们达成共识，并且会弹出一些证据，要求我们首先获取并验证最后 10,000 个区块。节点无法(按顺序)执行此操作并在该轮结束之前进行投票。此外，由于我们不惩罚无效证据，恶意节点可以轻松地向链发送垃圾邮件，只是为了让一堆“无状态”节点执行一堆无用的工作。
- 探索完全反向同步。目前我们只获取光块。将来获取和持久化整个块可能会有好处，尤其是如果我们将控制权交给应用程序来执行此操作。

## 结果

### 积极的

- 所有节点都应该有足够的历史来验证所有类型的证据
- 状态同步节点可以使用 p2p 层进行状态的轻客户端验证。这具有更好的用户体验并且可能更快，但我没有进行基准测试。

### 消极的

- 引入更多代码 = 更多维护

### 中性的

## 参考

- [反向同步 RFC](https://github.com/tendermint/spec/blob/master/rfc/005-reverse-sync.md)
- [原始问题](https://github.com/tendermint/tendermint/issues/5617)
