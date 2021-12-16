# ADR 058:事件散列

## 变更日志

- 2020-07-17:初始版本
- 2020-07-27:修复了 Ismail 和 Ethan 的评论
- 2020-07-27:拒绝

## 语境

在 [PR#4845](https://github.com/tendermint/tendermint/pull/4845) 之前，
`Header#LastResultsHash` 是从 `DeliverTx` 构建的 Merkle 树的根
结果。仅包含“代码”、“数据”字段，因为“信息”和“日志”
字段是不确定的。

在某些时候，我们向 `ResponseBeginBlock`、`ResponseEndBlock` 添加了事件，
和 `ResponseDeliverTx` 为应用程序提供一种附加一些额外的方法
块/交易的信息。

从那时起，许多应用程序似乎已经开始使用它们。

然而，在 [PR#4845](https://github.com/tendermint/tendermint/pull/4845) 之前
没有办法证明某些事件是结果的一部分
(_除非应用程序开发人员将它们包含在状态树中_)。

因此，[PR#4845](https://github.com/tendermint/tendermint/pull/4845) 是
打开。其中，散列时包含`GasWanted`和`GasUsed`
`DeliverTx` 结果。此外，来自`BeginBlock`、`EndBlock` 和`DeliverTx` 的事件
结果被散列到 `LastResultsHash` 中，如下所示:

- 由于我们不希望 `BeginBlock` 和 `EndBlock` 包含许多事件，
  这些将被 Protobuf 编码并作为叶子包含在 Merkle 树中。
- 因此，`LastResultsHash` 是具有 3 个叶子的 Merkle 树的根哈希:
  原型编码的 `ResponseBeginBlock#Events`，默克尔树构建的根哈希
  来自 `ResponseDeliverTx` 响应(日志、信息和代码空间字段是
  忽略)，以及原型编码的 `ResponseEndBlock#Events`。
- 事件顺序不变 - 与从 ABCI 应用程序收到的相同。

[Spec PR](https://github.com/tendermint/spec/pull/97/files)

虽然能够证明某些事情当然很好，但引入新事件
或者删除此类变得困难，因为它破坏了“LastResultsHash”。它
意味着每次添加、删除或更新事件时，您都需要一个
硬分叉。这无疑对不断发展的应用程序不利
没有一个稳定的事件集。

## 决定

作为一种折衷办法，该提议是增加
`Block#LastResultsEvents` 共识参数是所有事件的列表
将在标头中散列。
```
@ proto/tendermint/abci/types.proto:295 @ message BlockParams {
  int64 max_bytes = 1;
  // Note: must be greater or equal to -1
  int64 max_gas = 2;
  // List of events, which will be hashed into the LastResultsHash
  repeated string last_results_events = 3;
}
```

最初列表是空的。 ABCI 应用程序可以通过 `InitChain` 更改它
或`EndBlock`。

例子:

```go
func (app *MyApp) DeliverTx(req types.RequestDeliverTx) types.ResponseDeliverTx {
    //...
    events := []abci.Event{
        {
            Type: "transfer",
            Attributes: []abci.EventAttribute{
                {Key: []byte("sender"), Value: []byte("Bob"), Index: true},
            },
        },
    }
    return types.ResponseDeliverTx{Code: code.CodeTypeOK, Events: events}
}
```

对于要散列的“传输”事件，`LastResultsEvents` 必须包含一个
字符串“传输”。

## 状态

拒绝

**直到有更多的稳定性/动机/用例/需求，决定是
推动这个完全应用程序端，只有想要事件的应用程序
可以证明将它们插入到他们的应用程序端默克尔树中。当然
这给他们的应用程序状态带来了更大的压力，并使事件证明
特定于应用程序，但它可能有助于建立更好的用例意识
以及 Tendermint 最终应该如何做到这一点。**

## 结果

### 积极的

1. 网络可以执行参数更改建议以在添加新事件时更新此列表
2. 允许网络避免硬分叉
3. 事件仍然可以随意添加到应用程序中，而不会破坏任何东西

### 消极的

1. 又一个共识参数
2.在tendermint状态下跟踪更多的东西

## 参考

- [ADR 021](./adr-021-abci-events.md)
- [索引交易](../app-dev/indexing-transactions.md)

## 附录 A. 替代提案

另一个提议是在 `Event` 中添加 `Hash bool` 标志，类似于
`Index bool` EventAttribute 的字段。当为 true 时，Tendermint 会将其散列为
`LastResultsEvents`。缺点是逻辑是隐含的，取决于
主要取决于节点的运营商，他们决定运行什么应用程序代码。这
上述提议使其(逻辑)明确且易于升级
治理。
