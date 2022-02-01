# 像专业人士一样调试

## 介绍

Tendermint Core 是一个相当强大的 BFT 复制引擎. 不幸的是，与其他软件一样，有时确实会发生故障. 那么问题是当系统偏离预期行为时“你会怎么做”.

第一反应通常是查看日志. 默认情况下，Tendermint 将日志写入标准输出 ¹.

```sh
I[2020-05-29|03:03:16.145] Committed state                              module=state height=2282 txs=0 appHash=0A27BC6B0477A8A50431704D2FB90DB99CBFCB67A2924B5FBF6D4E78538B67C1I[2020-05-29|03:03:21.690] Executed block                               module=state height=2283 validTxs=0 invalidTxs=0I[2020-05-29|03:03:21.698] Committed state                              module=state height=2283 txs=0 appHash=EB4E409D3AF4095A0757C806BF160B3DE4047AC0416F584BFF78FC0D44C44BF3I[2020-05-29|03:03:27.994] Executed block                               module=state height=2284 validTxs=0 invalidTxs=0I[2020-05-29|03:03:28.003] Committed state                              module=state height=2284 txs=0 appHash=3FC9237718243A2CAEE3A8B03AE05E1FC3CA28AEFE8DF0D3D3DCE00D87462866E[2020-05-29|03:03:32.975] enterPrevote: ProposalBlock is invalid       module=consensus height=2285 round=0 err="wrong signature (#35): C683341000384EA00A345F9DB9608292F65EE83B51752C0A375A9FCFC2BD895E0792A0727925845DC13BA0E208C38B7B12B2218B2FE29B6D9135C53D7F253D05"
```

如果您在生产中运行验证器，最好使用 filebeat 或类似工具转发日志以进行分析. 此外，您可以设置出现任何错误时的通知.

日志应该让您对所发生的事情有一个基本的了解. 在最坏的情况下，节点已经停止并且不会产生任何日志(或者只是恐慌).

下一步是调用 /status、/net_info、/consensus_state 和 /dump_consensus_state RPC 端点.

```sh
curl http://<server>:26657/status$ curl http://<server>:26657/net_info$ curl http://<server>:26657/consensus_state$ curl http://<server>:26657/dump_consensus_state
```

请注意，如果节点已停止，/consensus_state 和 /dump_consensus_state 可能不会返回结果(因为它们试图获取共识互斥锁).

这些端点的输出包含开发人员了解节点状态所需的所有信息. 它会让你知道节点是否落后于网络，它连接了多少对等点，以及最新的共识状态是什么.

此时，如果节点被停止并且您想重新启动它，您能做的最好的事情就是用 -6 信号杀死它:

```sh
kill -6 <PID>
```

这将转储当前正在运行的 goroutine 的列表. 该列表在调试死锁时非常有用.

`PID` 是 Tendermint 的进程 ID. 你可以通过运行`ps -a | 找到它. grep 薄荷| awk '{print $1}'`

## Tendermint 调试杀死

为了减轻收集不同数据片段的负担，Tendermint Core(自 v0.33 版本起)提供了 Tendermint 调试杀戮工具，它将为您完成上述所有步骤，将所有内容打包成一个不错的存档文件.

```sh
tendermint debug kill <pid> </path/to/out.zip> — home=</path/to/app.d>
```

这是官方文档页面——<https://docs.tendermint.com/master/tools/debugging>

如果你使用进程管理器，比如 systemd，它会自动重启 Tendermint. 我们强烈建议您在生产中安装一个. 如果没有，您将需要手动重新启动节点.

使用 Tendermint 调试的另一个优势是，在您认为存在软件问题的情况下，可以将相同的存档文件提供给 Tendermint 核心开发人员.

## Tendermint 调试转储

好的，但是如果节点没有停止，但它的状态随着时间的推移而下降怎么办？ Tendermint 调试转储来救援！

```sh
tendermint debug dump </path/to/out> — home=</path/to/app.d>
```

它不会杀死节点，但会收集上述所有数据并将其打包到存档文件中. 此外，它还会进行堆转储，这在 Tendermint 内存泄漏时应该会有所帮助.

此时，根据降级的严重程度，您可能需要重新启动该过程.

## Tendermint 检查

如果 Tendermint 节点由于不一致的共识状态而无法启动怎么办？

当运行 Tendermint 共识引擎的节点检测到不一致的状态时
它会使整个 Tendermint 进程崩溃.
Tendermint 共识引擎无法在这种不一致的状态下运行，所以节点
结果将无法启动.
在这种情况下，Tendermint RPC 服务器可以为调试提供有价值的信息.
Tendermint `inspect` 命令将运行 Tendermint RPC 服务器的一个子集
这对于调试不一致的状态很有用.

### 运行检查

使用以下命令在 Tendermint 崩溃的机器上启动 `inspect` 工具:
```bash
tendermint inspect --home=</path/to/app.d>
```

`inspect` 将使用 Tendermint 配置文件中指定的数据目录.
`inspect` 还将在 Tendermint 配置文件中指定的地址运行 RPC 服务器.

### 使用检查

随着 `inspect` 服务器的运行，您可以访问非常重要的 RPC 端点
用于调试.
调用 `/status`、`/consensus_state` 和 `/dump_consensus_state` RPC 端点
将返回有关 Tendermint 共识状态的有用信息.

##结尾

我们希望这些 Tendermint 工具将成为对任何事故的第一反应.

让我们知道您到目前为止的体验！ 您是否有机会尝试“tendermint debug”或“tendermint inspect”？

加入我们的 [discord chat](https://discord.gg/cosmosnetwork)，在那里我们讨论当前的问题和未来的改进.

—

[1]:当然，您可以自由地将 Tendermint 的输出重定向到文件或转发到另一台服务器.
