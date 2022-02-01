# ADR 025 提交

## 语境

目前，`Commit` 结构包含许多潜在的冗余或不必要的数据.
它包含来自每个验证器的预提交列表，其中预提交
包括整个“投票”结构. 因此，每个提交高度，圆形，
type 和 blockID 对每个验证器重复，并且可以进行重复数据删除，
导致非常显着的块大小节省.

```
type Commit struct {
    BlockID    BlockID `json:"block_id"`
    Precommits []*Vote `json:"precommits"`
}

type Vote struct {
    ValidatorAddress Address   `json:"validator_address"`
    ValidatorIndex   int       `json:"validator_index"`
    Height           int64     `json:"height"`
    Round            int       `json:"round"`
    Timestamp        time.Time `json:"timestamp"`
    Type             byte      `json:"type"`
    BlockID          BlockID   `json:"block_id"`
    Signature        []byte    `json:"signature"`
}
```

最初的跟踪问题是 [#1648](https://github.com/tendermint/tendermint/issues/1648).
我们已经讨论过用新的 `CommitSig` 替换 `Commit` 中的 `Vote` 类型
类型，其中至少包括投票签名. `Vote` 类型将
继续用于共识反应堆和其他地方.

一个主要的问题是什么应该包含在 `CommitSig` 中，超出
签名.当前的一个限制是我们必须包含一个时间戳，因为
这就是我们计算 BFT 时间的方式，尽管我们可以改变这一点 [在
未来](https://github.com/tendermint/tendermint/issues/2840).

这里的其他问题包括:

- 验证人地址 [#3596](https://github.com/tendermint/tendermint/issues/3596) -
    CommitSig 是否应该包含验证器地址？非常方便
    这样做，但可能没有必要.这也在 [#2226](https://github.com/tendermint/tendermint/issues/2226) 中讨论过.
- 缺席投票 [#3591](https://github.com/tendermint/tendermint/issues/3591) -
    如何代表缺席选票？目前，它们只是作为“nil”出现在
    Precommits 列表，实际上对于序列化是有问题的
- 其他 BlockID [#3485](https://github.com/tendermint/tendermint/issues/3485) -
    如何代表零和其他区块 ID 的投票？我们目前允许
    为 nil 投票并为替代块 ID 投票，但忽略它们


## 决定

删除重复字段并引入`CommitSig`:

```
type Commit struct {
    Height  int64
    Round   int
    BlockID    BlockID      `json:"block_id"`
    Precommits []CommitSig `json:"precommits"`
}

type CommitSig struct {
    BlockID  BlockIDFlag
    ValidatorAddress Address
    Timestamp time.Time
    Signature []byte
}


// indicate which BlockID the signature is for
type BlockIDFlag int

const (
	BlockIDFlagAbsent BlockIDFlag = iota // vote is not included in the Commit.Precommits
	BlockIDFlagCommit                    // voted for the Commit.BlockID
	BlockIDFlagNil                       // voted for nil
)

```

关于上下文中概述的问题:

**时间戳**:暂时保留时间戳.删除它并切换到
基于提议者的时间将需要更多的分析和工作，并将留待
未来的突破性变化.与此同时，对当前方法的担忧
BFT时间[可以是
减轻](https://github.com/tendermint/tendermint/issues/2840#issuecomment-529122431).

**ValidatorAddress**:我们现在将其包含在 `CommitSig` 中.虽然这
确实不必要地增加了块大小(每个验证器 20 字节)，它具有一些符合人体工程学和调试的优点:

- `Commit` 包含重建 `[]Vote` 所需的一切，并且不依赖于对 `ValidatorSet` 的额外访问
- Lite 客户端可以检查他们是否知道提交中的验证器，而无需
  重新下载验证器集
- 很容易直接在提交中看到哪些验证者没有签署什么
  获取验证器集

如果我们再次更改`CommitSig`，例如删除时间戳，
我们可以重新考虑是否应该删除 ValidatorAddress.

**缺席投票**:我们明确包括缺席投票，没有签名或
时间戳，但带有 ValidatorAddress.这应该解决序列化
问题，并可以轻松查看哪些验证者的投票未包含在内.

**其他区块ID**:我们使用单个字节来指示哪个区块ID是`CommitSig`
是为了.唯一的选择是:
    - `缺席` - 没有从这个验证者那里收到投票，所以没有签名
    - `Nil` - 验证者投票为零 - 意味着他们没有及时看到波尔卡
    - `Commit` - 验证者投票支持这个区块

请注意，这意味着我们不允许为任何其他区块 ID 投票.如果签名是
包含在提交中，它要么是 nil 要么是正确的 blockID.根据
Tendermint 协议和假设，正确的验证者无法
在实际提交的同一轮中为冲突块 ID 预提交
创建.这是大家的共识
[#3485](https://github.com/tendermint/tendermint/issues/3485)

我们以后可能会考虑支持其他的 blockID，作为捕获的一种方式
可能有帮助的证据.我们应该澄清是否/何时/如何这样做
实际上先帮助.为了实现它，我们可以改变`Commit.BlockID`
字段到切片，其中第一个条目是正确的块 ID，另一个是
条目是验证者之前预先提交的其他 BlockID. BlockIDFlag
枚举可以扩展以表示每个块上的这些附加块 ID
基础.

## 状态

实施的

## 结果

### 积极的

删除 Type/Height/Round/Index 和 BlockID 可以为每次预提交节省大约 80 个字节.
它会有所不同，因为某些整数是 varint. BlockID 包含两个 32 字节的哈希一个整数，
高度为 8 字节.

对于具有 100 个验证器的链，每个区块最多可节省 8kB！


### 消极的

- 块和提交结构的重大改变
- 需要区分 Vote 和 CommitSig 对象之间的代码，这可能会增加一些复杂性(需要重构投票以进行验证和八卦)

### 中性的

- Commit.Precommits 不再包含 nil 值
