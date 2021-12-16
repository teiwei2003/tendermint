# 块结构

Tendermint 共识引擎记录所有协议
绝大多数节点进入区块链，在所有节点之间复制
节点。 该区块链可通过各种 RPC 端点访问，主要是
`/block?height=` 获取完整的块，以及
`/blockchain?minHeight=_&maxHeight=_` 获取头部列表。 但是什么
究竟是存储在这些块？

[规范](https://github.com/tendermint/spec/blob/8dd2ed4c6fe12459edeb9b783bdaaaeb590ec15c/spec/core/data_structures.md) 包含每个组件的详细描述 - 这是开始的最佳位置。

要深入挖掘，请查看 [类型包文档](https://godoc.org/github.com/tendermint/tendermint/types)。
