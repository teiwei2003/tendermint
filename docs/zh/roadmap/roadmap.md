# Tendermint 路线图

*上次更新:2021 年 10 月 8 日星期五*

本文档致力于让更广泛的 Tendermint 社区了解 Tendermint Core 的开发计划和优先级，以及我们期望何时交付功能。它旨在广泛告知 Tendermint 的所有用户，包括应用程序开发人员、节点运营商、集成商以及工程和研究团队。

任何希望提议工作成为此路线图的一部分的人都应该通过在规范中打开一个 [问题](https://github.com/tendermint/spec/issues/new/choose) 来这样做。错误报告和其他实施问题应在 [核心存储库](https://github.com/tendermint/tendermint) 中提出。

该路线图应被视为计划和优先事项的高级指南，而不是对时间表和可交付成果的承诺。路线图前面的功能通常比后面的功能更具体和详细。我们将定期更新此文档以反映当前状态。

升级分为两个部分:**史诗**，定义发布的功能，在很大程度上决定发布的时间；和**未成年人**，规模较小和优先级较低的功能，可能会出现在相邻版本中。

## V0.35(2021 年第三季度完成)
### 优先内存池

交易之前按照它们到达内存池的顺序添加到块中。通过“CheckTx”添加优先级字段使应用程序可以更好地控制将哪些事务放入块中。这在存在交易费用的情况下很重要。 [更多](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-067-mempool-refactor.md)

###重构P2P框架

Tendermint P2P 系统正在进行大规模的重新设计，以提高其性能和可靠性。此重新设计的第一阶段包含在 0.35 中。此阶段清理和解耦抽象，改进对等生命周期管理、对等地址处理并启用可插拔传输。它被实现为与之前的实现协议兼容。 [更多](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-062-p2p-architecture.md)

### 状态同步改进

在状态同步的初始版本之后，进行了一些改进。其中包括添加证据处理所需的 [Reverse Sync](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-068-reverse-sync.md)，引入 [ P2P 状态提供程序](https://github.com/tendermint/tendermint/pull/6807) 作为 RPC 端点的替代方案、用于调整吞吐量的新配置参数以及一些错误修复。

### 自定义事件索引 + PSQL 索引器

添加了一个新的“EventSink”接口，以允许替代 Tendermint 的专有交易索引器。我们还添加了一个 PostgreSQL 索引器实现，允许丰富的基于 SQL 的索引查询。 [更多](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-065-custom-event-indexing.md)

###小作品

- 重新组织了几个 Go 包，使公共 API 和实现细节之间的区别更加清晰。
- 块索引器来索引开始块和结束块事件。 [更多](https://github.com/tendermint/tendermint/pull/6226)
- 块、状态、证据和光存储键被重新设计以保持字典顺序。此更改需要数据库迁移。 [更多](https://github.com/tendermint/tendermint/pull/5771)
- Tendermint 模式介绍。此更改的一部分包括运行仅运行 PEX 反应器的单独种子节点的可能性。 [更多](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-052-tendermint-mode.md)

## V0.36(预计 2022 年第一季度)

### ABCI++

彻底改革应用程序和共识之间的现有接口，使应用程序能够更好地控制区块构建。 ABCI++ 添加了新的钩子，允许在交易进入区块之前修改交易，在投票前验证区块，将签名信息注入投票，以及在协议后更紧凑地交付区块(以允许并发执行)。 [更多](https://github.com/tendermint/spec/blob/master/rfc/004-abci%2B%2B.md)

### 基于提议者的时间戳

基于提议者的时间戳是 [BFT 时间](https://docs.tendermint.com/master/spec/consensus/bft-time.html) 的替代品，其中提议者选择时间戳，验证者仅在以下情况下对区块进行投票时间戳被视为*及时*。这增加了对准确本地时钟的依赖，但作为交换，区块时间更可靠，更能抵抗故障。这在轻客户端、IBC 中继器、CosmosHub 通货膨胀和启用签名聚合方面具有重要的用例。 [更多](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-071-proposer-based-timestamps.md)

### 软升级

我们正在开发一套工具和模式，使节点运营商和应用程序开发人员能够更轻松地快速安全地升级到更新版本的 Tendermint。 [更多](https://github.com/tendermint/spec/pull/222)

###小作品

- 删除“遗留”的 P2P 框架，并清理 P2P 包。 [更多](https://github.com/tendermint/tendermint/issues/5670)
- 从本地 ABCI 客户端删除全局互斥锁以启用应用程序控制的并发。 [更多](https://github.com/tendermint/tendermint/issues/7073)
- 为轻客户端启用 P2P 支持
- 服务的节点编排 + 节点初始化和可组合性
- 删除多个数据结构中的冗余。删除未使用的组件，例如块同步 v2 反应器、RPC 层中的 gRPC 和基于套接字的远程签名器。
- 通过引入更多指标来提高节点可见性

## V0.37(预计 2022 年第三季度)

### 完成 P2P 重构

完成 P2P 系统的最后阶段。正在进行的研究和规划正在决定是否采用 [libp2p](https://libp2p.io/)，替代“MConn”的传输方式，例如 [QUIC](https://en.wikipedia.org/wiki/ QUIC) 和握手/身份验证协议，例如 [Noise](https://noiseprotocol.org/)。研究更先进的八卦技巧。

### 简化存储引擎

Tendermint 目前有一个抽象，允许支持多个数据库后端。这种普遍性会导致维护开销并干扰 Tendermint 可以使用的特定于应用程序的优化(ACID 保证等)。我们计划集中在一个数据库上并简化 Tendermint 存储引擎。 [更多](https://github.com/tendermint/tendermint/pull/6897)

### 评估进程间通信

Tendermint 节点目前与其他进程有多个通信领域(例如 ABCI、远程签名者、P2P、JSONRPC、websockets、事件)。其中许多有多个实现，其中一个就足够了。巩固和清理IPC。 [更多](https://github.com/tendermint/tendermint/blob/master/docs/rfc/rfc-002-ipc-ecosystem.md)

###小作品

- 失忆症攻击处理。 [更多](https://github.com/tendermint/tendermint/issues/5270)
- 删除/更新共识 WAL。 [更多](https://github.com/tendermint/tendermint/issues/6397)
- 签名聚合。 [更多](https://github.com/tendermint/tendermint/issues/1319)
- 删除 gogoproto 依赖项。 [更多](https://github.com/tendermint/tendermint/issues/5446)

## V1.0(预计 2022 年第四季度)

具有与 V0.37 相同的功能集，但侧重于测试、协议正确性和细微调整，以确保产品稳定。这些工作可能包括扩展[共识测试框架](https://github.com/tendermint/tendermint/issues/5920)、使用金丝雀/长寿命测试网和更大的集成测试。

## 发布 1.0 工作

- 使用擦除编码和/或紧凑块改进块传播。 [更多](https://github.com/tendermint/spec/issues/347)
- 共识引擎重构
- 双向 ABCI
- 随机领导选举
- ZK证明/其他加密原语
- 多链 Tendermint
