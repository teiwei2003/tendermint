# ADR 017:链式版本

## 去做

- 阐明如何在 ChainID 更改时处理斜线

## 变更日志

- 28-07-2018:审核更新
  - 分成两个 ADR - 一个用于协议，一个用于链
- 16-07-2018:初始草案 - 最初是协议和链的联合 ADR
  版本

## 语境

软件和协议版本包含在单独的 ADR 中。

在这里，我们专注于链版本。

## 要求

我们需要跨协议、网络、分叉等对区块链进行版本控制。
我们需要链标识符和描述，以便我们可以讨论众多链，
尤其是它们之间的差异，以一种有意义的方式。

### 网络

我们需要支持许多运行相同版本软件的独立网络，
甚至可能从相同的初始状态开始。
他们必须有不同的标识符，以便对等方知道他们正在加入哪个
验证者和用户可以防止重放攻击。

称其为“NetworkName”(注意我们目前在软件中称其为“ChainID”。在这个
ADR，ChainID 有不同的含义)。
它代表正在运行的应用程序和社区或意图
运行它。

对等点仅连接到具有相同 NetworkName 的其他对等点。

### 叉子

我们需要支持现有网络的升级和分叉，其中他们可以做以下任何一种:

    - 恢复到某个高度，继续使用相同的版本但新的块
    - 在某个高度任意改变状态，继续使用相同的版本(例如 Dao Fork)
    - 在某个高度更改 AppVersion

注意由于 Tendermint 的投票权阈值规则，一条链只能在“原始”规则和新规则下进行扩展
如果 1/3 或更多是双重签名，这是明确禁止的，并且应该导致他们在两条链上受到惩罚。因为他们可以审查
惩罚，预计链将被硬分叉以移除验证器。因此，如果两个分支都在分叉后继续，
他们每个人都需要一个新的标识符，而旧的链标识符将被淘汰(即只对同步历史有用，对新块没有用)。

TODO:解释当链 id 更改时如何处理斜线！

我们需要一种一致的方式来描述分叉。

## 提议

### 链描述

ChainDescription 是对区块链的完整不可变描述。它采用以下形式:

```
ChainDescription = <NetworkName>/<BlockVersion>/<AppVersion>/<StateHash>/<ValHash>/<ConsensusParamsHash>
```

这里，StateHash 是初始状态的默克尔根，ValHash 是初始 Tendermint 验证者集的默克尔根，
ConsensusParamsHash 是初始 Tendermint 共识参数的默克尔根。

`genesis.json` 文件必须包含足够的信息来计算这个值。它不需要包含 StateHash 或 ValHash 本身，
但包含可以使用给定协议版本计算它们的状态。

注意:考虑将 NetworkName 拆分为 NetworkName 和 AppName - 这允许
人们独立地为不同的网络使用相同的应用程序(即我们
可以想象多个验证者社区想要建立一个 Hub 使用
相同的应用程序，但具有不同的网络名称。可以说不需要，如果
差异将来自不同的初始状态/验证器)。

####链ID

定义`ChainID = TMHASH(ChainDescriptor)`。它是区块链的唯一 ID。

当由用户处理时，它应该是 Bech32 编码的，例如。带有 `cosmoschain` 前缀。

#### 分叉和升级

当一个链分叉或升级但继续相同的历史时，它需要一个新的 ChainDescription 如下:

``
ChainDescription = <ChainID>/x/<Height>/<ForkDescription>
``

在哪里

- ChainID 是来自之前 ChainDescription 的 ChainID(即它的哈希)
- `x` 表示发生了变化
- `Height` 是发生变化的高度
- ForkDescription 与 ChainDescription 具有相同的形式，但用于分叉
- 这允许分叉为tendermint或应用程序指定新版本，以及对状态或验证器集的任意更改
