# 内存池

内存池是潜在有效交易的内存池，
既向其他节点广播，也向其他节点提供
当它被选为区块提议者时，共识反应器.

内存池状态有两个方面:

- 外部:获取、检查和广播新交易
- 内部:返回有效交易，区块提交后更新列表

## 外部功能

外部功能通过网络接口公开
给可能不受信任的参与者.

- CheckTx - 通过 RPC 或 P2P 触发
- 广播 - 成功检查后的八卦消息

## 内部功能

内部功能通过方法调用公开给其他
代码编译成tendermint二进制文件.

- ReapMaxBytesMaxGas - 获取 txs 以在下一个区块中提议.保证
    txs 的大小小于 MaxBytes，gas 小于 MaxGas
- 更新 - 删除包含在最后一个块中的 tx
- ABCI.CheckTx - 调用 ABCI 应用程序来验证 tx

它为共识反应堆提供了什么？
它需要 ABCI 应用程序提供哪些保证？
(谈谈并发中的交错进程)

## 优化

该库中的实现还实现了 tx 缓存.
这样一来，如果 tx 具有，则不必重新验证签名
之前已经看过了.
但是，我们只在缓存中存储有效的 tx，而不是无效的.
这是因为无效的 txs 可能会在以后变好.
包含在块中的 Tx 不会从缓存中删除，
因为它们仍然可能通过 p2p 网络被接收.
这些交易通过它们的散列存储在缓存中，以减轻内存问题.

应用程序应实施重放保护，阅读 [重放
保护](https://github.com/tendermint/tendermint/blob/8cdaa7f515a9d366bbc9f0aff2a263a1a6392ead/docs/app-dev/app-development.md#replay-protection)了解更多信息.

## 配置

内存池有各种可配置的参数

发送错误编码的数据或超过 `maxMsgSize` 的数据将导致
在停止对等方.

`maxMsgSize` 等于 `MaxBatchBytes` (10MB) + 4(原型开销).
`MaxBatchBytes` 是一个内存池配置参数 -> 在本地定义.反应堆
将交易批量发送到连接的对等点.最大尺寸一
批处理是`MaxBatchBytes`.

内存池不会将 tx 发送回任何接收它的对等方.

反应器为每个对等体分配一个“uint16”编号，并维护一个来自
p2p.ID 到 `uint16`.每个内存池交易都携带所有发件人的列表
(`[]uint16`).每次内存池收到交易时都会更新列表
已经看到了. `uint16` 假设一个节点永远不会有超过 65535 个活动节点
peers(0 保留用于未知来源 - 例如 RPC).
