# ADR 053:状态同步原型

状态同步现在[合并](https://github.com/tendermint/tendermint/pull/4705)。最新的 ABCI 文档[可用](https://github.com/tendermint/spec/pull/90)，请参考它而不是这个 ADR 了解详细信息。

此 ADR 概述了初始状态同步原型的计划，并且可能会随着我们获得反馈和经验而发生变化。它建立在 [ADR-042](./adr-042-state-sync.md) 中的讨论和发现之上，有关背景信息，请参阅该内容。

## 变更日志

* 2020-01-28:初稿(埃里克·格里纳克)

* 2020-02-18:初始原型后的更新(Erik Grinaker)
    * ABCI:添加了缺失的“原因”字段。
    * ABCI:使用 32 位基于 1 的块索引(64 位基于 0)。
    * ABCI:将`RequestApplySnapshotChunk.chain_hash` 移动到`RequestOfferSnapshot.app_hash`。
    * Gaia:快照还必须包括节点版本，包括内部节点和叶节点。
    * 添加了实验原型信息。
    * 添加了开放问题和实施计划。

* 2020-03-29:加强和简化的 ABCI 界面(Erik Grinaker)
    * ABCI:在 `Snapshot` 中用 `chunk_hashes` 替换了 `chunks`。
    * ABCI:删除了`SnapshotChunk` 消息。
    * ABCI:将`GetSnapshotChunk` 重命名为`LoadSnapshotChunk`。
    * ABCI:块现在简单地交换为“字节”。
    * ABCI:块现在是 0 索引，用于与 `chunk_hashes` 数组奇偶校验。
    * 将最大块大小减少到 16 MB，并将快照消息大小增加到 4 MB。

* 2020-04-29:更新最终发布的 ABCI 界面(Erik Grinaker)

## 语境

状态同步将允许新节点在不下载块或通过共识的情况下接收应用程序状态的快照。这使节点的引导速度明显快于当前的快速同步系统，后者重播所有历史块。

[ADR-042](./adr-042-state-sync.md) 中详细介绍了背景讨论和理由。其建议可概括为:

* 应用程序会定期拍摄完整状态快照(即热切快照)。

* 应用程序将快照拆分为更小的块，可以根据链应用程序哈希单独验证这些块。

* Tendermint 使用轻客户端获取可信链应用哈希进行验证。

* Tendermint 从多个对等点并行发现和下载快照块，并通过 ABCI 将它们传递给应用程序，以根据链应用程序哈希进行应用和验证。

* 历史区块不会被回填，因此状态同步的节点将有一个截断的区块历史。

## Tendermint 提案

这描述了从 Tendermint 看到的快照/恢复过程。界面尽可能小而通用，以便为应用程序提供最大的灵活性。

### 快照数据结构

一个节点可以在不同高度拍摄多个快照。快照可以采用不同的应用程序指定格式(例如 MessagePack 作为格式“1”和 Protobuf 作为格式“2”，或类似的模式版本控制)。每个快照由包含实际状态数据的多个块组成，用于并行下载和减少内存使用。

```proto
message Snapshot {
  uint64 height   = 1;  // The height at which the snapshot was taken
  uint32 format   = 2;  // The application-specific snapshot format
  uint32 chunks   = 3;  // Number of chunks in the snapshot
  bytes  hash     = 4;  // Arbitrary snapshot hash - should be equal only for identical snapshots
  bytes  metadata = 5;  // Arbitrary application metadata
}
```

块仅作为“字节”交换，并且不能大于 16 MB。 `Snapshot` 消息应该小于 4 MB。

### ABCI Interface

```proto
// Lists available snapshots
message RequestListSnapshots {}

message ResponseListSnapshots {
  repeated Snapshot snapshots = 1;
}

// Offers a snapshot to the application
message RequestOfferSnapshot {
  Snapshot snapshot = 1;  // snapshot offered by peers
  bytes    app_hash = 2;  // light client-verified app hash for snapshot height
 }

message ResponseOfferSnapshot {
  Result result = 1;

  enum Result {
    accept        = 0;  // Snapshot accepted, apply chunks
    abort         = 1;  // Abort all snapshot restoration
    reject        = 2;  // Reject this specific snapshot, and try a different one
    reject_format = 3;  // Reject all snapshots of this format, and try a different one
    reject_sender = 4;  // Reject all snapshots from the sender(s), and try a different one
  }
}

// Loads a snapshot chunk
message RequestLoadSnapshotChunk {
  uint64 height = 1;
  uint32 format = 2;
  uint32 chunk  = 3; // Zero-indexed
}

message ResponseLoadSnapshotChunk {
  bytes chunk = 1;
}

// Applies a snapshot chunk
message RequestApplySnapshotChunk {
  uint32 index  = 1;
  bytes  chunk  = 2;
  string sender = 3;
 }

message ResponseApplySnapshotChunk {
  Result          result         = 1;
  repeated uint32 refetch_chunks = 2;  // Chunks to refetch and reapply (regardless of result)
  repeated string reject_senders = 3;  // Chunk senders to reject and ban (regardless of result)

  enum Result {
    accept          = 0;  // Chunk successfully accepted
    abort           = 1;  // Abort all snapshot restoration
    retry           = 2;  // Retry chunk, combine with refetch and reject as appropriate
    retry_snapshot  = 3;  // Retry snapshot, combine with refetch and reject as appropriate
    reject_snapshot = 4;  // Reject this snapshot, try a different one but keep sender rejections
  }
}
```

### 拍摄快照

Tendermint 根本不知道快照过程，这完全是一个应用程序问题。必须提供以下保证:

* **定期:** 快照必须定期拍摄，而不是按需拍摄，以实现更快的恢复、更低的负载和更低的 DoS 风险。

* **确定性:** 快照必须是确定性的，并且在所有节点上都相同 - 通常通过在给定的高度间隔拍摄快照。

* **一致:** 快照必须一致，即不受并发写入的影响 - 通常使用支持版本控制和/或快照隔离的数据存储。

* **异步:** 快照必须是异步的，即不停止块处理和状态转换。

* **Chunked:** 快照必须拆分成合理大小的块(以兆字节为单位)，并且每个块都必须可以根据链应用程序哈希进行验证。

* **垃圾收集:**快照必须定期进行垃圾收集。

### 恢复快照

节点应该有启用状态同步和/或快速同步的选项，并为轻客户端提供可信的头哈希。

在启用状态同步和快速同步的情况下启动一个空节点时，快照恢复如下:

1. 节点检查它是否为空，即它没有状态也没有块。

2. 节点联系给定的种子以发现对等点。

3. 节点联系一组全节点，并通过轻客户端使用给定的哈希验证可信块头。

4. 节点通过“RequestListSnapshots”通过 P2P 从对等方请求可用快照。对等点将返回 10 个最近的快照，每个快照一条消息。

5.节点聚合来自多个peer的快照，按高度和格式排序(反向)。如果不同快照之间存在不匹配，则选择由最多对等点托管的快照。节点按高度和格式以相反的顺序遍历所有快照，直到找到满足以下所有条件的快照:

    * 快照高度的区块被轻客户端认为是可信的(即快照高度大于可信头并且在最新的可信块的解绑期内)。

    * 快照的高度或格式没有被早期的 `RequestOfferSnapshot` 明确拒绝。

    * 应用程序接受`RequestOfferSnapshot` 调用。

6. 节点通过`RequestLoadSnapshotChunk`从多个对等点并行下载块。块消息不能超过 16 MB。

7. 节点通过“RequestApplySnapshotChunk”将数据块依次传递给应用程序。

8. 一旦应用了所有块，节点将应用程序哈希与链应用程序哈希进行比较，如果它们不匹配，则错误或丢弃状态并重新开始。

9. 节点切换到快速同步以赶上恢复快照时提交的块。

10.节点切换到普通共识模式。

## 盖亚提案

这描述了从 Gaia 看到的快照过程，使用格式版本“1”。序列化格式未指定，但可能是压缩的 Amino 或 Protobuf。

### 快照元数据

在初始版本中没有快照元数据，因此将其设置为空字节缓冲区。

成功构建所有块后，快照元数据应存储在数据库中并通过“RequestListSnapshots”提供。

### 快照块格式

Gaia 数据结构由一组命名的 IAVL 树组成。根哈希是通过获取每个 IAVL 树的根哈希来构建的，然后构建一个经过排序的名称/哈希映射的 Merkle 树。

IAVL 树是版本化的，但快照仅包含与快照高度相关的版本。所有历史版本都被忽略。

IAVL 树依赖于插入顺序，因此必须以适当的插入顺序设置键/值对以生成相同的树分支结构。这个插入顺序可以通过对所有节点(包括内部节点)进行广度优先扫描并按顺序收集唯一键来找到。但是，节点哈希值还取决于节点的版本，因此快照也必须包含内部节点的版本号。

对于初始原型，每个块都包含整个 IAVL 树中所有节点的所有节点数据的完整转储。因此，块的数量等于 Gaia 中持久存储的数量。不进行块的增量验证，仅在快照恢复结束时进行最终的应用程序哈希比较。

对于生产版本，按插入顺序存储所有节点(叶节点和内部节点)的键/值/版本应该就足够了，以某种适当的方式分块。如果需要对每个块进行验证，则块还必须包含足够的信息来重建默克尔证明，一直到多存储的根，例如通过存储完整子树的键/值/版本数据以及所有其他分支的 Merkle 哈希值，直到多存储根。确切的方法将取决于大小、时间和验证之间的权衡。不推荐使用 IAVL RangeProofs，因为它们包括冗余数据，例如可以从上述数据中导出的中间节点和叶节点的证明。

应该通过收集不超过某个大小限制(例如 10 MB)的节点数据并将其序列化来贪婪地构建块。块数据作为`snapshots/<height>/<format>/<chunk>`存储在文件系统中，并且SHA-256校验和与快照元数据一起存储。

### 快照调度

快照应该以一些可配置的高度间隔拍摄，例如每 1000 个区块。所有节点最好都应该有相同的快照时间表，这样所有节点都可以为给定的快照提供块。

通过对 IAVL 树进行版本控制，可以极大地简化对 IAVL 树的一致快照:只需对与快照高度对应的版本进行快照，同时并发写入创建新版本。 IAVL 修剪不得修剪正在快照的版本。

快照也必须在一些可配置的时间后进行垃圾收集，例如通过保留最新的 `n` 个快照。

##已解决的问题

* 状态同步节点没有历史区块或历史 IAVL 版本是否可以？

    > 是的，正如预期的那样。也许稍后回填块。

* 第一个版本需要增量块验证吗？

    > 不，我们从简单的开始。可以通过新的快照格式添加块验证，而无需在 Tendermint 中进行任何重大更改。对于对抗性条件，可以考虑支持将节点列入白名单以从中下载块。

* 快照 ABCI 接口应该是一个单独的可选 ABCI 服务，还是强制性的？

    > 强制性的，暂时保持简单。因此，这将是一个突破性的变化并推动发布。对于使用 Cosmos SDK 的应用程序，我们可以提供一个默认实现，在尝试应用它们时不提供快照和错误。

* 我们如何确保`ListSnapshots` 数据有效？攻击者可以向 DoS 对等方提供虚假/无效的快照。

    > 目前，只需选择在大量对等点上可用的快照。也许支持白名单。我们可以考虑例如稍后将快照清单放在区块链上。

* 我们是否应该惩罚提供无效快照的节点？如何？

    > 不，这些是完整节点而不是验证器，所以我们不能惩罚它们。只需与它们断开连接并忽略它们即可。

* 我们应该称这些快照吗？ SDK 已经为`PruningOptions.SnapshotEvery` 使用了术语“快照”，状态同步将引入额外的 SDK 选项用于快照调度和修剪，这些选项与 IAVL 快照或修剪无关。

    > 是的。希望这些概念足够清晰，我们可以参考状态同步快照和 IAVL 快照而不会造成太多混淆。

* 我们应该在数据库中存储快照和块元数据吗？我们可以将数据库用于块吗？

    > 作为第一种方法，将元数据存储在数据库中，并将块存储在文件系统中。

* 高度 H 的快照应该在处理 H 处的块之前还是之后拍摄？例如。 RPC `/commit` 在 _previous_ 高度之后返回 app_hash，即 _before_ 当前高度。

    > 提交后。

* 我们是否需要支持所有版本的区块链反应器(即快速同步)？

    > 一旦 v2 稳定下来，我们应该完全移除 v1 反应器。

* `ListSnapshots` 应该是一个流 API 而不是请求/响应 API？

    > 不，只需使用最大消息大小。

## 状态

实施的

## 参考

* [ADR-042](./adr-042-state-sync.md) 及其参考
