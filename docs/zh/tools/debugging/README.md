# 调试

## Tendermint 调试杀死

Tendermint 带有一个 `debug` 子命令，可以让你杀死一个实时
Tendermint 处理同时在压缩档案中收集有用信息.
信息包括使用的配置、共识状态、网络
状态、节点的状态、WAL，甚至进程的堆栈跟踪
退出前. 这些文件可用于在调试故障时检查
Tendermint 过程.

```bash
tendermint debug kill <pid> </path/to/out.zip> --home=</path/to/app.d>
```


将调试信息写入压缩存档. 存档将包含
下列的:

```sh
├── config.toml
├── consensus_state.json
├── net_info.json
├── stacktrace.out
├── status.json
└── wal
```

在幕后，`debug kill` 从`/status`、`/net_info` 和
`/dump_consensus_state` HTTP 端点，并用 `-6` 终止进程，
捕获 go-routine 转储.

## Tendermint 调试转储

此外，`debug dump` 子命令允许您将调试数据转储到
定期压缩档案. 这些档案包含 goroutine
除了共识状态、网络信息、节点之外，还有堆配置文件
状态，甚至 WAL.

```bash
tendermint debug dump </path/to/out> --home=</path/to/app.d>
```

除了它只轮询节点和
每隔频率秒将调试数据转储到
给定的目标目录. 每个档案将包含:

```sh
├── consensus_state.json
├── goroutine.out
├── heap.out
├── net_info.json
├── status.json
└── wal
```

注意:goroutine.out 和 heap.out 只有在配置文件地址为
提供并运行. 此命令正在阻塞，并将记录任何错误.

## Tendermint 检查

Tendermint 包含一个 `inspect` 命令，用于查询 Tendermint 的状态存储和块
通过 Tendermint RPC 存储.

当 Tendermint 共识引擎检测到不一致的状态时，它会崩溃
整个 Tendermint 过程.
在这种不一致的状态下，运行 Tendermint 共识引擎的节点将不会启动.
`inspect` 命令仅运行 Tendermint RPC 端点的一个子集来查询块存储
和状态商店.
`inspect` 允许操作员查询舞台的只读视图.
`inspect` 根本不运行共识引擎，因此可用于调试
由于状态不一致而崩溃的进程.


要启动“检查”过程，请运行
```bash
tendermint inspect
```

### RPC 端点
可以通过向 RPC 端口发出请求来找到可用 RPC 端点的列表.
对于在 `127.0.0.1:26657` 上运行的 `inspect` 进程，将浏览器导航到
`http://127.0.0.1:26657/` 来检索已启用的 RPC 端点列表.

有关 Tendermint RPC 端点的其他信息可以在 [rpc 文档](https://docs.tendermint.com/master/rpc) 中找到.
