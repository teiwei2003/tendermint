# ADR 037:交付块

作者:丹尼尔·拉辛(@danil-lashin)

## 变更日志

13-03-2019:初稿

## 语境

初始对话:https://github.com/tendermint/tendermint/issues/2901

一些应用程序可以并行处理事务，或者至少一些
部分 tx 处理可以并行化.现在开发人员不可能
并行执行 txs，因为 Tendermint 会相应地提供它们.

## 决定

现在 Tendermint 有 `BeginBlock`、`EndBlock`、`Commit`、`DeliverTx` 步骤
在执行块时.该文档建议将这些步骤合并为一个 `DeliverBlock`
步.它将允许应用程序的开发人员决定他们想要的方式
执行事务(并行或连续).它也将简化和
加速应用程序和 Tendermint 之间的通信.

正如@jaekwon [提到](https://github.com/tendermint/tendermint/issues/2901#issuecomment-477746128)
在讨论中，并非所有应用程序都将从该解决方案中受益.在某些情况下，
当应用程序相应地处理交易时，它会减慢区块链，
因为它需要等到完整的块传输到应用程序才能启动
处理它.另外，在ABCI完全改变的情况下，我们需要强制所有的应用程序
彻底改变他们的实施.这就是为什么我提议再引入一个 ABCI
类型.

# 实施变更

除了现在具有此结构的默认应用程序界面

```go
type Application interface {
    // Info and Mempool methods...

    // Consensus Connection
    InitChain(RequestInitChain) ResponseInitChain    // Initialize blockchain with validators and other info from TendermintCore
    BeginBlock(RequestBeginBlock) ResponseBeginBlock // Signals the beginning of a block
    DeliverTx(tx []byte) ResponseDeliverTx           // Deliver a tx for full processing
    EndBlock(RequestEndBlock) ResponseEndBlock       // Signals the end of a block, returns changes to the validator set
    Commit() ResponseCommit                          // Commit the state and return the application Merkle root hash
}
```

this doc proposes to add one more:

```go
type Application interface {
    // Info and Mempool methods...

    // Consensus Connection
    InitChain(RequestInitChain) ResponseInitChain           // Initialize blockchain with validators and other info from TendermintCore
    DeliverBlock(RequestDeliverBlock) ResponseDeliverBlock  // Deliver full block
    Commit() ResponseCommit                                 // Commit the state and return the application Merkle root hash
}

type RequestDeliverBlock struct {
    Hash                 []byte
    Header               Header
    Txs                  Txs
    LastCommitInfo       LastCommitInfo
    ByzantineValidators  []Evidence
}

type ResponseDeliverBlock struct {
    ValidatorUpdates      []ValidatorUpdate
    ConsensusParamUpdates *ConsensusParams
    Tags                  []kv.Pair
    TxResults             []ResponseDeliverTx
}

```

此外，我们需要添加新的配置参数，它将指定 ABCI 应用程序使用的类型.
例如，它可以是 `abci_type`. 然后我们将有两种类型:
- `高级` - 当前的 ABCI
- `简单` - 建议的实现

## 状态

审核中

## 结果

### 积极的

- 为新开发人员提供更简单的介绍和教程(而不是实施 5 种方法乳清
只需要实施 3)
- txs 可以并行处理
- 更简单的界面
- Tendermint 和应用程序之间更快的通信

### 消极的

- Tendermint 现在应该支持 2 种 ABCI
