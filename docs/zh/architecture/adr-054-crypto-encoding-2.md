# ADR 054:加密编码(第 2 部分)

## 变更日志

2020-2-27:创建
2020-4-16:更新

## 语境

氨基一直是生态系统中许多用户的痛点.虽然 Tendermint 不会受到氨基引入的性能下降的严重影响，但我们正在努力将编码格式转移到一种广泛采用的格式，[Protocol Buffers](https://developers.google.com/protocol-buffers).通过这次迁移，需要一个新的密钥编码标准.这将导致生态系统范围内的重大变化.

目前，amino 将键编码为 `<PrefixBytes> <Length> <ByteArray>`.

## 决定

以前，Tendermint 定义了在 Tendermint 和 Cosmos-SDK 中使用的所有关键类型.展望未来，Cosmos-SDK 将为密钥定义自己的 protobuf 类型.这将允许 Tendermint 仅定义在代码库中使用的密钥 (ed25519).
有机会仅定义 ed25519 (`bytes`) 的用法而不是将其设为 `oneof`，但这意味着 `oneof` 工作只会被推迟到以后的日期.当使用 `oneof` protobuf 类型时，我们必须手动切换可能的密钥类型，然后将它们传递给需要的接口.

为最大程度地减少用户头痛而采取的方法是，所有密钥编码都将转移到 protobuf，而在依赖氨基编码的情况下，将有自定义编组和解组功能.

Protobuf 消息:

```proto
message PubKey {
  oneof key {
    bytes ed25519 = 1;
  }

message PrivKey {
  oneof sum {
    bytes ed25519 = 1;
  }
}
```

> 注意:需要向后兼容的地方还不清楚.

当前所有模块都不依赖于氨基编码的字节，并且密钥不是用于创世的氨基编码，因此需要进行硬分叉升级以采用这些更改.

这项工作将分解为几个 PR，这项工作将合并为一个 proto-breakage 分支，所有 PR 将在合并之前进行审查:

1. protobuf 和 protobuf 消息的密钥编码
2. 将 Tendermint 类型移至 protobuf，主要是正在编码的类型.
3. 一一通过反应器，将氨基编码的信息转移到 protobuf.
4. 使用 cosmos-sdk 和/或 testnets repo 进行测试.

## 状态

实施的

## 结果

- 将密钥移动到 protobuf 编码，在需要向后兼容的地方，将使用氨基编组和解组功能.

### 积极的

- 协议缓冲区编码将不会改变.
- 从密钥中删除氨基开销将有助于 KSM.
- 拥有庞大的受支持语言生态系统.

### 消极的

- 需要硬分叉才能将其集成到正在运行的链中.

### 中性的

## 参考

> 是否有任何相关的 PR 评论、导致此问题的问题，或关于我们为何做出给定设计选择的参考文章？如果是这样，请在此处链接它们！

- {参考链接}
