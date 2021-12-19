# ADR 023:ABCI `ProposeTx` 方法

## 变更日志

25-06-2018:初稿基于 [#1776](https://github.com/tendermint/tendermint/issues/1776)

## 语境

[#1776](https://github.com/tendermint/tendermint/issues/1776) 是
关于使用 Tendermint 实施 Plasma 子链的开放
核心作为共识/复制引擎。

由于 [Minimal Viable Plasma (MVP)](https://ethresear.ch/t/minimal-viable-plasma/426) 和 [Plasma Cash](https://ethresear.ch/t/plasma- cash-plasma-with-much-less-per-user-data-checking/1298)，ABCI 应用程序需要有一种机制来处理以下情况(在不久的将来可能会出现更多):

1. 根链上的`deposit`交易，必须由一个区块组成
    单笔交易，没有输入，只有一个输出
    有利于存款人。 在这种情况下，一个“块”由
    具有以下形状的交易:
   ```
   [0, 0, 0, 0, #input1 - zeroed out
    0, 0, 0, 0, #input2 - zeroed out
    <depositor_address>, <amount>, #output1 - in favour of depositor
    0, 0, #output2 - zeroed out
    <fee>,
   ]
   ```

“退出”交易也可以用类似的方式处理，其中
   输入是根链上退出的UTXO，输出属于
   保留的“刻录”地址，例如，‘0x0’。在这种情况下，有利于
   包含块只保存一个可能收到的交易
   特殊待遇。

2. 子链上的其他“内部”交易，可能发起
   单方面。最基本的例子是coinbase交易
   实施验证者节点激励，但也可能是特定于应用程序的。在
   在这些情况下，可能有利于此类交易
   以特定方式排序，例如，coinbase 交易将始终是
   在索引 0 处。一般而言，此类策略增加了确定性和
   区块链应用的可预测性。

虽然可以使用
现有的 ABCI，当前可用导致次优的解决方法。两个是
下面更详细地解释。

### 解决方案 1:基于应用状态的 Plasma 链

在这个工作中，应用程序维护了一个带有相应的“PlasmaStore”
`守门员`。 PlasmaStore 负责维护第二个独立的
符合MVP规范的区块链，包括`deposit`
块和其他“内部”交易。然后广播这些“虚拟”块
到根链。

然而，这种幼稚的方法从根本上是有缺陷的，因为它根据定义
与 Tendermint 维护的规范链不同。这是进一步
如果生成此类交易的业务逻辑是
潜在的不确定性，因为这甚至不应该在
`Begin/EndBlock`，因此可能会破坏共识保证。

此外，这对“观察者”——独立第三方，有着严重的影响，
甚至是辅助区块链，负责确保记录的区块
根链上的与 Plasma 链的一致。因为，在这种情况下，
Plasma 链与 Tendermint 维护的规范链不一致
核心，似乎不存在验证合法性的紧凑方法
Plasma 链，而无需重放从创世开始的每个状态转换(！)。

### 解决方案 2:从 ABCI 应用程序广播到 Tendermint Core

这种方法受到“tendermint”的启发，其中以太坊交易是
转发到 Tendermint 核心。它需要应用程序来维护客户端连接
到共识引擎。

每当需要创建“内部”交易时，该交易的提议者
当前块将交易或交易广播到 Tendermint 作为
需要以确保 Tendermint 链和 Plasma 链是
完全一致。

这允许“内部”交易通过完全共识
过程，并且可以在诸如“CheckTx”之类的方法中进行验证，即由
提议者，在语义上是否正确等。请注意，这涉及通知
区块提议者的 ABCI 应用程序，暂时被黑客入侵作为一种手段
进行这个实验，虽然这不应该是必要的
当前提议者被传递给“BeginBlock”。

将这些交易直接中继到根要容易得多
链智能合约和/或维护一个“压缩”的辅助链
100% 反映规范 (Tendermint) 的 Plasma 友好块
区块链。不幸的是，这种方法不是惯用的(即，利用
Tendermint 共识引擎以意想不到的方式)。此外，它不
允许应用程序开发人员:

- 控制提议区块中交易的 _ordering_(例如，索引 0，
  或 0 到 `n` 用于 coinbase 交易)
- 控制区块中交易的_数量_(例如，当“存款”
  块是必需的)

由于确定性在区块链工程中至关重要，因此这种方法，
虽然更可行，但也不应被视为适合生产。

## 决定

###`ProposeTx`

为了解决上述困难，ABCI 接口必须
公开一个额外的方法，暂定名为“ProposeTx”。

它应该具有以下签名:

```
ProposeTx(RequestProposeTx) ResponseProposeTx
```


其中 `RequestProposeTx` 和 `ResponseProposeTx` 是带有
以下形状:

```
message RequestProposeTx {
  int64 next_block_height = 1; // height of the block the proposed tx would be part of
  Validator proposer = 2; // the proposer details
}

message ResponseProposeTx {
  int64 num_tx = 1; // the number of tx to include in proposed block
  repeated bytes txs = 2; // ordered transaction data to include in block
  bool exclusive = 3; // whether the block should include other transactions (from `mempool`)
}
```

`ProposeTx` 将在 `mempool.Reap` 之前被调用
[行](https://github.com/tendermint/tendermint/blob/9cd9f3338bc80a12590631632c23c8dbe3ff5c34/consensus/state.go#L935)。
根据 `exclusive` 是 `true` 还是 `false`，建议的
然后将交易推送到从
`mempool.Reap`。

###`DeliverTx`

由于从 `ProposeTx` 接收到的 `tx` 列表 _not_ 通过 `CheckTx`，
提供一种区分“内部”交易的方法可能是个好主意
来自用户生成的，以防应用程序开发人员需要/想要采取额外措施
确保拟议交易的有效性。

因此，应该更改“RequestDeliverTx”消息以提供额外的标志，如下所示:

```
message RequestDeliverTx {
	bytes tx = 1;
	bool internal = 2;
}
```

或者，可以添加一个额外的方法“DeliverProposeTx”作为伴随
`ProposeTx`。但是，目前尚不清楚是否需要额外的开销
鉴于现在一个简单的标志可能就足够了，以保持共识保证。

## 状态

待办的

## 结果

### 积极的

- Tendermint ABCI 应用程序将能够作为最低限度可行的 Plasma 链运行。
- 从而可以向 `cosmos-sdk` 添加扩展以启用
  ABCI 应用程序支持 IBC 和 Plasma，最大限度地提高互操作性。
- ABCI 应用程序将在管理区块链状态方面具有极大的控制力和灵活性，
  不必求助于非确定性黑客和/或不安全的解决方法

### 消极的

- 暴露额外 ABCI 方法的维护开销
- 可能被忽视但现在必须进行广泛测试的潜在安全问题

### 中性的

- ABCI 开发人员必须处理增加的(尽管是名义上的)API 表面积。

## 参考

- [#1776 Plasma 和 ABCI 应用程序中的“内部”交易](https://github.com/tendermint/tendermint/issues/1776)
- [最小活血浆](https://ethresear.ch/t/minimal-viable-plasma/426)
- [Plasma Cash:Plasma 每用户数据检查少得多](https://ethresear.ch/t/plasma-cash-plasma-with-much-less-per-user-data-checking/1298)
