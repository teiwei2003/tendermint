# ADR 71:基于提议者的时间戳

## 变更日志

 - 2021 年 7 月 15 日:由@williambanfield 创建
 - 2021 年 8 月 4 日:@williambanfield 完成草案
 - 2021 年 8 月 5 日:更新草案以包括@williambanfield 的数据结构更改
 - 2021 年 8 月 20 日:@williambanfield 完成语言编辑
 - 2021 年 10 月 25 日:更新 ADR 以匹配@williambanfield 来自@cason 的更新规范
 - 2021 年 11 月 10 日:根据@cason 的反馈，@williambanfield 进行了其他语言更新

## 状态

 **公认**

## 语境

Tendermint 目前提供一种单调递增的时间源，称为 [BFTTime](https://github.com/tendermint/spec/blob/master/spec/consensus/bft-time.md)。
这种产生时间源的机制相当简单。
每个正确的验证器都会为它发送的每个“预提交”消息添加一个时间戳。
它发送的时间戳要么是验证者当前已知的 Unix 时间，要么是比前一个区块时间大一毫秒，具体取决于哪个值更大。
当一个区块产生时，提议者选择区块时间戳作为提议者收到的所有“预提交”消息中时间的加权中值。
权重与验证者在网络上的投票权或权益成正比。
这种产生时间戳的机制既是确定性的，又是拜占庭容错的。

这种用于生成时间戳的当前机制有一些缺点。
验证者完全不必就选定的区块时间戳与他们自己当前已知的 Unix 时间有多接近达成一致。
此外，任何数量的投票权“>1/3”都可以直接控制区块时间戳。
因此，时间戳很可能不是特别有意义。

这些缺点在 Tendermint 协议中存在问题。
轻客户端使用时间戳来验证块。
轻客户端依靠他们自己当前已知的 Unix 时间和区块时间戳之间的对应关系来验证他们看到的区块；
然而，由于“BFTTime”的限制，他们目前已知的 Unix 时间可能与区块时间戳有很大差异。

基于提议者的时间戳规范提出了一种生成块时间戳的替代方法，以解决这些问题。
基于提议者的时间戳以两种主要方式改变了当前生成区块时间戳的机制:

1. 区块提议者被修改为提供其当前已知的 Unix 时间作为下一个区块的时间戳，而不是“BFTTime”。
1. 只有当提议的区块时间戳与他们自己当前已知的 Unix 时间足够接近时，正确的验证者才会批准提议的区块时间戳。

这些更改的结果是一个更有意义的时间戳，无法由验证者投票权的“<= 2/3”控制。
本文档概述了 Tendermint 中必要的代码更改，以实现相应的 [基于提议者的时间戳规范](https://github.com/tendermint/spec/tree/master/spec/consensus/proposer-based-timestamp)。

## 替代方法

### 完全删除时间戳

由于各种原因，计算机时钟必然会出现偏差。
在我们的协议中使用时间戳意味着要么接受时间戳不可靠，要么影响协议的活性保证。
这种设计需要影响协议的活跃度，以使时间戳更可靠。
另一种方法是从块协议中完全删除时间戳。
`BFTTime` 是确定性的，但可能任意不准确。
但是，拥有可靠的时间来源对于构建在区块链之上的应用程序和协议非常有用。

因此我们决定不删除时间戳。
应用程序通常希望某些交易发生在某一天、某个固定时期或在不同事件之后的一段时间后发生。
所有这些都需要对商定的时间进行一些有意义的表示。
以下协议和应用程序功能需要可靠的时间来源:
* Tendermint 轻客户端 [依赖其已知时间](https://github.com/tendermint/spec/blob/master/spec/light-client/verification/README.md#definitions-1) 和区块时间之间的对应关系用于块验证。
* Tendermint 证据的有效性取决于 [无论是在高度还是时间方面](https://github.com/tendermint/spec/blob/8029cf7a0fcc89a5004e173ec065aa48ad5ba3c8/spec/consensus/evidence.md#verification)。
* 在 Cosmos Hub 中解除抵押资产 [在 21 天后发生](https://github.com/cosmos/governance/blob/ce75de4019b0129f6efcbb0e752cd2cc9e6136d3/params-change/Staking.md#unbondingtime)。
* IBC 数据包可以使用 [时间戳或高度来超时数据包传送](https://docs.cosmos.network/v0.43/ibc/overview.html#acknowledgements)。

最后，Cosmos Hub 中的通货膨胀分布使用时间的近似值来计算年百分比率。
这个时间的近似值是使用[区块高度和一年中产生的区块估计数量](https://github.com/cosmos/governance/blob/master/params-change/Mint.md#blocksperyear)计算的。
基于提议者的时间戳将允许这种通货膨胀计算使用更有意义和准确的时间来源。


## 决定

实施基于提议者的时间戳并删除“BFTTime”。

## 详细设计

### 概述

实现基于提议者的时间戳需要对 Tendermint 的代码进行一些更改。
这些更改将针对以下组件:
* `internal/consensus/` 包。
* `state/` 包。
* `Vote`、`CommitSig` 和 `Header` 类型。
* 共识参数。

### 更改为 `CommitSig`

[CommitSig](https://github.com/tendermint/tendermint/blob/a419f4df76fe4aed668a6c74696deabb9fe73211/types/block.go#L604) 结构目前包含一个时间戳。
这个时间戳是验证器在为块发布“预提交”时已知的当前 Unix 时间。
此时间戳不再使用，将在此更改中删除。

`CommitSig` 将更新如下:

```diff
type CommitSig struct {
	BlockIDFlag      BlockIDFlag `json:"block_id_flag"`
	ValidatorAddress Address     `json:"validator_address"`
--	Timestamp        time.Time   `json:"timestamp"`
	Signature        []byte      `json:"signature"`
}
```

### 对“投票”消息的更改

`Precommit` 和 `Prevote` 消息使用共同的 [投票结构](https://github.com/tendermint/tendermint/blob/a419f4df76fe4aed668a6c74696deabb9fe73211/types/vote.go#L50)。
此结构当前包含时间戳。
这个时间戳是使用 [voteTime](https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/internal/consensus/state.go#L2241) 函数设置的，因此投票时间对应于当前的 Unix 时间验证器。
对于 precommits，这个时间戳用于构造 [CommitSig 包含在 LastCommit 中的块中](https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/types/block.go#L754)
对于预投票，此字段当前未使用。
基于提议者的时间戳将使用提议者在区块中设置的时间戳，因此不再需要在投票消息中包含时间戳。
因此，此时间戳不再有用，将被删除。

`Vote` 将更新如下:

```diff
type Vote struct {
	Type             tmproto.SignedMsgType `json:"type"`
	Height           int64                 `json:"height"`
	Round            int32                 `json:"round"`
	BlockID          BlockID               `json:"block_id"` // zero if vote is nil.
--	Timestamp        time.Time             `json:"timestamp"`
	ValidatorAddress Address               `json:"validator_address"`
	ValidatorIndex   int32                 `json:"validator_index"`
	Signature        []byte                `json:"signature"`
}
```

### 新的共识参数

基于提议者的时间戳规范包括多个新参数，这些参数在所有验证器中必须相同。
这些参数是“PRECISION”、“MSGDELAY”和“ACCURACY”。

`PRECISION` 和 `MSGDELAY` 参数用于确定建议的时间戳是否可接受。
如果提案时间戳被认为是“及时的”，验证者只会对提案进行预投票。
如果提案时间戳在验证器已知的 Unix 时间的 `PRECISION` 和 `MSGDELAY` 内，则被认为是 `timely`。
更具体地说，如果`validatorLocalTime - PRECISION < proposalTime < validatorLocalTime + PRECISION + MSGDELAY`，则提案时间戳是`timely`。

因为`PRECISION`和`MSGDELAY`参数在所有验证器中必须相同，它们将被添加到[共识参数](https://github.com/tendermint/tendermint/blob/master/proto/tendermint/types /params.proto#L13) 作为 [持续时间](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration)。

基于提议者的时间戳规范还包括一个[新的准确度参数](https://github.com/tendermint/spec/blob/master/spec/consensus/proposer-based-timestamp/pbts-sysmodel_001_draft.md#pbts-clocksync -external0)。
直观地说，“准确度”代表了正确验证者的“真实”时间和当前已知时间之间的差异。
当前已知的任何验证器的 Unix 时间总是与实时有些不同。
`ACCURACY` 是每个验证者的时间和实际时间之间的最大差异，作为绝对值。
这不是计算机可以自行确定的，必须由运行基于 Tendermint 的链的社区指定为估计值。
它在新算法中用于[计算提议步骤的超时时间](https://github.com/tendermint/spec/blob/master/spec/consensus/proposer-based-timestamp/pbts-algorithm_001_draft.md# pbts-alg-startround0)。
假定所有验证器的“准确度”都相同，因此应作为共识参数包含在内。

共识将更新为包含此“时间戳”字段，如下所示:

```diff
type ConsensusParams struct {
	Block     BlockParams     `json:"block"`
	Evidence  EvidenceParams  `json:"evidence"`
	Validator ValidatorParams `json:"validator"`
	Version   VersionParams   `json:"version"`
++	Timestamp TimestampParams `json:"timestamp"`
}
```

```go
type TimestampParams struct {
	Accuracy  time.Duration `json:"accuracy"`
	Precision time.Duration `json:"precision"`
	MsgDelay  time.Duration `json:"msg_delay"`
}
```

### 修改区块提议步骤

#### Proposer 选择区块时间戳

Tendermint 目前使用“BFTTime”算法来生成块的“Header.Timestamp”。
[提议逻辑](https://github.com/tendermint/tendermint/blob/68ca65f5d79905abd55ea999536b1a3685f9f19d/internal/state/state.go#L269) 将`LastCommit.Commit's`sigs`中时间的加权中位数设置为提议的块`Header.Timestamp`。

在基于提议者的时间戳中，提议者仍然会在 `Header.Timestamp` 中设置一个时间戳。
提议者在 `Header` 中设置的时间戳将根据区块之前是否收到过 [polka](https://github.com/tendermint/tendermint/blob/053651160f496bb44b107a434e3e6482530bb287/docs/introduction/what-is-tendermint .md#consensus-overview)与否。

#### 先前未收到波尔卡的区块的提案

如果提议者正在提议一个新块，那么它会将提议者当前已知的 Unix 时间设置到 `Header.Timestamp` 字段中。
提议者还将将此相同的时间戳设置到它发出的“提议”消息的“时间戳”字段中。

#### 重新提议之前收到过 polka 的区块

如果提议者重新提议之前在网络上收到过 polka 的区块，则提议者不会更新该区块的 `Header.Timestamp`。
相反，提议者只是重新提议完全相同的区块。
这样，提议的区块与先前提议的区块具有完全相同的区块 ID，并且已经收到该区块的验证者不需要再次尝试接收它。

提议者将重新提议的区块的`Header.Timestamp` 设置为`Proposal` 消息的`Timestamp`。

#### 提议者等待

块时间戳必须单调递增。
在“BFTTime”中，如果验证者的时钟落后，[验证者将前一个区块的时间增加 1 毫秒，并在其投票消息中使用](https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0共识/state.go#L2246)。
添加基于提议者的时间戳的目标是强制执行某种程度的时钟同步，因此完全忽略验证者时间的 Unix 时间的机制不再有效。

验证器时钟不会完全同步。
因此，提议者当前已知的 Unix 时间可能小于前一个区块的 `Header.Time`。
如果提议者当前已知的 Unix 时间小于前一个块的 `Header.Time`，提议者将休眠，直到其已知的 Unix 时间超过它。

此更改将需要修改 [defaultDecideProposal](https://github.com/tendermint/tendermint/blob/822893615564cb20b002dd5cf3b42b8d364cb7d9/internal/consensus/state.go#L1180) 方法。
此方法现在应该安排一个超时，当提议者的时间大于前一个块的 `Header.Time` 时触发。
当超时触发时，提议者将最终发出“提议”消息。

#### 更改建议步骤超时

目前，如果达到配置的提议超时并且没有看到提议，则等待提议的验证器将通过提议步骤。
基于提议者的时间戳需要更改此超时逻辑。

提议者现在将等到其当前已知的 Unix 时间超过前一个区块的 `Header.Time` 来提议一个区块。
验证者现在必须在决定何时超时建议步骤时考虑这一点和其他一些因素。
具体来说，提议步骤超时还必须考虑验证者时钟和提议者时钟的潜在不准确性。
此外，从提议者向其他验证者传达提议消息可能会有延迟。

因此，等待提案的验证者必须等到上一个区块的“Header.Time”之后才会超时。
考虑到其自身时钟可能不准确、提议者时钟不准确和消息延迟，等待提议的验证器将等到前一个块的 `Header.Time + 2*ACCURACY + MSGDELAY`。
 规范将其定义为“waitingTime”。

[提议步骤的超时设置在 enterPropose](https://github.com/tendermint/tendermint/blob/822893615564cb20b002dd5cf3b42b8d364cb7d9/internal/consensus/state.go#L1108) 在 `state.go` 中。
`enterPropose` 将更改为使用新的共识参数计算等待时间。
`enterPropose` 中的超时时间将被设置为 `waitingTime` 和 [配置的提案步骤超时时间](https://github.com/tendermint/tendermint/blob/dc7c212c41a360bfe6eb38a6dd8c709bbc39aae7/config/config.go#L1013) 的最大值.

### 更改提案验证规则

验证提议块的规则将被修改以实现基于提议者的时间戳。
我们将更改验证逻辑以确保提案“及时”。

根据基于提议者的时间戳规范，仅当一个区块在一轮中没有收到 +2/3 多数票时才需要检查“及时”。
如果一个区块之前在上一轮中获得了 +2/3 的多数赞成票，那么 +2/3 的投票权就认为该区块的时间戳足够接近他们在该轮中当前已知的 Unix 时间。

验证逻辑将被更新，以检查之前在一轮中没有收到 +2/3 预投票的区块的“及时”。
在一轮中获得 +2/3 的预选票通常被称为“波尔卡”，为了简单起见，我们将使用这个术语。

#### 当前时间戳验证逻辑

为了更好地理解时间戳验证所需的更改，我们将首先详细说明时间戳验证当前在 Tendermint 中的工作原理。

[validBlock 函数](https://github.com/tendermint/tendermint/blob/c3ae6f5b58e07b29c62bfdc5715b6bf8ae5ee951/state/validation.go#L14) 目前[以三种方式验证提议的区块时间戳](https://github.com/ tendermint/tendermint/blob/c3ae6f5b58e07b29c62bfdc5715b6bf8ae5ee951/state/validation.go#L118)。
首先，验证逻辑检查此时间戳是否大于前一个块的时间戳。

其次，它验证块时间戳是否正确计算为 [块的 LastCommit] 中时间戳的加权中值(https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/types/block.go#L48)。

最后，验证逻辑验证“LastCommit.CommitSig”中的时间戳。
每个 `CommitSig` 中的加密签名是通过使用投票验证器的私钥对区块中的字段散列进行签名来创建的。
这个 `signedBytes` 哈希中的一项是 `CommitSig` 中的时间戳。
为了验证 `CommitSig` 时间戳，验证器验证投票会构建一个包含 `CommitSig` 时间戳的字段的哈希值，并根据签名检查此哈希值。
这发生在 [VerifyCommit 函数](https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/types/validation.go#L25) 中。

#### 删除未使用的时间戳验证逻辑

`BFTTime` 验证不再适用，将被删除。
这意味着验证器将不再检查区块时间戳是否是“LastCommit”时间戳的加权中值。
具体来说，我们将删除对[在validateBlock函数中的MedianTime](https://github.com/tendermint/tendermint/blob/4db71da68e82d5cb732b235eeb2fd69d62114b45/state/validation.go#L117)的调用。
`MedianTime` 函数可以完全删除。

由于 `CommitSig` 将不再包含时间戳，验证提交的验证器将不再在其构建的字段哈希中包含 `CommitSig` 时间戳以检查加密签名。

#### 区块未收到 polka 时的时间戳验证

`Proposal` 消息中的 [POLRound](https://github.com/tendermint/tendermint/blob/68ca65f5d79905abd55ea999536b1a3685f9f19d/types/proposal.go#L29) 表明区块在哪一轮收到了波尔卡。
`POLRound` 字段中的负值表示该块之前没有在网络上被提议。
因此，验证逻辑将在“POLRound < 0”时及时检查。

当验证者收到一个 `Proposal` 消息时，验证者将检查 `Proposal.Timestamp` 至多是大于验证器已知的当前 Unix 时间的`PRECISION`，并且至少小于当前的`PRECISION + MSGDELAY`验证器已知的 Unix 时间。
如果时间戳不在这些范围内，则提议的块将不被视为“及时”。

一旦收到与“Proposal”消息匹配的完整块，验证器还将检查块的“Header.Timestamp”中的时间戳是否与此“Proposal.Timestamp”匹配。
使用 `Proposal.Timestamp` 来检查 `timely` 可以更精细地调整 `MSGDELAY` 参数，因为 `Proposal` 消息不会改变大小，因此比网络上的完整块更快地被八卦。

验证器还将检查提议的时间戳是否大于先前高度的块的时间戳。
如果时间戳不大于上一个区块的时间戳，则该区块将不被视为有效，这与当前逻辑相同。

#### 区块收到 polka 时的时间戳验证

当一个区块被重新提议并且已经在网络上收到 +2/3 多数`Prevote`s 时，重新提议的区块的 `Proposal` 消息被创建为 `POLRound`，即 `>=0 `.
如果提议消息具有非负的“POLRound”，则验证器将不会检查“提议”是否为“及时”。
如果`POLRound` 为非负值，则每个验证器将简单地确保它在`POLRound` 指示的轮次中接收到提议块的`Prevote` 消息。

如果验证器没有收到“POLRound”中提议块的“Prevote”消息，那么它会prevote nil。
验证器已经检查过在`POLRound` 中看到了 +2/3 的预投票，所以这并不代表预投票逻辑的改变。

验证器还将检查提议的时间戳是否大于先前高度的块的时间戳。
如果时间戳不大于上一个区块的时间戳，则该区块将不被视为有效，这与当前逻辑相同。

此外，可以更新此验证逻辑以检查 `Proposal.Timestamp` 是否与提议区块的 `Header.Timestamp` 匹配，但它不太相关，因为检查是否收到投票足以确保区块时间戳是正确的。

### 更改为 prevote 步骤

目前，验证者将在以下三种情况之一中对提案进行投票:

* Case 1: Validator 没有被锁定的区块并且收到一个有效的提案。
* 情况 2:验证者有一个锁定的区块，并收到与其锁定的区块匹配的有效提案。
* 情况 3:验证者有一个锁定的区块，看到一个与其锁定的区块不匹配的有效提案，但在当前轮或大于或等于它锁定其的轮的一轮中看到对该提议区块的 +⅔ 赞成票锁定块。

我们将对 prevote 步骤进行的唯一更改是验证者认为有效提案的内容，如上所述。

### 对预提交步骤的更改

预提交步骤不需要太多修改。
除了“及时”检查之外，它的提案验证规则将以与验证在预投票步骤中更改的方式相同的方式更改:预提交验证永远不会检查时间戳是否为“及时”。

### 完全删除投票时间

[voteTime](https://github.com/tendermint/tendermint/blob/822893615564cb20b002dd5cf3b42b8d364cb7d9/internal/consensus/state.go#L2229) 是一种给定当前已知验证器时间和计算下一个“BFTTime”的机制前一个区块时间戳。
如果前一个区块时间戳大于验证者当前已知的 Unix 时间，则 voteTime 返回一个比前一个区块时间戳大一毫秒的值。
此逻辑在多个地方使用，不再需要用于基于提议者的时间戳。
因此，它应该被完全删除。

## 未来的改进

* 实现 BLS 签名聚合。
通过从 `Precommit` 消息中删除字段，我们能够聚合签名。

## 结果

### 积极的

* `<2/3` 的验证器不再影响区块时间戳。
* 区块时间戳与实时有更强的对应性。
* 提高轻客户端区块验证的可靠性。
* 启用 BLS 签名聚合。
* 使证据处理能够使用时间而不是高度来保证证据的有效性。

### 中性的

* 改变 Tendermint 的活性属性。
Liveness 现在要求所有正确的验证器在一个边界内具有同步的时钟。
Liveness 现在还需要验证器的时钟向前移动，这在“BFTTime”下是不需要的。

### 消极的

* 如果前一个提议者与当前提议者的本地 Unix 时间之间存在较大偏差，则可能会增加提议步骤的长度。
这个偏斜将受到`PRECISION`值的约束，所以它不太可能太大。

* 当前的区块时间戳在未来很远的链将需要暂停共识直到错误的区块时间戳之后，或者必须保持同步但非常不准确的时钟。

## 参考

* [PBTS 规范](https://github.com/tendermint/spec/tree/master/spec/consensus/proposer-based-timestamp)
* [BFTTime 规范](https://github.com/tendermint/spec/blob/master/spec/consensus/bft-time.md)
