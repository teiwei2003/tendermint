# ADR 016:协议版本

## 去做

- 我们如何/应该对经过身份验证的加密握手本身进行版本控制(即。
  P2PVersion的前期协议协商)
- 我们如何/应该对 ABCI 本身进行版本控制？它应该被吸收吗
  块版本？

## 变更日志

- 18-09-2018:在实施上做了一些工作后更新
    - ABCI 握手需要独立于启动应用程序发生
      conns 所以我们可以看到结果
    - 增加关于ABCI协议版本的问题
- 16-08-2018:与 SDK 团队讨论后的更新
    - 从 Header/ABCI 中删除下一版本的信号
- 03-08-2018:与 Jae 讨论的更新:
  - ProtocolVersion 包含 Block/AppVersion，而不是 Current/Next
  - 使用 EndBlock 字段将信号升级到 Tendermint
  - 不要通过版本限制对等兼容性以简化同步旧节点
- 28-07-2018:审核更新
  - 分成两个 ADR - 一个用于协议，一个用于链
  - 在标题中包含升级信号
- 16-07-2018:初始草案 - 最初是协议和链的联合 ADR
  版本

## 语境

在这里，我们关注与软件无关的协议版本。

软件版本由 SemVer 涵盖并在别处描述。
它与协议描述无关，只要说如果有任何协议版本
更改，软件版本会更改，但不一定反之亦然。

为了方便/诊断，应在 NodeInfo 中包含软件版本。

我们也对跨不同区块链的版本控制感兴趣
有意义的方式，例如区分有争议的分支
硬分叉。我们将其留待稍后的 ADR。

## 要求

我们需要对可以独立升级的区块链组件进行版本控制。
我们需要以一种可扩展和可维护的方式来做——我们不能只是乱扔垃圾
带条件的代码。

我们可以考虑协议的完整版本包含以下子版本:
BlockVersion、P2PVersion、AppVersion。这些版本反映了主要的子组件
可能以不同速度和不同方式一起发展的软件，
如下所述。

BlockVersion 定义了区块链数据结构的核心和
应该不经常改变。

P2PVersion 定义了对等点如何相互连接和通信 - 它是
不是区块链数据结构的一部分，而是定义用于构建区块链的协议
区块链。它可能会逐渐改变。

AppVersion 决定了我们如何计算特定于应用程序的信息，例如
AppHash 和结果。

所有这些版本都可能在区块链的整个生命周期中发生变化，我们需要
能够帮助新节点跨版本更改同步。这意味着我们必须愿意
连接到旧版本的对等点。

###块版本

- 所有tendermint 散列数据结构(标题、投票、交易、响应等)。
  - 注意交易的语义可能会根据 AppVersion 发生变化，但 txs 被 merklize 到 header 的方式是 BlockVersion 的一部分
- 它应该是最不频繁/最不可能改变的。
  - Tendermint 应该是稳定的 - 这只是原子广播。
  - 我们可以在一年内开始考虑使用 Tendermint v2.0
- 很容易从一个块的序列化形式确定它的版本

### P2P 版本

- 所有 p2p 和反应器消息传递(消息、可检测行为)
- 将随着反应堆的发展而逐渐改变以提高性能并支持新功能 - 例如在内存池中提议的新消息类型 BatchTx 和共识中的 HasBlockPart
- 很容易从第一个序列化消息中确定对等体的版本
- 新版本必须至少兼容一个旧版本才能逐步升级

### 应用程序版本

- ABCI 状态机(交易、开始/结束块行为、提交散列)
- 行为和消息类型将在链的生命周期中突然改变
- 需要尽量减少代码的复杂性，以支持不同高度的不同 AppVersions
- 理想情况下，软件的每个版本一次仅支持一个_single_ AppVersion
  - 这意味着我们在不同的高度检查不同版本的软件，而不是乱扔代码
    带条件
  - 最小化跨 AppVersion 所需的数据迁移次数(即，大多数 AppVersion 应该能够从磁盘读取与以前的 AppVersion 相同的状态)。

## 理想的

软件的每个组件都以模块化方式独立版本控制，易于混合搭配和升级。

## 提议

BlockVersion、AppVersion、P2PVersion，每一个都是单调递增的uint64。

要使用这些版本，我们需要更新区块 Header、p2p NodeInfo 和 ABCI。

### 标题

Block Header 应该包含一个 `Version` 结构作为它的第一个字段，例如:

```
type Version struct {
    Block uint64
    App uint64
}
```

这里，`Version.Block` 定义了当前块的规则，而
`Version.App` 定义了处理最后一个块并计算的应用程序版本
当前区块中的“AppHash”。 它们一起提供了完整的描述
共识关键协议。

由于我们已经确定了 proto3 标头，因此从序列化标头中读取 BlockVersion 的能力是一致的。

使用 Version 结构使我们可以更灵活地添加字段而不会破坏
标题。

ProtocolVersion 结构体包括 Block 和 App 版本 - 它应该
作为共识关键协议的完整描述。

###节点信息

NodeInfo 应该包含一个 Version 结构作为它的第一个字段，例如:

```
type Version struct {
    P2P uint64
    Block uint64
    App uint64

    Other []string
}
```

请注意，这有效地使 `Version.P2P` 成为 NodeInfo 中的第一个字段，因此它
如果需要方便升级，应该很容易从序列化的标头中读取它。

