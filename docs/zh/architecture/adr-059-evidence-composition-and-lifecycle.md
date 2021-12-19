# ADR 059:证据构成和生命周期

## 变更日志

- 04/09/2020:初稿(未删节)
- 07/09/2020:第一个版本
- 13/03/2021:修正以适应向前的疯狂攻击
- 29/06/2021:添加有关 ABCI 特定字段的信息

## 范围

本文档旨在整理和揭示一些涉及 Tendermint 证据的困境:其组成和生命周期。然后，它旨在找到解决这些问题的方法。范围不扩展到某些类型证据的验证或检测，而主要涉及证据的一般形式以及它如何从开始到应用。

## 背景

长期以来，在共识反应堆中形成的“DuplicateVoteEvidence”是 Tendermint 拥有的唯一证据。它是在同一轮中同一验证者的两次投票时产生的
被观察到，因此设计每个证据都是针对单个验证者的。据预测，可能会有更多形式的证据，因此“DuplicateVoteEvidence”被用作“证据”界面的模型以及发送到应用程序的证据数据的形式。需要注意的是，Tendermint 只关注证据的检测和报告，应用程序有责任进行惩罚。
```go
type Evidence interface { //existing
  Height() int64                                     // height of the offense
  Time() time.Time                                   // time of the offense
  Address() []byte                                   // address of the offending validator
  Bytes() []byte                                     // bytes which comprise the evidence
  Hash() []byte                                      // hash of the evidence
  Verify(chainID string, pubKey crypto.PubKey) error // verify the evidence
  Equal(Evidence) bool                               // check equality of evidence

  ValidateBasic() error
  String() string
}
```

```go
type DuplicateVoteEvidence struct {
  VoteA *Vote
  VoteB *Vote

  timestamp time.Time // taken from the block time
}
```

