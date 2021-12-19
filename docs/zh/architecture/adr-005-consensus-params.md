# ADR 005:共识参数

## 语境

迄今为止，控制区块链容量的共识关键参数已被硬编码、从本地配置加载或被忽略。
因为它们在不同的网络中可能需要不同，并且可能会随着时间的推移而演变
网络，我们试图在创世文件中初始化它们，并通过 ABCI 公开它们。

虽然我们现在有一些特定的参数，比如最大块和交易大小，但我们希望将来有更多，
例如证据有效的时间段或检查点的频率。

## 决定

### 共识参数

在`config.toml` 中不应该找到一致的关键参数。

一个新的 ConsensusParams 可选地包含在 `genesis.json` 文件中，
并加载到“状态”中。未包含的任何项目都设置为其默认值。
值 0 是未定义的(参见下面的 ABCI)。值 -1 用于指示该参数不适用。
这些参数用于通过所有相关参数的并集来确定块(和 tx)的有效性。

```
type ConsensusParams struct {
    BlockSize
    TxSize
    BlockGossip
}

type BlockSize struct {
    MaxBytes int
    MaxTxs int
    MaxGas int
}

type TxSize struct {
    MaxBytes int
    MaxGas int
}

type BlockGossip struct {
    BlockPartSizeBytes int
}
```

`ConsensusParams` 可以通过添加涵盖共识规则不同方面的新结构随着时间的推移而发展。

`BlockPartSizeBytes` 和 `BlockSize.MaxBytes` 强制大于 0。
前者因为我们需要一个部件大小，后者让我们总是至少对块的大小进行一些完整性检查。

### ABCI

#### 初始化链

InitChain 当前采用初始验证器集。它应该扩展到也包含 ConsensusParams 的一部分。
有一些情况可以让它占据整个创世记，除非可能在创世中有一些东西，
就像 BlockPartSize，应用程序不应该真正知道。

#### 结束块

EndBlock 响应包括一个 `ConsensusParams`，它包括 BlockSize 和 TxSize，但不包括 BlockGossip。
其他参数结构可以在未来添加到`ConsensusParams`。
`0` 值用于表示没有变化。
任何其他值都将更新“State.ConsensusParams”中的该参数，以应用于下一个块。
Tendermint 应该有硬编码的上限作为健全性检查。

## 状态

实施的

## 结果

### 积极的

- 无需重新编译软件即可指定替代容量限制和共识参数。
- 它们还可以在应用程序的控制下随时间变化

### 消极的

- 暴露的参数越多越复杂
- 区块链中不同高度的不同规则使快速同步变得复杂

### 中性的

- 检查有效性的 TxSize 可能与决定提案大小的配置的 `max_block_size_tx` 冲突
