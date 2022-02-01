# 块同步
*以前称为快速同步*

在工作量证明区块链中，与链同步是一样的
与共识保持同步的过程:下载块，以及
寻找总工作量最多的那个.在股权证明中，
共识过程更复杂，因为它涉及多轮
节点之间的通信以确定应该是什么块
接下来犯了.使用此过程与区块链同步
从头开始可能需要很长时间.直接下载会快很多
阻止并检查验证器的默克尔树而不是实时运行
共识八卦协议.

## 使用块同步

为了支持更快的同步，Tendermint 提供了一个“blocksync”模式，它
默认情况下启用，并且可以在`config.toml`或通过
`--blocksync.enable=false`.

在这种模式下，Tendermint 守护进程将同步数百倍
而不是使用实时共识过程.一旦追上，
守护进程将退出块同步并进入正常的共识模式.
运行一段时间后，节点被认为“赶上”，如果它
至少有一个 peer 并且它的高度至少和 max 一样高
报告的同龄人身高.见 [IsCaughtUp
方法](https://github.com/tendermint/tendermint/blob/b467515719e686e4678e6da4e102f32a491b85a0/blockchain/pool.go#L128).

注意:Block Sync 有多个版本.请使用 v0，因为不再支持其他版本.
  如果你想使用不同的版本，你可以通过更改 `config.toml` 中的版本来实现:

```toml
#######################################################
###       Block Sync Configuration Connections       ###
#######################################################
[blocksync]

# If this node is many blocks behind the tip of the chain, BlockSync
# allows them to catchup quickly by downloading blocks in parallel
# and verifying their commits
enable = true

# Block Sync version to use:
#   1) "v0" (default) - the standard Block Sync implementation
#   2) "v2" - DEPRECATED, please use v0
version = "v0"
```

如果我们滞后得足够多，我们应该回到块同步，但是
这是一个 [公开问题](https://github.com/tendermint/tendermint/issues/129).

## 块同步事件
当tendermint区块链核心启动时，它可能会切换到`block-sync`
模式赶上状态到当前网络的最佳高度. 核心将发出
一个快速同步事件来公开当前状态和同步高度. 一旦抓住
网络最佳高度，它将切换到状态同步机制，然后发出
另一个用于公开快速同步“完成”状态和状态“高度”的事件.

用户可以通过订阅`EventQueryBlockSyncStatus`来查询事件
详情请查看 [types](https://pkg.go.dev/github.com/tendermint/tendermint/types?utm_source=godoc#pkg-constants).

## 执行

要阅读有关实现的更多信息，请参阅 [reactor doc](./reactor.md) 和 [implementation doc](./implementation.md)
