# 在生产中运行

如果您正在从源代码构建 Tendermint 以用于生产，请确保签出适当的 Git 标签而不是分支.

## 数据库

默认情况下，Tendermint 使用 `syndtr/goleveldb` 包作为它的进程内
键值数据库.如果您想要最大的性能，最好安装
LevelDB 的真正 C 实现并编译 Tendermint 以使用它
`make build TENDERMINT_BUILD_OPTIONS=cleveldb`.请参阅 [安装
说明](../introduction/install.md) 了解详情.

Tendermint 在 `$TMROOT/data` 中保存了多个不同的数据库:

- `blockstore.db`:保留整个区块链 - 存储块，
  块提交和块元数据，每个都按高度索引.用于同步新
  同行.
- `evidence.db`:存储所有经过验证的不当行为证据.
- `state.db`:存储当前区块链状态(即高度、验证器、
  共识参数).只有在共识参数或验证器发生变化时才会增长.还
  用于在块处理期间临时存储中间结果.
- `tx_index.db`:通过 tx 哈希和 DeliverTx 结果事件索引 txs(及其结果).

默认情况下，Tendermint 只会根据它们的哈希和高度来索引 txs，而不是它们的 DeliverTx
结果事件.请参阅 [索引交易](../app-dev/indexing-transactions.md) 了解
细节.

应用程序可以向节点操作员公开块修剪策略.请阅读您的应用程序的文档
了解更多详情.

应用程序可以使用 [state sync](state-sync.md) 来帮助节点快速启动.

## 记录

