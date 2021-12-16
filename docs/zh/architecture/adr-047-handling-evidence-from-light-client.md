# ADR 047:处理来自轻客户端的证据

## 变更日志
* 18-02-2020:初稿
* 24-02-2020:第二版
* 13-04-2020:添加 PotentialAmnesiaEvidence 和一些备注
* 31-07-2020:删除 PhantomValidatorEvidence
* 14-08-2020:引入光迹(现在列为替代方法)
* 20-08-2020:轻客户端在检测到时产生证据而不是传递给全节点
* 16-09-2020:实施后修订
* 15-03-2020:修正前向疯子攻击的情况

### 专业术语

- `LightBlock` 是轻客户端接收、验证和存储的数据单元。
它由一个高度相同的验证器集、提交和标头组成。
- **Trace** 被视为一系列高度范围内的光块
由于跳过验证而创建的。
- **Provider** 是一个完整的节点，轻客户端连接到并为轻客户端提供服务
客户端签名的标头和验证器集。
- `VerifySkipping`(有时称为二等分或验证非相邻)是一种方法
轻客户端用于从受信任的标头中验证目标标头。该过程包括验证
通过确保 1/3 的验证者签署了两者之间的中间标头
受信任的标头也签署了不受信任的标头。
- **Light Bifurcation Point**:如果轻客户端要使用两个提供程序运行“VerifySkipping”
(即主要和见证人)，分叉点是标题的高度
这些提供商中的每一个都不同但有效。这表明其中一个提供者
可能试图欺骗轻客户端。

## 语境

轻客户端使用的头部校验的二分法暴露
如果轻客户端信任期内的任何块有
一群恶意验证者，其权力超过轻客户端信任级别
(默认为 1/3)。为了提高轻客户端(和整个网络)的安全性，轻
客户端有一个检测器组件，用于比较由
主要针对见证标头。此 ADR 概述了减轻攻击的过程
在轻客户端上通过使用见证节点进行交叉引用。

## 替代方法

之前讨论的处理证据的方法是传递所有数据
轻客户端见证了当它观察到完整节点的不同标头时
过程。这被称为光迹，具有以下结构:

```go
type ConflictingHeadersTrace struct {
  Headers []*types.SignedHeader
}
```

这种方法的优点是不需要对光进行太多处理
客户端发生攻击时。虽然，这不是一个重要的
不同之处在于轻客户端在任何情况下都必须验证所有标头
从见证人和主要。使用trace会消耗大量带宽
并向全节点添加 DDOS 向量。


## 决定

