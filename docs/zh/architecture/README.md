# 架构决策记录 (ADR)

这是记录tendermint项目中所有高级架构决策的位置.

您可以在此 [博客文章](https://product.reverb.com/documenting-architecture-decisions-the-reverb-way-a3563bb24bd0#.78xhdix6t) 中阅读有关 ADR 概念的更多信息.

ADR 应提供:

- 相关目标和当前状态的背景
- 为实现目标而提出的变更
- 利弊总结
- 参考
- 变更日志

请注意 ADR 和规范之间的区别. ADR 提供上下文、直觉、推理和
改变架构的理由，或某物的架构
新的.该规范对所有内容进行了更加压缩和简化的总结，因为
它今天站立.

如果发现缺少记录的决策，请召集讨论，在此处记录新决策，然后修改代码以匹配.

注意上下文/背景应该用现在时写.

## 目录

### 实施的

- [ADR-001: 日志](./adr-001-logging.md)
- [ADR-002: 事件订阅](./adr-002-event-subscription.md)
- [ADR-003: ABCI-APP-RPC](./adr-003-abci-app-rpc.md)
- [ADR-004:历史验证器](./adr-004-historical-validators.md)
- [ADR-005: Consensus-Params](./adr-005-consensus-params.md)
- [ADR-008:Priv-Validator](./adr-008-priv-validator.md)
- [ADR-009: ABCI-Design](./adr-009-ABCI-design.md)
- [ADR-010:加密变更](./adr-010-crypto-changes.md)
- [ADR-011:监控](./adr-011-monitoring.md)
- [ADR-014:Secp-Malleability](./adr-014-secp-malleability.md)
- [ADR-015: 加密编码](./adr-015-crypto-encoding.md)
- [ADR-016: 协议版本](./adr-016-protocol-versions.md)
- [ADR-017:链版本](./adr-017-chain-versions.md)
- [ADR-018: ABCI-Validators](./adr-018-ABCI-Validators.md)
- [ADR-019: Multisigs](./adr-019-multisigs.md)
- [ADR-020:块大小](./adr-020-block-size.md)
- [ADR-021: ABCI-Events](./adr-021-abci-events.md)
- [ADR-025:提交](./adr-025-commit.md)
- [ADR-026: General-Merkle-Proof](./adr-026-general-merkle-proof.md)
- [ADR-033: 发布订阅](./adr-033-pubsub.md)
- [ADR-034:Priv-Validator-File-Structure](./adr-034-priv-validator-file-structure.md)
- [ADR-043: Blockchain-RiRi-Org](./adr-043-blockchain-riri-org.md)
- [ADR-044: Lite-Client-With-Weak-Subjectivity](./adr-044-lite-client-with-weak-subjectivity.md)
- [ADR-046: Light-Client-Implementation](./adr-046-light-client-implementation.md)
- [ADR-047:Handling-Evidence-From-Light-Client](./adr-047-handling-evidence-from-light-client.md)
- [ADR-051:双重签名风险降低](./adr-051-double-signing-risk-reduction.md)
- [ADR-052: Tendermint-Mode](./adr-052-tendermint-mode.md)
- [ADR-053:状态同步原型](./adr-053-state-sync-prototype.md)
- [ADR-054: Crypto-Encoding-2](./adr-054-crypto-encoding-2.md)
- [ADR-055: Protobuf-Design](./adr-055-protobuf-design.md)
- [ADR-056: Light-Client-Amnesia-Attacks](./adr-056-light-client-amnesia-attacks.md)
- [ADR-059:证据组合和生命周期](./adr-059-evidence-composition-and-lifecycle.md)
- [ADR-062: P2P-Architecture](./adr-062-p2p-architecture.md)
- [ADR-063: Privval-gRPC](./adr-063-privval-grpc.md)
- [ADR-066: E2E-Tes​​ting](./adr-066-e2e-testing.md)
- [ADR-072:恢复评论请求](./adr-072-request-for-comments.md)

### 接受

- [ADR-006: Trust-Metric](./adr-006-trust-metric.md)
- [ADR-024:符号字节](./adr-024-sign-bytes.md)
- [ADR-035:文档](./adr-035-documentation.md)
- [ADR-039: Peer-Behaviour](./adr-039-peer-behaviour.md)
- [ADR-060: Go-API-Stability](./adr-060-go-api-stability.md)
- [ADR-061: P2P-Refactor-Scope](./adr-061-p2p-refactor-scope.md)
- [ADR-065:自定义事件索引](./adr-065-custom-event-indexing.md)
- [ADR-068: 反向同步](./adr-068-reverse-sync.md)
- [ADR-067:内存池重构](./adr-067-mempool-refactor.md)

### 拒绝

- [ADR-023: ABCI-Propose-tx](./adr-023-ABCI-propose-tx.md)
- [ADR-029: Check-Tx-Consensus](./adr-029-check-tx-consensus.md)
- [ADR-058: 事件哈希](./adr-058-event-hashing.md)


### 建议的

- [ADR-007: Trust-Metric-Usage](./adr-007-trust-metric-usage.md)
- [ADR-012: Peer-Transport](./adr-012-peer-transport.md)
- [ADR-013:对称加密](./adr-013-symmetric-crypto.md)
- [ADR-022: ABCI-Errors](./adr-022-abci-errors.md)
- [ADR-030:共识重构](./adr-030-consensus-refactor.md)
- [ADR-037:Deliver-Block](./adr-037-deliver-block.md)
- [ADR-038: 非零起始高度](./adr-038-non-zero-start-height.md)
- [ADR-041: Proposer-Selection-via-ABCI](./adr-041-proposer-selection-via-abci.md)
- [ADR-045: ABCI-Evidence](./adr-045-abci-evidence.md)
- [ADR-057: RPC](./adr-057-RPC.md)
- [ADR-069: 节点初始化](./adr-069-flexible-node-initialization.md)
- [ADR-071:基于提议者的时间戳](adr-071-proposer-based-timestamps.md)
