# ADR 018:ABCI 验证器改进

## 变更日志

016-08-2018:审查跟进: - 恢复提交轮次的更改 - 提醒删除公钥的理由 - 更新优点/缺点
05-08-2018:初稿

## 语境

ADR 009 围绕验证器和使用对 ABCI 进行了重大改进
氨基的。 在这里，我们跟进了一些额外的更改以改进命名
以及验证器消息的预期用途。

## 决定

### 验证器

目前一个 Validator 包含 `address` 和 `pub_key`，其中一个是
可选/不发送取决于用例。 相反，我们应该有一个
`Validator`(只有地址，用于RequestBeginBlock)
和一个 `ValidatorUpdate`(带有公钥，用于 ResponseEndBlock):

```
message Validator {
    bytes address
    int64 power
}

message ValidatorUpdate {
    PubKey pub_key
    int64 power
}
```

如 [ADR-009](adr-009-ABCI-design.md) 中所述，
`Validator` 不包含公钥，因为量子公钥是
相当大，并且将它们与每个块一起发送到整个 ABCI 会很浪费。
因此，想要利用 BeginBlock 中的信息的应用程序
_需要_以状态存储公钥(或使用效率低得多的懒惰方式
验证 BeginBlock 数据)。

### RequestBeginBlock

LastCommitInfo 当前有一个 `SigningValidator` 数组，其中包含
整个验证器集中每个验证器的信息。
相反，这应该称为“VoteInfo”，因为它是关于
验证者投票。

请注意，提交中的所有投票必须来自同一轮。

```
message LastCommitInfo {
  int64 round
  repeated VoteInfo commit_votes
}

message VoteInfo {
    Validator validator
    bool signed_last_block
}
```

### ResponseEndBlock

使用 ValidatorUpdates 而不是 Validators。那么很明显我们不需要
地址，我们确实需要一个公钥。

我们可以要求这里的地址以及健全性检查，但似乎没有
必要的。

###初始化链

对请求和响应都使用 ValidatorUpdates。初始链
与 BeginBlock 不同，是关于设置/更新初始验证器集
这只是信息性的。

## 状态

实施的

## 结果

### 积极的

- 阐明了验证者信息的不同用途之间的区别

### 消极的

- 应用程序仍必须将公钥存储在状态才能使用 RequestBeginBlock 信息

### 中性的

- ResponseEndBlock 不需要地址

## 参考

- [最新 ABCI 规范](https://github.com/tendermint/tendermint/blob/v0.22.8/docs/app-dev/abci-spec.md)
- [ADR-009](https://github.com/tendermint/tendermint/blob/v0.22.8/docs/architecture/adr-009-ABCI-design.md)
- [问题 #1712 - 不要发送 PubKey
  RequestBeginBlock](https://github.com/tendermint/tendermint/issues/1712)
