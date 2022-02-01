# ADR 009:ABCI 用户体验改进

## 变更日志

23-06-2018:审查中的一些小修正
07-06-2018:基于与 Jae 讨论的一些更新
07-06-2018:与 ABCI v0.11 中发布的内容相匹配的初始草案

## 语境

ABCI 于 2015 年底首次推出.其目的是:

- 状态机与其复制引擎之间的通用接口
- 与编写状态机的语言无关
- 与驱动它的复制引擎无关

这意味着 ABCI 应该为可插拔应用程序和
可插拔的共识引擎.

为了实现这一点，它使用协议缓冲区 (proto3) 作为消息类型.占主导地位的
实现是在 Go 中.

经过最近在 github 上与社区的讨论，以下是
确定为痛点:

- 氨基编码类型
- 管理验证器集
- 在 protobuf 文件中导入

有关更多信息，请参阅 [references](#references).

### 进口

Go 中的原生 proto 库会生成不灵活且冗长的代码.
Go 社区中的许多人都采用了一个名为
[gogoproto](https://github.com/gogo/protobuf) 提供了一个
旨在改善开发人员体验的各种功能.
虽然 `gogoproto` 很好，但它创建了一个额外的依赖，并编译
已报告其他语言的 protobuf 类型在使用 `gogoproto` 时失败.

###氨基

Amino 是一种编码协议，旨在改善 protobuf 的不足.
它的目标是成为proto4.

很多人对protobuf不兼容感到沮丧，
并且要求在 ABCI 中完全使用氨基.

我们打算使 Amino 足够成功，以便我们最终可以将其用于 ABCI
消息类型直接.到那时它应该被称为proto4.同时，
我们希望它易于使用.

### 公钥

PubKeys 使用 Amino 编码(在此之前，go-wire).
理想情况下，公钥是一种接口类型，我们不知道所有的
实现类型，因此不适合使用 `oneof` 或 `enum`.

### 地址

ED25519公钥地址是Amino的RIPEMD160
编码的公钥.这在地址生成中引入了氨基依赖，
一个被广泛需要并且应该易于计算的功能
可能的.

### 验证器

要更改验证器集，应用程序可以返回验证器更新列表
与 ResponseEndBlock.在这些更新中，_必须_包含公钥，
因为 Tendermint 需要公钥来验证验证者签名.这
意味着 ABCI 开发人员必须使用 PubKeys.也就是说，这也将是
处理地址信息很方便，而且操作起来也很简单.

### 缺席验证器

Tendermint 还提供了 BeginBlock 中未签署的验证者列表
最后一块.这允许应用程序反映可用性行为
应用程序，例如通过惩罚没有包含选票的验证器
在提交中.

###初始化链

Tendermint 在此处传入验证器列表，仅此而已.它会
使应用程序能够控制初始验证器集.为了
例如，创世文件可以包含基于应用程序的信息
应用程序可以处理的初始验证器集以确定
初始验证器集.此外，InitChain 将受益于获得所有
创世信息.

### 标题

ABCI 在 RequestBeginBlock 中提供了 Header，因此应用程序可以有
有关区块链最新状态的重要信息.

## 决定

### 进口

远离 gogoproto.短期内，我们只维持一秒
protobuf 文件，没有 gogoproto 注释.在中期，我们将
复制 Golang 中的所有结构体并来回穿梭.在漫长的
术语，我们将使用氨基.

###氨基

为了在短期内简化 ABCI 应用程序开发，
Amino 将从 ABCI 中完全删除:

- 公钥编码不需要它
- 计算公钥地址不需要

也就是说，我们正在努力使 Amino 取得巨大成功，并成为 proto4.
为了在短期内促进采用和跨语言兼容性，Amino
v1 将:

- 与不包括`oneof`的proto3子集完全兼容
- 使用 Amino 前缀系统提供接口类型，而不是 `oneof`
  样式联合类型.

也就是说，将致力于提高 Amino v2 的性能
格式及其在加密应用程序中的可用性.

### 公钥

编码方案会感染软件.作为一个通用的中间件，ABCI 的目标是拥有
一些跨方案兼容性.为此，它别无选择，只能包括不透明
字节不时.虽然我们不会对这些强制执行氨基编码
字节数，我们需要提供一个类型系统.最简单的方法是
使用类型字符串.

公钥现在看起来像:

```
message PubKey {
    string type
    bytes data
}
```

其中 `type` 可以是:

- “ed225519”，带有`data = <raw 32-byte pubkey>`
- “secp256k1”，带有`data = <33-byte OpenSSL 压缩公钥>`

因为我们希望在这里保持灵活性，而且理想情况下，PubKey 将是一个
接口类型，我们不使用 `enum` 或 `oneof`.

### 地址

为了简化和改进计算地址，我们将其更改为 SHA256 的前 20 个字节
原始 32 字节公钥.

我们继续对 secp256k1 密钥使用比特币地址方案.

### 验证器

添加一个“字节地址”字段:

```
message Validator {
    bytes address
    PubKey pub_key
    int64 power
}
```

### RequestBeginBlock 和 AbsentValidators

为了简化这一点，RequestBeginBlock 将包含完整的验证器集，
包括每个验证者的地址和投票权，以及
使用布尔值表示他们是否投票:

```
message RequestBeginBlock {
  bytes hash
  Header header
  LastCommitInfo last_commit_info
  repeated Evidence byzantine_validators
}

message LastCommitInfo {
  int32 CommitRound
  repeated SigningValidator validators
}

message SigningValidator {
    Validator validator
    bool signed_last_block
}
```

请注意，在 RequestBeginBlock 的验证器中，我们不包含公钥. 公钥是
比地址大，在未来，有了量子计算机，
更大. 传递它们的开销，尤其是在快速同步期间，是
重要的.

此外，地址正在变得更易于计算，进一步删除
需要在此处包含公钥.

简而言之，ABCI 开发人员必须了解地址和公钥.

### ResponseEndBlock

由于 ResponseEndBlock 包含 Validator，它现在必须包含它们的地址.

###初始化链

更改 RequestInitChain 以向应用程序提供创世文件中的所有信息:

```
message RequestInitChain {
    int64 time
    string chain_id
    ConsensusParams consensus_params
    repeated Validator validators
    bytes app_state_bytes
}
```

更改 ResponseInitChain 以允许应用程序指定初始验证器集
和共识参数.

```
message ResponseInitChain {
    ConsensusParams consensus_params
    repeated Validator validators
}
```

### 标题

现在 Tendermint Amino 将与 proto3 兼容，即 ABCI 中的 Header
应该完全匹配 Tendermint 标头 - 然后它们将被编码
在 ABCI 和 Tendermint Core 中相同.

## 状态

实施的

## 结果

### 积极的

- 开发人员更容易在 ABCI 上构建
- ABCI 和 Tendermint 标头是相同的序列化

### 消极的

- 替代类型编码方案的维护开销
- 传递每个区块的所有验证器信息的性能开销(至少它的
  只有地址，而不是公钥)
- 重复类型的维护开销

### 中性的

- ABCI 开发人员必须了解验证器地址

## 参考

- [ABCI v0.10.3 规范(在此之前
  提案)](https://github.com/tendermint/abci/blob/v0.10.3/specification.rst)
- [ABCI v0.11.0 规范(实现本规范的初稿)
  提案)](https://github.com/tendermint/abci/blob/v0.11.0/specification.md)
- [Ed25519 地址](https://github.com/tendermint/go-crypto/issues/103)
- [InitChain 包含
  创世纪](https://github.com/tendermint/abci/issues/216)
- [公钥](https://github.com/tendermint/tendermint/issues/1524)
- [注意事项
  标题](https://github.com/tendermint/tendermint/issues/1605)
- [Gogoproto 问题](https://github.com/tendermint/abci/issues/256)
- [缺少验证器](https://github.com/tendermint/abci/issues/231)