Tendermint 现在引入了一种新型证据来保护轻客户端免受攻击。这个`LightClientAttackEvidence`(见[这里](https://github.com/informalsystems/tendermint-rs/blob/31ca3e64ce90786c1734caf186e30595832297a4/docs/spec/lightclient/attacks/evidence)对于更多不同的信息处理是非常不同的。 `DuplicateVoteEvidence` 因为它在物理上有很大不同，包含完整的签名标头和验证器集。它是在轻客户端中形成的，而不是在共识反应器中形成，并且需要来自状态的更多信息来验证(`VerifyLightClientAttack(commonHeader,trustedHeader *SignedHeader, commonVals *ValidatorSet)` vs `VerifyDuplicateVote(chainID string, pubKey PubKey)`)。最后，它将验证器分批在一起(单个证据暗示同一高度的多个恶意验证器)，而不是拥有单独的证据(每条证据是每个高度的每个验证器)。该证据扩展了用于容纳新类型证据的现有模式，因此促使我们重新考虑应如何格式化和处理证据。

```go
type LightClientAttackEvidence struct { // proposed struct in spec
  ConflictingBlock *LightBlock
  CommonHeight int64
  Type  AttackType     // enum: {Lunatic|Equivocation|Amnesia}

  timestamp time.Time // taken from the block time at the common height
}
```
*注:这三种攻击类型已被研究团队证明是详尽无遗的*

## 证据组合的可能方法

### 个人框架

证据保留在每个验证者的基础上。这对当前流程造成的干扰最小，但要求我们将“LightClientAttackEvidence”分解为每个恶意验证器的几个证据。这不仅会对性能产生影响，因为数据库操作数量是 n 倍，而且证据八卦将需要更多的带宽(通过要求每个部分的标头)，而且它可能会影响我们验证它的能力。以批处理形式，全节点可以运行与轻客户端相同的过程，以查看公共块和冲突块中都存在 1/3 的验证能力，而在不打开恶意验证器的可能性的情况下，这变得更加难以单独验证伪造不利于无辜的证据。不仅如此，“LightClientAttackEvidence”还处理健忘症攻击，不幸的是，这种攻击的特点是我们知道所涉及的验证器集，但不知道实际上是恶意的子集(稍后将对此进行更多说明)。最后将证据分成单独的部分使得很难理解攻击的严重性(即攻击中涉及的总投票权)

#### 一个可能的实现路径的例子

我们将忽略健忘症证据(因为很难单独制作)并恢复到我们之前的初始拆分，其中“DuplicateVoteEvidence”也用于轻客户端模棱两可攻击，因此我们只需要“LunaticEvidence”。我们也很可能需要从界面中删除“Verify”，因为这并不是真正可以使用的东西。

``` go
type LunaticEvidence struct { // individual lunatic attack
  header *Header
  commonHeight int64
  vote *Vote

  timestamp time.Time // once again taken from the block time at the height of the common header
}
```

### 批处理框架

此类别的最后一种方法是仅考虑批处理证据。这适用于“LightClientAttackEvidence”，但需要更改“DuplicateVoteEvidence”，这很可能意味着共识会将相互冲突的投票发送到证据模块中的缓冲区，然后将所有投票按高度包装在一起，然后再将它们八卦到其他节点并试图在链上提交它。乍一看，这可能会提高 IO 和验证速度，也许更重要的是，对验证器进行分组可以让应用程序和 Tendermint 更好地了解攻击的严重程度。

然而，单个证据的优点是很容易检查节点是否已经拥有该证据，这意味着我们只需要检查哈希值即可知道我们之前已经验证过该证据。批处理证据意味着每个节点可能有不同的重复投票组合，这可能会使事情复杂化。

#### 一个可能的实现路径的例子

`LightClientAttackEvidence` 不会改变，但证据界面需要看起来像上面提议的那样，并且 `DuplicateVoteEvidence` 需要改变以包含多个双重投票。批处理证据的一个问题是它需要是唯一的，以避免人们提交不同的排列。

## 决定

决定是采用混合设计。

我们允许单个证据和批次证据共存，这意味着根据证据类型进行验证，并且大部分工作在证据池本身中完成(包括形成要发送给应用程序的证据)。


## 详细设计

证据有以下简单的界面:

```go
type Evidence interface {  //proposed
  Height() int64                                     // height of the offense
  Bytes() []byte                                     // bytes which comprise the evidence
  Hash() []byte                                      // hash of the evidence
  ValidateBasic() error
  String() string
}
```

接口的更改是向后兼容的，因为这些方法都存在于先前版本的接口中。 但是，随着验证的变化，网络将需要升级才能处理新的证据。

我们有两种具体类型的证据可以满足这个接口

```go
type LightClientAttackEvidence struct {
  ConflictingBlock *LightBlock
  CommonHeight int64 // the last height at which the primary provider and witness provider had the same header

  // abci specific information
	ByzantineValidators []*Validator // validators in the validator set that misbehaved in creating the conflicting block
	TotalVotingPower    int64        // total voting power of the validator set at the common height
	Timestamp           time.Time    // timestamp of the block at the common height
}
```
其中`Hash()` 是头部和commonHeight 的哈希值。

注意:还讨论了是否包含提交哈希来捕获对标头进行签名的验证器。 然而，这将为某人提供机会提出相同证据的多个排列(通过不同的提交签名)，因此它被省略了。 因此，当涉及到验证区块中的证据时，对于“LightClientAttackEvidence”，我们不能只检查哈希值，因为有人可能拥有与我们相同的哈希值，但提交不同的提交，其中不到 1/3 的验证者投票，这将是无效的 证据的版本。 (有关更多详细信息，请参阅“fastCheck”)

```go
type DuplicateVoteEvidence {
  VoteA *Vote
  VoteB *Vote

  // abci specific information
	TotalVotingPower int64
	ValidatorPower   int64
	Timestamp        time.Time
}
```
其中`Hash()`是两票的哈希值

对于这两种类型的证据，`Bytes()` 表示证据的原始编码字节数组格式，`ValidateBasic` 是
初始一致性检查，以确保证据具有有效的结构。

### 证据池

`LightClientAttackEvidence` 在轻客户端中生成，`DuplicateVoteEvidence` 在共识中生成。两者都通过“AddEvidence(ev Evidence) error”发送到证据池。证据池的主要目的是验证证据。它还可以将证据八卦到其他节点的证据池，并提供给共识，以便在链上提交并将相关信息发送到应用程序以进行惩罚。添加证据时，池首先运行“Has(ev Evidence)”以检查它是否已经收到(通过比较哈希值)，然后运行“Verify(ev Evidence) error”。验证后，证据池将其存储为待处理数据库。有两个数据库:一个用于尚未提交的待决证据，另一个用于提交的证据(避免提交两次证据)

#### 确认

`Verify()` 执行以下操作:

- 使用散列来查看我们提交的数据库中是否已经有了这个证据。

- 使用高度检查证据是否未过期。

- 如果它已过期，则使用高度查找区块头并检查时间是否也已过期，在这种情况下我们丢弃证据

- 然后对两个证据中的每一个进行 switch 语句:

对于`DuplicateVote`:

- 检查高度、圆形、类型和验证器地址是否相同

- 检查块 ID 是否不同

- 检查地址查找表以确保已经没有针对此验证器的证据

- 获取验证人集合并确认地址在攻击高度的集合中

- 检查链 ID 和签名是否有效。

对于`LightClientAttack`

- 从公共高度获取公共签名头和val集，并使用跳过验证来验证冲突的头

- 获取与冲突头部相同高度的可信签名头部，并与冲突头部进行比较，以确定它是哪种类型的攻击，并在这样做时返回恶意验证器。注意:如果节点在冲突头的高度处没有签名头，它会获取最新的头并检查它是否可以证明基于违反头时间的证据。这被称为前向疯狂攻击。

  - 如果模棱两可，则返回为受信任和已签名标头的提交签名的验证器

  - 如果疯了，从在冲突块中签名的公共验证集返回验证器

  - 如果失忆，则不返回验证器(因为我们无法知道哪些验证器是恶意的)。这也意味着我们目前不会向应用程序发送失忆证据，尽管我们将在未来的 Tendermint Core 版本中引入更强大的失忆证据处理

- 检查冲突标头和可信标头的哈希值是否不同

- 在前向疯子攻击的情况下，可信头高度小于冲突头高度，节点检查可信头的时间晚于冲突头的时间。这证明冲突的标头中断了单调增加的时间。如果该节点在以后没有受信任的标头，则它现在无法验证证据。

- 最后，对于每个验证器，检查查找表以确保已经没有针对该验证器的证据

验证后，我们将带有关键字“height/hash”的证据保存到证据池中的待处理证据数据库中。

#### ABCI 证据

两种证据结构都包含传递给应用程序所必需的数据(例如时间戳)，但严格来说并不构成不当行为的证据。因此，最后验证这些字段。如果这些字段中的任何一个对节点无效，即它们与它们的状态不对应，则节点将从现有字段重建新的证据结构，并用它们自己的状态数据重新填充 abci 特定字段。

#### 广播和接收证据

证据池还运行一个反应堆，广播新验证的
向所有连接的对等方提供证据。

从其他证据反应器接收证据的工作方式与从共识反应器或轻客户端接收证据的方式相同。


#### 在区块上提出证据

当涉及到预先投票和预先提交包含证据的提案时，全节点将再次
调用证据池使用`CheckEvidence(ev []Evidence)`来验证证据:

这将执行以下操作:

1. 遍历所有证据以检查没有任何重复

2. 对于每个证据，运行“fastCheck(ev evidence)”，它的工作原理类似于“Has”，但如果它具有“LightClientAttackEvidence”
相同的哈希然后继续检查它拥有的验证器是否都是冲突标头提交中的签名者。如果它没有通过快速检查(因为它之前没有看到证据)，那么它就必须验证证据。

3. 运行 `Verify(ev Evidence)` - 注意:这也将证据保存到数据库中，如前所述。


#### 更新应用程序和池

生命周期的最后一部分是提交块，然后“BlockExecutor”更新状态。作为此过程的一部分，“BlockExecutor”获取证据池以创建要发送到应用程序的证据的简化格式。这发生在“ApplyBlock”中，执行程序调用“Update(Block, State) []abci.Evidence”。

```go
abciResponses.BeginBlock.ByzantineValidators = evpool.Update(block, state)
```

以下是申请将收到的证据格式。 如上所示，这在“BeginBlock”中存储为数组。
除了使用枚举而不是字符串作为证据类型之外，对应用程序的更改很小(它仍然为每个恶意验证器形成一个)。

```go
type Evidence struct {
  // either LightClientAttackEvidence or DuplicateVoteEvidence as an enum (abci.EvidenceType)
	Type EvidenceType `protobuf:"varint,1,opt,name=type,proto3,enum=tendermint.abci.EvidenceType" json:"type,omitempty"`
	// The offending validator
	Validator Validator `protobuf:"bytes,2,opt,name=validator,proto3" json:"validator"`
	// The height when the offense occurred
	Height int64 `protobuf:"varint,3,opt,name=height,proto3" json:"height,omitempty"`
	// The corresponding time where the offense occurred
	Time time.Time `protobuf:"bytes,4,opt,name=time,proto3,stdtime" json:"time"`
	// Total voting power of the validator set in case the ABCI application does
	// not store historical validators.
	// https://github.com/tendermint/tendermint/issues/4581
	TotalVotingPower int64 `protobuf:"varint,5,opt,name=total_voting_power,json=totalVotingPower,proto3" json:"total_voting_power,omitempty"`
}
```


这个 `Update()` 函数执行以下操作:

- 跟踪用于测量到期的当前时间和高度的增量状态

- 将证据标记为已提交并保存到数据库。 这可以防止验证者在未来提出已提交的证据
   注意:db 只保存高度和哈希值。 无需保存全部提交的证据

- 形成这样的 ABCI 证据:(注意“DuplicateVoteEvidence”，验证器数组大小为 1)
  ```go
  for _, val := range evInfo.Validators {
    abciEv = append(abciEv, &abci.Evidence{
      Type: evType,   // either DuplicateVote or LightClientAttack
      Validator: val,   // the offending validator (which includes the address, pubkey and power)
      Height: evInfo.ev.Height(),    // the height when the offense happened
      Time: evInfo.time,      // the time when the offense happened
      TotalVotingPower: evInfo.totalVotingPower   // the total voting power of the validator set
    })
  }
  ```

- 从挂起和提交的数据库中删除过期的证据

然后通过“BlockExecutor”将 ABCI 证据发送到应用程序。

#### 概括

总而言之，我们可以看到证据的生命周期如下:

![evidence_lifecycle](../imgs/evidence_lifecycle.png)

首先在轻客户端和共识反应器中检测和创建证据。它被验证并存储为“EvidenceInfo”，并传到其他节点的证据池中。共识反应器稍后与证据池通信以检索要放入块中的证据，或验证共识反应器在块中检索到的证据。最后，当一个块被添加到链中时，块执行器将提交的证据发送回证据池，这样指向证据的指针就可以存储在证据池中，并且可以更新它的高度和时间。最后，它将提交的证据转换为 ABCI 证据，并通过块执行器将证据传递给应用程序，以便应用程序可以处理它。

## 状态

实施的

## 结果

<!--> 本节描述应用决定后的后果。所有的后果都应该在这里总结，而不仅仅是“积极的”后果。 -->

### 积极的

- 证据更好地包含在证据池/模块中
- LightClientAttack 保持在一起(更容易验证和带宽)
- LightClientAttack 中提交信号的变化不会导致多重排列和多重证据
- 证据映射地址可防止 DOS 攻击，在这种攻击中，单个验证者可以通过大量提交证据来对网络进行 DOS 攻击

### 消极的

- 更改了`Evidence`界面，因此是一个块破坏更改
- 更改了 ABCI `Evidence`，因此是 ABCI 的重大更改
- 没有证据池无法查询地址/时间的证据

### 中性的


## 参考

<!--> 是否有任何相关的 PR 评论、导致此问题的问题，或关于我们为何做出给定设计选择的参考文章？如果是这样，请在此处链接它们！ -->

- [LightClientAttackEvidence](https://github.com/informalsystems/tendermint-rs/blob/31ca3e64ce90786c1734caf186e30595832297a4/docs/spec/lightclient/attacks/evidence-handling.md)
