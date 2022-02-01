# 状态同步


状态同步允许新节点通过发现、获取、
并恢复状态机快照.有关更多信息，请参阅 [状态同步 ABCI 部分](https://docs.tendermint.com/master/spec/abci/abci.html#state-sync)).

状态同步反应器有两个主要职责:

* 将本地 ABCI 应用程序拍摄的状态机快照提供给新加入的节点
  网络.

* 发现现有快照并为空的本地应用程序获取快照块
  被引导.

用于引导新节点的状态同步过程在链接的部分中详细描述
多于.虽然技术上是反应器的一部分(参见 `statesync/syncer.go` 和相关组件)，
本文档将仅涵盖 P2P 反应器组件.

有关 ABCI 方法和数据类型的详细信息，请参阅 [ABCI 文档](https://docs.tendermint.com/master/spec/abci/).

有关如何配置状态同步的信息位于 [节点部分](../../nodes/state-sync.md)

## 状态同步 P2P 协议

当一个新节点开始状态同步时，它会询问它遇到的所有对等点是否有
可用快照:

```go
type snapshotsRequestMessage struct{}
```

接收者将通过 ListSnapshots 查询本地 ABCI 应用程序，并发送消息
包含最近 10 个快照中每一个的快照元数据(限制为 4 MB):

```go
type snapshotsResponseMessage struct {
 Height   uint64
 Format   uint32
 Chunks   uint32
 Hash     []byte
 Metadata []byte
}
```

节点运行状态同步将通过以下方式将这些快照提供给本地 ABCI 应用程序
`OfferSnapshot` ABCI 调用，并跟踪哪些对等点包含哪些快照. 一次快照
被接受，状态同步器将从适当的对等方请求快照块:

```go
type chunkRequestMessage struct {
 Height uint64
 Format uint32
 Index  uint32
}
```

接收器将通过“LoadSnapshotChunk”从其本地应用程序加载请求的块，
并响应它(限制为 16 MB):

```go
type chunkResponseMessage struct {
 Height  uint64
 Format  uint32
 Index   uint32
 Chunk   []byte
 Missing bool
}
```

这里，“Missing”用于表示在对等方上找不到该块，因为一个空的
chunk 是一个有效的(虽然不太可能)响应.

返回的块通过“ApplySnapshotChunk”提供给 ABCI 应用程序，直到快照
被恢复. 如果在一段时间内没有返回块响应，它将被重新请求，
可能来自不同的同行.

作为 ABCI 协议的一部分，ABCI 应用程序能够请求对等禁止和块重新获取.

如果没有状态同步正在进行(即在正常操作期间)，任何未经请求的响应消息
被丢弃.
