# 配置

这里我们描述了围绕内存池的配置选项。
出于本文档的目的，它们被描述为
在 toml 文件中，但其中一些也可以作为
环境变量。

配置:

```toml
[mempool]

recheck = true
broadcast = true
wal-dir = ""

# Maximum number of transactions in the mempool
size = 5000

# Limit the total size of all txs in the mempool.
# This only accounts for raw transactions (e.g. given 1MB transactions and
# max-txs-bytes=5MB, mempool will only accept 5 transactions).
max-txs-bytes = 1073741824

# Size of the cache (used to filter transactions we saw earlier) in transactions
cache-size = 10000

# Do not remove invalid transactions from the cache (default: false)
# Set to true if it's not possible for any invalid transaction to become valid
# again in the future.
keep-invalid-txs-in-cache = false

# Maximum size of a single transaction.
# NOTE: the max size of a tx transmitted over the network is {max-tx-bytes}.
max-tx-bytes = 1048576

# Maximum size of a batch of transactions to send to a peer
# Including space needed by encoding (one varint per transaction).
# XXX: Unused due to https://github.com/tendermint/tendermint/issues/5796
max-batch-bytes = 0
```

<!-- Flag: `--mempool.recheck=false`

Environment: `TM_MEMPOOL_RECHECK=false` -->

##复查

重新检查确定内存池是否重新检查所有挂起
区块提交后的交易。一次块
已提交，内存池将删除所有有效交易
已成功包含在块中。

如果 `recheck` 为真，那么它将重新运行 CheckTx
具有新区块状态的所有剩余交易。

## 播送

确定此节点是否闲聊任何有效交易
到达内存池。默认是八卦任何事情
通过 checktx。如果禁用此选项，则事务不会
八卦，而是存储在本地并添加到下一个
阻止这个节点是提议者。

## WalDir

这定义了内存池写入预写的目录
日志。这些文件可用于重新加载未广播的
节点崩溃时的交易。

如果传入的目录是绝对路径，则wal文件为
在那里创建。如果目录是相对路径，则路径为
附加到tendermint进程的主目录到
生成wal目录的绝对路径
(默认`$HOME/.tendermint` 或通过`TM_HOME` 或`--home` 设置)

## 尺寸

大小定义了存储在内存池中的交易总量。默认值为“5_000”，但可以调整为您想要的任何数字。尺寸越大，节点上的应变越大。

## 最大交易字节数

最大交易字节数定义了内存池中所有交易的总大小。默认值为 1 GB。

## 缓存大小

缓存大小决定了我们已经看到的缓存保存事务的大小。缓存的存在是为了避免每次收到交易时都运行 `checktx`。

## 在缓存中保留无效事务

将无效事务保留在缓存中确定是否应该驱逐缓存中无效的事务。此处的无效交易可能意味着该交易可能依赖于未包含在区块中的不同 tx。

## 最大交易字节数

最大交易字节数定义了一个交易可以用于您的节点的最大大小。如果您希望您的节点仅跟踪较小的交易，则需要更改此字段。默认为 1MB。

## 最大批处理字节

最大批处理字节数定义了节点将发送到对等方的字节数。默认值为 0。

> 注意:由于 https://github.com/tendermint/tendermint/issues/5796 未使用