轻客户端将分为两个组件:一个“验证器”(顺序或
跳过)和一个`检测器`(见[非正式的检测器](https://github.com/informalsystems/tendermint-rs/blob/master/docs/spec/lightclient/detection/detection.md))
.检测器将从主服务器中获取标题的踪迹，并对照所有
证人。对于具有发散头的见证人，检测器将首先验证头
通过平分由主要提供的迹线定义的所有高度。如果有效，
轻客户端将遍历两条痕迹并找到它的分叉点
可以继续提取任何证据(稍后将详细讨论)。

成功检测到证据后，轻客户端会将其发送给主和
停下来前作证。它不会向其他同行发送证据，也不会继续验证
主要的标头与任何其他标头。


## 详细设计

轻客户端的验证过程将从可信头开始，并使用二分法
算法来验证给定高度的标题。这将成为经过验证的标头(不
意味着它是可信的)。在两者之间验证的所有标头都被缓存并称为
中间标头和整个数组有时称为跟踪。

轻客户端的检测器然后获取所有标头并运行检测功能。

```golang
func (c *Client) detectDivergence(primaryTrace []*types.LightBlock, now time.Time) error
```

该函数采用它收到的最后一个标头，即目标标头并将其与所有见证人进行比较
它通过以下功能:

```golang
func (c *Client) compareNewHeaderWithWitness(errc chan error, h *types.SignedHeader,
	witness provider.Provider, witnessIndex int)
```

err 通道用于发回所有结果，以便它们可以并行处理。
无效的标头导致丢弃见证、缺少响应或没有标头被忽略
就像具有相同散列的标头一样。 然而，标题，
不同的散列然后触发主要和该特定见证人之间的检测过程。

这首先通过跳过并行运行的验证来验证见证人的标头
定位光分岔点

![](../imgs/light-client-detector.png)

这是通过以下方式完成的:
```golang
func (c *Client) examineConflictingHeaderAgainstTrace(
	trace []*types.LightBlock,
	targetBlock *types.LightBlock,
	source provider.Provider,
	now time.Time,
	) ([]*types.LightBlock, *types.LightBlock, error)
```

执行以下操作

1. 检查可信头是否相同。目前，它们在理论上应该没有区别
因为在客户端初始化后无法添加和删除见证人。但我们以任何方式这样做
作为健全性检查。如果这失败了，我们必须放弃见证人。

2. 在所有节点的相同高度使用二分法查询和验证见证人的头部
主要的中间标头(在上面的例子中是 A、B、C、D、F、H)。如果二分失败
或者见证人停止响应，那么我们可以称见证人有问题并放弃它。

3. 我们最终通过见证人获得了一个经过验证的头部，它与中间头部不同
(在上面的例子中，这是 E)。这是分叉点(这也可能是最后一个标题)。

4. 有一种独特的情况，正在检查的跟踪具有更大的块
高度高于目标块。这可以作为前向疯狂攻击的一部分发生，其中主要有
提供一个高度大于链头的灯块(见附录 B)。在这
在这种情况下，轻客户端将验证源块直到 targetBlock 并返回块中的
在高度上紧跟在 targetBlock 之后的跟踪作为 `ConflictingBlock`

这个函数然后返回来自共同头和公用头之间的见证节点的块的踪迹。
主要的发散头，因为它很可能，如右侧示例所示，多个
需要的标题以验证不同的标题。这条痕迹将
稍后使用(如本文档后面所述)。

![](../imgs/bifurcation-point.png)

现在，已经检测到攻击，轻客户端必须形成证据来证明它。有
主要或见证人可能会进行三种类型的攻击来试图欺骗轻客户端
验证错误的标题:Lunatic、Equivocation 和 Amnesia。因为结果是一样的
证明它所需的数据也非常相似，我们将这些攻击方式捆绑在一起
证据:

```golang
type LightClientAttackEvidence struct {
	ConflictingBlock *LightBlock
	CommonHeight     int64
}
```

轻客户端采取首先怀疑主客户端的立场。鉴于找到的分岔点
上面，它采用两个不同的标头并比较来自主要的标头是否有效
对证人的尊重。这是通过调用 `isInvalidHeader()` 来完成的，它会查看是否
任何一个确定性派生的报头字段彼此不同。这可能是其中之一
`ValidatorsHash`、`NextValidatorsHash`、`ConsensusHash`、`AppHash` 和 `LastResultsHash`。
在这种情况下，我们知道这是一次疯子攻击，为了帮助证人验证我们发送高度
上例中为 1 或上例中为 C 的公共头。如果这一切
哈希值相同，那么我们可以推断它是 Equivocation 或 Amnesia。在这种情况下，我们发送
发散头的高度，因为我们知道验证器集是相同的，因此
恶意节点仍然绑定在那个高度。在上面的例子中，这是高度 10 和
上面的例子是 E 处的高度。

轻客户端现在拥有证据并将其广播给证人。

但是，可能是轻客户端使用的标头来自见证人针对主要
是伪造的，所以在停止轻客户端之前交换进程并因此怀疑证人和
使用主要来创建证据。这次它调用了 `examineConflictingHeaderAgainstTrace` 使用
较早发现的目击者踪迹。
如果主要是恶意的，它很可能不会响应，但如果它是无辜的，那么
轻客户端将提供相同的证据，但这次是相互矛盾的
区块将来自见证节点而不是主节点。然后形成证据并发送给
主节点。

这然后结束该过程，并且在开始时调用的验证函数将错误返回到
用户。

有关如何进行这三种攻击中的每一种的详细概述，请参阅
[fork 责任规范](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client/accountability.md)。

## 全节点验证

当全节点从轻客户端收到证据时，它需要验证
在与同行闲聊并尝试将其提交到链上之前，先为自己考虑。概述了这个过程
 在 [ADR-059](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-059-evidence-composition-and-lifecycle.md)。

## 状态

实施的

## 结果

### 积极的

* 轻客户端提高了针对 Lunatic、Equivocation 和 Amnesia 攻击的安全性。
* 不需要中间数据结构来封装恶意行为
* 泛化证据使代码更简单

### 消极的

* 轻客户端上从 0.33.8 及更低版本开始的重大更改。以前的
版本仍然会发送`ConflictingHeadersEvidence`，但不会被识别
通过全节点。然而，轻客户端仍将拒绝标头并关闭。
* 失忆症攻击虽然被发现，但不会受到惩罚，因为它不是
从当前信息中清除哪些节点是恶意行为。
* 证据模块必须同时处理个人证据和分组证据。

### 中性的

## 参考

* [分叉问责规范](https://github.com/tendermint/spec/blob/master/spec/consensus/light-client/accountability.md)
* [ADR 056:轻客户端健忘症攻击](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-056-light-client-amnesia-attacks.md)
* [ADR-059:证据组合和生命周期](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-059-evidence-composition-and-lifecycle.md)
* [Informal 的轻客户端检测器](https://github.com/informalsystems/tendermint-rs/blob/master/docs/spec/lightclient/detection/detection.md)


## 附录 A

PhantomValidatorEvidence 用于捕获仍在质押的验证者
(即在保税期内)但不在当前验证者集中已投票支持一个区块。

在后来的讨论中，有人认为虽然可以保留幻像验证器
证据，任何情况下都是一个可能有能力参与的幻影验证器
愚弄轻客户端必须得到 1/3+ 疯狂验证者的帮助。

由疯狂攻击注入的新验证器也不太可能
将是目前仍然持有某些东西的验证者。

不仅如此，还需要大量额外的计算来存储所有
当前质押的验证者可能属于
一个虚拟验证器。鉴于此，它被删除了。

## 附录 B

狂攻的一种独特风味是向前狂攻。 这是恶意的地方
node 提供一个高度大于区块链高度的头部。 因此有
没有能够反驳恶意标题的证人。 这样的攻击也会
需要一个共犯，即至少一个其他见证人也返回相同的伪造标题。
尽管此类攻击可以在任意高度进行，但它们仍必须保持在
轻客户端实时时钟漂移。 因此，为了检测这种攻击，光
客户将等待一段时间

``
2 * MAX_CLOCK_DRIFT + 滞后
``

让见证人提供它拥有的最新区块。 鉴于时间限制，如果证人
在区块链的头部运行，它将有一个具有较早高度的头部，但是
稍后的时间戳。 这个可以用来证明primary已经提交了一个疯子的header
这违反了单调增加的时间。
