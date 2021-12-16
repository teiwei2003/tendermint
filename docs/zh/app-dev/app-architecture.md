# 应用架构指南

在这里，我们提供了一个关于推荐架构的简要指南
Tendermint 区块链应用。

下图提供了一个极好的示例:

![cosmos-tendermint-stack](../imgs/cosmos-tendermint-stack-4k.jpg)

我们在这里区分两种形式的“应用程序”。第一个是
最终用户应用程序，例如用户下载的基于桌面的钱包应用程序，
这是用户实际与系统交互的地方。另一个是
ABCI 应用程序，这是实际运行在区块链上的逻辑。
最终用户应用程序发送的交易最终由 ABCI 处理
由 Tendermint 共识提交后的应用程序。

此图中的最终用户应用程序是位于底部的 [Lunie](https://lunie.io/) 应用程序
左边。 Lunie 与应用程序公开的 REST API 进行通信。
带有 Tendermint 节点并验证 Tendermint 轻客户端证明的应用程序
通过 Tendermint 核心 RPC。 Tendermint Core 进程与
本地 ABCI 应用程序，其中用户查询或交易实际上是
处理。

ABCI 应用程序必须是 Tendermint 的确定性结果
共识 - 对应用程序状态的任何外部影响没有
通过 Tendermint 可能会导致共识失败。因此_什么都没有_
应该通过 ABCI 与除 Tendermint 之外的 ABCI 应用程序进行通信。

如果 ABCI 应用程序是用 Go 编写的，它可以编译成
Tendermint 二进制文件。否则，它应该使用 unix 套接字进行通信
与 Tendermint。如果必须使用 TCP，则必须格外小心
加密和验证连接。

来自 ABCI 应用程序的所有读取都通过 Tendermint `/abci_query` 进行
端点。对 ABCI 应用程序的所有写入都通过 Tendermint 发生
`/broadcast_tx_*` 端点。

轻客户端守护进程为轻客户端(最终用户)提供
几乎全节点的所有安全性。它格式化和广播
交易，并验证查询和交易结果的证明。
请注意，它不必是守护进程——Light-Client 逻辑可以改为
在与最终用户应用程序相同的过程中实现。

注意对于那些安全要求较弱的 ABCI 应用程序，
轻客户端守护进程的功能可以移入 ABCI
申请流程本身。也就是说，公开 ABCI 申请流程
除了 ABCI 上的 Tendermint 之外的任何事情都需要格外小心，因为
所有事务，可能还有所有查询，仍应通过
嫩肤。

有关更广泛的文档，请参阅以下内容:

- [轻客户端 REST API 的链间标准](https://github.com/cosmos/cosmos-sdk/pull/1028)
- [Tendermint RPC 文档](https://docs.tendermint.com/master/rpc/)
- [生产中的 Tendermint](../tendermint-core/running-in-production.md)
- [ABCI 规范](https://github.com/tendermint/spec/tree/95cf253b6df623066ff7cd4074a94e7a3f147c7a/spec/abci)