此处的“Version.Other”应包括附加信息，例如软件客户端的名称和
它是 SemVer 版本 - 这只是为了方便。例如。
`tendermint-core/v0.22.8`。它是一个 `[]string` 所以它可以包含有关
Tendermint 的版本、应用程序的版本、Tendermint 库的版本等。

### ABCI

由于 ABCI 负责保持 Tendermint 和应用程序同步，我们
需要通过它来传达版本信息。

在启动时，我们使用 Info 来执行基本的握手。它应该包括所有
版本信息。

我们还需要能够在区块链的生命周期中更新版本。这
这样做的自然场所是 EndBlock。

请注意，目前握手的结果未在任何地方公开，因为
握手发生在`proxy.AppConns` 抽象内部。我们将需要
从 `proxy` 包中删除握手，以便我们可以独立调用它
并获取结果，其中应包含应用程序版本。

#### 信息

RequestInfo 应该添加对协议版本的支持，例如:

```
message RequestInfo {
  string version
  uint64 block_version
  uint64 p2p_version
}
```

同样，ResponseInfo 应该返回版本:

```
message ResponseInfo {
  string data

  string version
  uint64 app_version

  int64 last_block_height
  bytes last_block_app_hash
}
```

现有的 `version` 字段应该被称为 `software_version` 但我们离开
它们现在可以减少破坏性更改的数量。

#### 结束块

可以使用新字段或使用
现有的“标签”。 由于我们正试图传达信息
包含在 Tendermint 区块头中，它应该是 ABCI 原生的，而不是
通过标签中的某种方案嵌入的东西。 因此，版本更新应该
通过 EndBlock 进行通信。

EndBlock 已经包含`ConsensusParams`。 我们可以添加版本信息到
ConsensusParams 也是:

```
message ConsensusParams {

  BlockSize block_size
  EvidenceParams evidence_params
  VersionParams version
}

message VersionParams {
    uint64 block_version
    uint64 app_version
}
```

现在，`block_version` 将被忽略，因为我们不允许块版本
待更新。如果设置了 `app_version`，它表示应用程序的
协议版本已更改，新的`app_version` 将包含在
下一个块的`Block.Header.Version.App`。

###块版本

BlockVersion 包含在 Header 和 NodeInfo 中。

更改 BlockVersion 应该很少发生，理想情况下仅适用于
关键升级。目前，它不是用 ABCI 编码的，尽管它总是
可以使用标签向外部进程发出信号以协调升级。

注意以太坊不必进行这样的升级(一切都在状态机级别，AFAIK)。

### P2P 版本

P2PVersion 不包含在区块头中，只包含在 NodeInfo 中。

P2PVersion 是 NodeInfo 中的第一个字段。 NodeInfo 也是 proto3，所以这很容易读出。

请注意，在发送消息时，我们需要 peer/reactor 协议来考虑 peers 的版本:

- 不要发送他们不理解的信息
- 不要发送他们不期望的消息

这样做将特定于正在进行的升级。

请注意，我们还在 NodeInfo 中包含了反应器通道列表，并且已经不会为对等方不理解的通道发送消息。
如果升级总是使用新通道，这会简化向后兼容的开发成本。

注意 NodeInfo 仅在经过身份验证的加密握手后交换，以确保它是私有的。
在加密之前进行任何版本交换都可能被视为信息泄漏，尽管我不确定
与能够升级协议相比，这有多重要。

XXX:如果需要，我们可以改变第一条消息的第一个字节的含义来编码握手版本吗？
这是 32 字节 ed25519 公钥的第一个字节。

### 应用程序版本

AppVersion 也包含在块 Header 和 NodeInfo 中。

AppVersion 本质上定义了 AppHash 和 LastResults 的计算方式。

### 对等兼容性

基于版本限制对等兼容性很复杂，因为需要
帮助可能在旧版本上的旧同行同步区块链。

我们可能会想说我们只连接到具有相同
AppVersion 和 BlockVersion(因为这些定义了关键的共识
计算)，以及 P2PVersions 的选择列表(即那些与
我们的)，但是我们需要为连接到同龄人与
正确的 Block/AppVersion 对应于它们所在的高度。

目前，我们将连接到任何版本的对等点并限制兼容性
仅基于 ChainID。我们对peer 留下了更多的限制性规则
与未来提案的兼容性。

### 未来的变化

支持 `/unsafe_stop?height=_` 端点告诉 Tendermint 在给定高度关闭可能很有价值。
这可以由监督升级的外部管理器进程使用
检查并安装新的软件版本并重新启动该过程。它
将订阅相关的升级事件(需要实现)并调用 `/unsafe_stop` 在
正确的高度(当然只有在得到用户的批准后！)

## 结果

### 积极的

- 使 ABCI 原生的 Tendermint 和应用程序版本更清晰
  沟通他们
- 明确区分协议版本和软件版本以
  促进其他语言的实现
- 以易于识别的方式包含在关键数据结构中的版本
- 允许提议者发出升级信号，并允许应用程序决定何时实际更改
  版本(并开始发出新版本的信号)

### 中性的

- 不清楚如何对初始 P2P 握手本身进行版本控制
- 尚未使用版本来限制对等兼容性
- 新版本的信号通过提议者发生，并且必须是
  在应用程序中记录/跟踪。

### 消极的

- 向 ABCI 添加更多字段
- 意味着单个代码库必须能够处理多个版本
