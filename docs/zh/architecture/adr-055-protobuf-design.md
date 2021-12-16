# ADR 055:Protobuf 设计

## 变更日志

- 2020-4-15:创建(@marbar3778)
- 2020-6-18:更新(@marbar3778)

## 语境

目前我们在 Tendermint 中使用 [go-amino](https://github.com/tendermint/go-amino)。 Tendermint 团队不再维护 Amino(2020 年 4 月 15 日)，并且发现存在问题:

- https://github.com/tendermint/go-amino/issues/286
- https://github.com/tendermint/go-amino/issues/230
- https://github.com/tendermint/go-amino/issues/121

这些是用户可能遇到的一些已知问题。

Amino 支持快速原型设计和功能开发。虽然这很好，但氨基并没有提供预期的性能和开发人员便利。要使 Tendermint 被广泛采用作为 BFT 协议引擎，需要过渡到采用的编码格式。以下是一些可以探索的可能选项。

有几个选项可供选择:

- `Protobuf`:协议缓冲区是谷歌的语言中立、平台中立、可扩展的结构化数据序列化机制——想想 XML，但更小、更快、更简单。它支持无数种语言，并已在生产中得到验证多年。

- `FlatBuffers`:FlatBuffers 是一个高效的跨平台序列化库。 Flatbuffers 比 Protobuf 更有效，因为它没有解析/解包到第二个表示的速度。 FlatBuffers 已经在生产中进行了测试和使用，但并未被广泛采用。

- `CapnProto`:Cap'n Proto 是一种非常快速的数据交换格式和基于功能的 RPC 系统。 Cap'n Proto 没有编码/解码步骤。它尚未在整个行业中得到广泛采用。

- @erikgrinaker - https://github.com/tendermint/tendermint/pull/4623#discussion_r401163501
  ``
  Cap'n'Proto 很棒。它是由原始 Protobuf 开发人员之一编写的，用于修复其一些问题，并支持例如随机访问以处理大量消息而不将它们加载到内存中，以及在需要确定性时(例如在状态机中)非常有用的(选择加入)规范形式。也就是说，由于更广泛的采用，我怀疑 Protobuf 是更好的选择，尽管这让我有点难过，因为 Cap'n'Proto 在技术上更好。
  ``

## 决定

由于其性能和工具，将 Tendermint 过渡到 Protobuf。 Protobuf 背后的生态系统非常庞大，并且具有出色的[对多种语言的支持](https://developers.google.com/protocol-buffers/docs/tutorials)。

我们将通过将当前类型保持在当前形式(手写)并创建一个 `/proto` 目录来实现这一点，所有 `.proto` 文件都将保存在该目录中。在需要编码的地方，在磁盘和网络上，我们将调用 util 函数，将类型从手写的 go 类型转换为 protobuf 生成的类型。这符合 [buf](https://buf.build) 中推荐的文件结构。您可以在 [此处](https://buf.build/docs/lint-checkers#file_layout) 中找到有关此文件结构的更多信息。

通过采用这种设计，我们将支持未来对类型的更改，并允许更加模块化的代码库。

## 状态

实施的

## 结果

### 积极的

- 允许未来的模块化类型
- 更少的重构
- 允许将来将原始文件拉入规范存储库。
- 表现
- 多种语言的工具和支持

### 消极的

- 当开发人员更新类型时，他们需要确保也更新原型

### 中性的

## 参考