默认日志级别(`log-level = "info"`)应该足以满足
正常操作模式.读这个
帖子](https://blog.cosmos.network/one-of-the-exciting-new-features-in-0-10-0-release-is-smart-log-level-flag-e2506b4ab756)
有关如何配置“日志级别”配置变量的详细信息.某些
模块可以在[这里](logging.md#list-of-modules)找到.如果
您正在尝试调试 Tendermint 或要求提供调试日志
日志记录级别，您可以通过运行 Tendermint 来实现
`--log-level="调试"`.

### 共识 WAL

Tendermint 使用预写日志 (WAL) 来达成共识. `consensus.wal` 用于确保我们可以在任何时候从崩溃中恢复
在共识状态机中.它写入所有共识消息(超时、提案、块部分或投票)
到单个文件，在处理来自它自己的消息之前刷新到磁盘
验证器.由于预计 Tendermint 验证者永远不会签署相互冲突的投票，因此
WAL 确保我们始终可以确定性地恢复到共识的最新状态，而无需
使用网络或重新签署任何共识消息.共识 WAL 最大大小为 1GB 并自动轮换.

如果您的 `consensus.wal` 已损坏，请参阅 [下文](#wal-corruption).

## DOS 暴露和缓解

验证器应该设置 [Sentry Node
架构](./validators.md)
以防止拒绝服务攻击.

###点对点

Tendermint 点对点系统的核心是“MConnection”.每个
连接有`MaxPacketMsgPayloadSize`，这是最大的数据包
大小和有界发送和接收队列.一个人可以施加限制
每个连接的发送和接收速率(`SendRate`、`RecvRate`).

打开的 P2P 连接的数量会变得非常大，并影响操作系统的打开
文件限制(因为 TCP 连接在基于 UNIX 的系统上被视为文件).节点应该是
给定一个相当大的打开文件限制，例如8192，通过 `ulimit -n 8192` 或其他特定于部署的
机制.

### RPC

返回多个条目的端点默认限制为返回 30
元素(最多 100 个).参见[RPC文档](https://docs.tendermint.com/master/rpc/)
想要查询更多的信息.

速率限制和身份验证是帮助保护的另一个关键方面
抵御 DOS 攻击.验证器应该使用外部工具，如
[NGINX](https://www.nginx.com/blog/rate-limiting-nginx/) 或
[traefik](https://docs.traefik.io/middlewares/ratelimit/)
达到同样的目的.

## 调试 Tendermint

如果你不得不调试 Tendermint，你应该做的第一件事就是
查看日志.参见 [Logging](../nodes/logging.md)，我们在那里
解释某些日志语句的含义.

如果在浏览日志后，事情仍然不清楚，接下来的事情
尝试查询`/status` RPC 端点.它提供了必要的信息:
无论何时节点同步与否，它的高度是多少等等.

```bash
curl http(s)://{ip}:{rpcPort}/status
```

`/dump_consensus_state` 会给你一个共识的详细概述
状态(提议者、最新验证者、对等状态). 从它，你应该能够
例如，找出网络停止的原因.

```bash
curl http(s)://{ip}:{rpcPort}/dump_consensus_state
```

这个端点有一个简化版本 - `/consensus_state`，它返回
只是在当前高度看到的选票.

如果在查阅了日志和以上端点后，您仍然不知道
发生了什么，请考虑使用“tendermint debug kill”子命令.这
命令将废弃所有可用信息并终止进程.看
[调试](../tools/debugging/README.md) 获取确切格式.

您可以自己检查生成的存档或在
[Github](https://github.com/tendermint/tendermint).在打开问题之前
但是，请务必检查是否有 [no existing
问题](https://github.com/tendermint/tendermint/issues)已经.

## 监控 Tendermint

每个 Tendermint 实例都有一个标准的 `/health` RPC 端点，它响应
200(OK)如果一切正常，500(或没有响应) - 如果有什么
错误的.

其他有用的端点包括前面提到的`/status`、`/net_info` 和
`/验证器`.

Tendermint 还可以报告和提供 Prometheus 指标.看
[指标](./metrics.md).

`tendermint debug dump` 子命令可用于定期转储有用的
信息归档.更多信息请参见 [调试](../tools/debugging/README.md)
信息.

## 当我的应用程序死亡时会发生什么

您应该在 [进程
主管](https://en.wikipedia.org/wiki/Process_supervision)(如
systemd 或 runit).它将确保 Tendermint 始终运行(尽管
可能的错误).

回到最初的问题，如果您的应用程序死了，
Tendermint 会恐慌.在流程主管重新启动您的
应用程序，Tendermint 应该能够成功重新连接.这
重启顺序无关紧要.

## 信号处理

我们捕获 SIGINT 和 SIGTERM 并尝试很好地清理.对于他人
我们在 Go 中使用默认行为的信号:[信号的默认行为
在围棋中
程序](https://golang.org/pkg/os/signal/#hdr-Default_behavior_of_signals_in_Go_programs).

##腐败

**注意:** 确保您有 Tendermint 数据目录的备份.

### 可能的原因

请记住，大多数损坏是由硬件问题引起的:

- RAID 控制器的备用电池出现故障/磨损，以及意外断电
- 启用了回写缓存​​的硬盘驱动器，并且意外断电
- 低价固态硬盘，断电保护不足，意外断电
- 有缺陷的内存
- CPU 有缺陷或过热

其他原因可能是:

- 配置了 fsync=off 和操作系统崩溃或断电的数据库系统
- 文件系统配置为使用写屏障加上一个忽略写屏障的存储层. LVM 是一个特别的罪魁祸首.
- Tendermint 错误
- 操作系统错误
- 管理员错误(例如，直接修改 Tendermint 数据目录内容)

(来源:<https://wiki.postgresql.org/wiki/Corruption>)

### WAL 腐败

如果共识 WAL 在最新高度损坏并且您正在尝试启动
Tendermint，重放会因恐慌而失败.

从数据损坏中恢复可能既困难又耗时.您可以采用以下两种方法:

1、删除WAL文件，重启Tendermint.它将尝试与其他对等点同步.
2.尝试手动修复WAL文件:

1) 创建损坏的 WAL 文件的备份:

    ```sh
    cp "$TMHOME/data/cs.wal/wal" > /tmp/corrupted_wal_backup
    ```

2) 使用 `./scripts/wal2json` 创建一个人类可读的版本:

    ```sh
    ./scripts/wal2json/wal2json "$TMHOME/data/cs.wal/wal" > /tmp/corrupted_wal
    ```

3) 搜索“CORRUPTED MESSAGE”行.
4)通过查看前一条消息和损坏后的消息
    并查看日志，尝试重建消息. 如果随之而来
    消息也被标记为已损坏(如果长度头
    已损坏或某些写入未进入 WAL ~ 截断)，
    然后删除从损坏的行开始的所有行并重新启动
    嫩肤.

    ```sh
    $EDITOR /tmp/corrupted_wal
    ```

5) 编辑完成后，运行以下命令将此文件转换回二进制形式:

    ```sh
    ./scripts/json2wal/json2wal /tmp/corrupted_wal  $TMHOME/data/cs.wal/wal
    ```

## 硬件

### 处理器和内存

虽然实际规格因负载和验证器数量而异，但最小
要求是:

- 1GB 内存
- 25GB 磁盘空间
- 1.4 GHz 中央处理器

SSD 磁盘更适合具有高事务吞吐量的应用程序.

受到推崇的:

- 2GB 内存
- 100GB 固态硬盘
- x64 2.0 GHz 2v CPU

就目前而言，Tendermint 存储所有历史记录，并且可能需要大量
随着时间的推移磁盘空间，我们计划实现状态同步(参见 [this
问题](https://github.com/tendermint/tendermint/issues/828)).所以，存储所有
过去的块将是不必要的.

### 验证器在 32 位架构(或 ARM)上签名

我们的 `ed25519` 和 `secp256k1` 实现都需要恒定的时间
`uint64` 乘法.非恒定时间加密可能(并且已经)泄露
`ed25519` 和 `secp256k1` 上的私钥.这在硬件中不存在
在 32 位 x86 平台上([来源](https://bearssl.org/ctmul.html))，它
取决于编译器强制执行它是恒定时间.不清楚在
每当 Golang 编译器为所有人正确执行此操作时
实现.

**我们不支持也不建议在 32 位架构上运行验证器或
“VIA Nano 2000 系列”，ARM 部分的架构被评为
“S-”.**

### 操作系统

由于 Go，Tendermint 可以为各种操作系统编译
语言(\$OS/\$ARCH 对的列表可以在
[这里](https://golang.org/doc/install/source#environment)).

虽然我们不偏爱任何操作系统，更安全稳定的Linux服务器
发行版(如 Centos)应优先于桌面操作系统
(如 Mac 操作系统).

不提供本机 Windows 支持.如果您使用的是 Windows 机器，您可以尝试使用 [bash shell](https://docs.microsoft.com/en-us/windows/wsl/install-win10).

### 各种各样的

注意:如果您打算在公共领域使用 Tendermint，请确保
您阅读了 [硬件建议](https://cosmos.network/validators) 中的验证器
宇宙网络.

## 配置参数

-`p2p.flush-throttle-timeout`
-`p2p.max-packet-msg-payload-size`
- `p2p.send-rate`
-`p2p.recv-rate`

如果您打算在私有域中使用 Tendermint 并且您有一个
同龄人之间的专用高速网络，降低
刷新油门超时并增加其他参数.

```toml
[p2p]
send-rate=20000000 # 2MB/s
recv-rate=20000000 # 2MB/s
flush-throttle-timeout=10
max-packet-msg-payload-size=10240 # 10KB
```

-`mempool.recheck`

在每个区块之后，Tendermint 都会重新检查留在区块中的每笔交易
内存池查看在该块中提交的事务是否影响了
应用程序状态，因此剩下的一些交易可能会变得无效.
如果这不适用于您的应用程序，您可以通过以下方式禁用它
设置`mempool.recheck=false`.

-`mempool.broadcast`

将此设置为 false 将阻止内存池中继交易
到其他对等点，直到它们被包含在一个块中.这意味着只有
您将 tx 发送到的对等方将看到它，直到它被包含在一个块中.

- `consensus.skip-timeout-commit`

当有经济学就行时，我们想要`skip-timeout-commit=false`
因为提议者应该等待更多的选票.但如果你不
关心这一点并想要最快的共识，你可以跳过它.它会
对于公共部署，默认情况下保持 false(例如 [Cosmos
Hub](https://hub.cosmos.network/main/hub-overview/overview.html)) 而对于企业
应用程序，将其设置为 true 不是问题.

- `consensus.peer-gossip-sleep-duration`

您可以尝试减少节点休眠的时间，然后再检查
有一些东西要发送给它的同行.

- `consensus.timeout-commit`

你也可以尝试降低`timeout-commit`(我们睡觉前的时间
提出下一个块).

-`p2p.addr-book-strict`

默认情况下，Tendermint 会在对等方的地址可路由之前进行检查
保存到通讯录.如果 IP
是 [有效且在允许范围内
范围](https://github.com/tendermint/tendermint/blob/27bd1deabe4ba6a2d9b463b8f3e3f1e31b993e61/p2p/netaddress.go#L209).

对于私有或本地网络，情况可能并非如此，您的 IP 范围通常是
严格限制和私人. 如果是这种情况，您需要设置 `addr-book-strict`
为`false`(将其关闭).

- `rpc.max-open-connections`

默认情况下，同时连接的数量是有限的，因为大多数操作系统
给你有限数量的文件描述符.

如果要接受更多的连接数，则需要增加
这些限制.

[Sysctls 调整系统以能够打开更多连接](https://github.com/satori-com/tcpkali/blob/master/doc/tcpkali.man.md#sysctls-to-tune-the-system -to-be-able-to-open-more-connections)

还必须增加过程文件限制，例如 通过`ulimit -n 8192`.

...对于 N 个连接，例如 50k:

```md
kern.maxfiles=10000+2*N         # BSD
kern.maxfilesperproc=100+2*N    # BSD
kern.ipc.maxsockets=10000+2*N   # BSD
fs.file-max=10000+2*N           # Linux
net.ipv4.tcp_max_orphans=N      # Linux

# For load-generating clients.
net.ipv4.ip_local_port_range="10000  65535"  # Linux.
net.inet.ip.portrange.first=10000  # BSD/Mac.
net.inet.ip.portrange.last=65535   # (Enough for N < 55535)
net.ipv4.tcp_tw_reuse=1         # Linux
net.inet.tcp.maxtcptw=2*N       # BSD

# If using netfilter on Linux:
net.netfilter.nf_conntrack_max=N
echo $((N/8)) > /sys/module/nf_conntrack/parameters/hashsize
```

存在用于限制 gRPC 连接数量的类似选项 -
`rpc.grpc-max-open-connections`.
