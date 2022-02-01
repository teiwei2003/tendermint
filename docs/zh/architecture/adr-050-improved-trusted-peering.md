# ADR 50:改进的可信对等互连

## 变更日志
* 22-10-2019:初稿
* 05-11-2019: 将 `maximum-dial-period` 修改为 `persistent-peers-max-dial-period`

## 语境

当达到一个节点的 `max-num-inbound-peers` 或 `max-num-outbound-peers` 时，该节点不能为任何对等节点分配更多的时隙
通过入站或出站.因此，在断开连接一段时间后，任何重要的对等连接都可能无限期丢失
因为所有时隙都被其他对等方消耗掉了，节点不再尝试拨打对等方.

发生这种情况的原因有两个:指数退避和受信对等方缺乏无条件对等功能.


## 决定

我们建议通过在 `config.toml` 中引入两个参数来解决这个问题，`unconditional-peer-ids` 和
`持久对等最大拨号周期`.

1)`无条件对等IDs`

节点操作员输入允许入站或出站连接的对等节点的 id 列表，而不管
用户节点的“max-num-inbound-peers”或“max-num-outbound-peers”是否已到达.

2) `persistent-peers-max-dial-period`

在指数退避期间，每次拨号到每个持久对等点之间的期限不会超过“persistent-peers-max-dial-period”.
因此，`dial-period` = min(`persistent-peers-max-dial-period`, `exponential-backoff-dial-period`)

替代方法

Persistent-peer 仅用于出站，因此不足以涵盖“unconditional-peer-ids”的全部效用.
@creamers158(https://github.com/Creamers158) 建议将 id-only 项目放入持久对等体中作为
`unconditional-peer-ids`，但它需要非常复杂的结构异常来处理持久对等节点中不同结构的项目.
因此，我们决定使用“unconditional-peer-ids”来独立覆盖这个用例.

## 状态

建议的

## 结果

### 积极的

节点操作员可以在`config.toml`中配置两个新参数，以便他/她可以确保tendermint允许连接
来自/到`unconditional-peer-ids`中的对等点.此外，他/她可以确保每个持久对等体将至少在每个
`persistent-peers-max-dial-period` 术语.它为受信任的对等点实现了更稳定和持久的对等互连.

### 消极的

新特性在 `config.toml` 中引入了两个新参数，需要对节点操作符进行解释.

### 中性的

## 参考

* 两个p2p功能增强提案(https://github.com/tendermint/tendermint/issues/4053)
