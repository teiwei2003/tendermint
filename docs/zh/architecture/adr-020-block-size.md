# ADR 020:限制块内的 txs 大小

## 变更日志

13-08-2018:初稿
15-08-2018:Dev 评论后的第二个版本
28-08-2018:Ethan 发表评论后的第三个版本
30-08-2018:AminoOverheadForBlock => MaxAminoOverheadForBlock
31-08-2018:边界证据和链 ID
13-01-2019:添加有关 MaxBytes 与 MaxDataBytes 的部分

## 语境

我们目前使用 MaxTxs 在提议一个区块时从内存池中获取 txs，
但是在解组块时强制执行 MaxBytes，所以我们可以很容易地提出一个
块太大而无效.

我们应该一起删除 MaxTxs 并坚持使用 MaxBytes，并有一个
`mempool.ReapMaxBytes`.

但是我们不能只收获 BlockSize.MaxBytes，因为 MaxBytes 是针对整个块的，
不适用于块内的 txs.有额外的氨基开销 + 实际
实际交易顶部的标题 + 证据 + 上次提交.
我们还可以考虑使用 MaxDataBytes 代替 MaxBytes 或除了 MaxBytes.

## MaxBytes 与 MaxDataBytes

[PR #3045](https://github.com/tendermint/tendermint/pull/3045) 建议
这里需要额外的澄清/理由，考虑到使用
除了或代替 MaxBytes 的 MaxDataBytes.

MaxBytes 对块的总大小提供了明确的限制，不需要
额外的计算，如果你想用它来限制资源使用，还有
关于优化 1MB 块左右的 Tendermint 的讨论已经相当多.
无论如何，我们需要一个块大小的最大值，以便我们可以避免
解组共识期间太大的块，似乎更多
直接为此提供一个固定的数字而不是一个
计算“MaxDataBytes + 您需要腾出空间的所有其他内容
(签名、证据、标题)”.MaxBytes 提供了一个简单的界限，因此我们可以
总是说“块小于 X MB”.

同时拥有 MaxBytes 和 MaxDataBytes 感觉像是不必要的复杂性.它是
MaxBytes 暗示最大大小并不特别令人惊讶
整个区块(不仅仅是 txs)，你只需要知道一个区块包含标题，
交易，证据，选票.为了更细粒度地控制包含在
块，有 MaxGas.在实践中，MaxGas 可能会做大部分
tx 节流和 MaxBytes 只是作为总数的上限
尺寸.应用程序可以将 MaxGas 用作 MaxDataBytes，只需将 gas 用于
每个 tx 都是它的大小(以字节为单位).

## 建议的解决方案

因此，我们应该

1)摆脱MaxTxs.
2) 将 MaxTxsBytes 重命名为 MaxBytes.

当我们需要从内存池中获取 ReapMaxBytes 时，我们计算上限如下:

```
ExactLastCommitBytes = {number of validators currently enabled} * {MaxVoteBytes}
MaxEvidenceBytesPerBlock = MaxBytes / 10
ExactEvidenceBytes = cs.evpool.PendingEvidence(MaxEvidenceBytesPerBlock) * MaxEvidenceBytes

mempool.ReapMaxBytes(MaxBytes - MaxAminoOverheadForBlock - ExactLastCommitBytes - ExactEvidenceBytes - MaxHeaderBytes)
```

其中 MaxVoteBytes、MaxEvidenceBytes、MaxHeaderBytes 和 MaxAminoOverheadForBlock
是在 `types` 包中定义的常量:

- MaxVoteBytes - 170 字节
- MaxEvidenceBytes - 364 字节
- MaxHeaderBytes - 476 字节(~276 字节散列 + 200 字节 - 50 UTF-8 编码
  链 ID 的符号在最坏情况下每个 4 个字节 + 氨基开销)
- MaxAminoOverheadForBlock - 8 字节(假设 MaxHeaderBytes 包括氨基
  编码头的开销，MaxVoteBytes - 编码投票等)

ChainID 最多需要绑定 50 个符号.

在收割证据时，我们使用 MaxBytes 来计算上限(例如 1/10)
为交易节省一些空间.

注意在获取内存池中的 `max int` 字节时，我们应该考虑到每个
交易将采用`len(tx)+aminoOverhead`，其中aminoOverhead=1-4 字节.

我们应该编写一个测试，如果底层结构发生变化，它就会失败，但是
MaxXXX 保持不变.

## 状态

实施的

## 结果

### 积极的

* 一种限制块大小的方法
* 更少的变量配置

### 消极的

* 底层结构改变时需要调整的常量

### 中性的
