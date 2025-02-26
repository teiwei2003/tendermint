# ADR 45 - ABCI 证据处理

## 变更日志
* 21-09-2019:初稿

## 语境

证据是 Tendermint 区块中的一个独特组件，并且拥有自己的反应器
用于高优先级的八卦.目前，Tendermint 只支持单一形式的证据，一个明确的证据
模棱两可，验证者同时签署冲突块
高度/圆形.在共识反应器中被实时检测，并被八卦
通过证据反应堆.证据也可以通过RPC提交.

目前，Tendermint 不能优雅地处理主链上的分叉.
如果检测到分叉，则节点会发生恐慌.此时人工干预和
社会共识需要重新配置.我们想做更多的事情
这里很优雅，但那是另一天.

可以在没有分叉的情况下愚弄 lite 客户
主链 - 所谓的 Fork-Lite.见
[分叉问责制](https://docs.tendermint.com/master/spec/light-client/accountability/)
文档了解更多详情.对于顺序精简版客户端，这可以通过
模棱两可或健忘症攻击.对于跳过 lite 客户端，这也可能发生
通过疯狂的验证器攻击.应用程序必须有某种方式来惩罚
所有形式的不当行为.

关键问题是 Tendermint 是否应该管理证据
验证，或者是否应该将证据更像交易(即.
任意字节)并让应用程序处理它(包括所有签名
检查).

目前，证据验证由 Tendermint 处理.一旦承诺，
[证据已过
ABCI](https://github.com/tendermint/tendermint/blob/master/proto/tendermint/abci/types.proto#L354)
以简化形式在 BeginBlock 中，仅包括
证据的类型、它的高度和时间戳、它来自的验证器，以及
验证者的总投票权设置在高度.该应用程序信任 Tendermint
执行证据验证，因为 ABCI 证据不包含
应用程序的签名和其他数据以进行自我验证.

支持在 Tendermint 中处理证据的论据:

1) 对全节点的攻击必须能够被全节点实时检测到，即.在共识反应堆中.
  所以至少，任何涉及某事的证据都可以愚弄一个完整的
  节点必须由 Tendermint 本地处理，否则就没有办法
  用于 ABCI 应用程序检测它(即我们不会发送我们在此期间收到的所有选票
  对应用程序的共识......).

2) Amensia 攻击不容易被检测到——它们需要一个交互式的
  所有验证者之间的协议，以提交他们过去的理由
  票.我们关于 [如何做到这一点
  当前](https://github.com/tendermint/tendermint/blob/c67154232ca8be8f5c21dff65d154127adc4f7bb/docs/spec/consensus/fork-detection.md)
  是通过一个集中的
  被信任的活跃度的监控服务来聚合数据
  当前和过去的验证器，但会产生不当行为的证明(即.
  通过失忆症)，任何人都可以验证，包括区块链.
  验证者必须提交他们看到的所有相关共识的投票
  高度来证明他们的预先承诺是合理的.这是 Tendermint 特有的
  协议，如果协议升级可能会改变.所以会很尴尬
  从应用程序协调这一点.

3)证据八卦和tx八卦类似，但应该更高
  优先事项.由于内存池尚不支持任何优先级概念，
  证据是通过一个独特的证据反应器八卦的.如果我们只是治疗
  像任何其他交易一样的证据，完全留给应用程序，
  Tendermint 无法知道如何确定优先级，除非/直到我们
  显着升级内存池.因此我们需要继续处理证据
  明确并更新 ABCI 以支持通过以下方式发送证据
  CheckTx/DeliverTx，或引入新的 CheckEvidence/DeliverEvidence 方法.
  在任何一种情况下，我们都需要对 ABCI 进行更多更改，然后如果 Tendermint
  处理的事情，我们刚刚添加了对可以包含的另一种证据类型的支持
  在开始块中.

4) 所有 ABCI 应用程序框架都将受益于大部分繁重的工作
  由 Tendermint 处理，而不是每个人都需要重新实施
  每种语言的所有证据验证逻辑.

支持将证据处理移至应用程序的论据:

5) 跳过 lite 客户端要求我们跟踪所有验证器的集合
  在一段时间内绑定，以防验证器解除绑定但仍然
  slashable 标志无效标头以愚弄 lite 客户端. Cosmos-SDK
  staking/slashing 模块跟踪这一点，因为它用于 slashing.
  Tendermint 目前不跟踪这个，尽管它会跟踪
  验证器设置在每个高度.这倾向于管理证据
  应用程序避免冗余管理历史验证器集数据
  嫩肤

6) 需要处理支持跨链验证的申请
  来自其他连锁店的证据.这些数据将以交易的形式出现，
  但这意味着该应用程序将需要具有处理的所有功能
  证据，即使其自身链的证据由直接处理
  嫩肤.

7) 来自 lite 客户端的证据可能很大并构成某种形式的 DoS
  针对完整节点的向量.将其放入交易中允许它收取应用程序的费用
  在证据不实的情况下支付处决费用的机制.
  这意味着证据提交者必须能够负担
  提交，但如果证据有效，当然应该退还.
  也就是说，负担主要在全节点上，这不一定会受益
  从费用.


## 决定

以上似乎大多表明证据检测属于 Tendermint.
(5) 没有对 Tendermint 施加特别大的义务，并且 (6) 只是
意味着该应用程序可以使用 Tendermint 库.也就是说，(7) 是潜在的
引起一些担忧，尽管它仍然可以攻击与验证器无关的完整节点
(即，不受益于费用).这可以在带外处理，例如
全节点通过支付渠道或通过某些渠道提供轻客户端服务
其他支付服务.这也可以通过禁止客户端 IP 来缓解，如果它们
发送不良数据.请注意，客户实际上向我们发送了很多
数据放在首位.

一个单独的 ADR 将描述 Tendermint 将如何处理这些新形式的
证据，关于它将如何参与描述的监测协议
叉子
检测](https://github.com/tendermint/tendermint/blob/c67154232ca8be8f5c21dff65d154127adc4f7bb/docs/spec/consensus/fork-detection.md)文档，
以及它将如何跟踪过去的验证者并管理 DoS 问题.

## 状态

建议的.

## 结果

### 积极的

- ABCI 没有真正的变化
- Tendermint 处理所有应用程序的证据

### 中性的

- 需要注意 Tendermint RPC 上的拒绝服务

### 消极的

- Tendermint 通过跟踪在此期间作为验证者的所有公钥来复制数据
  解绑期
