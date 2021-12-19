# アーキテクチャ決定記録(ADR)

これは、テンダーミントプロジェクトのすべての高レベルのアーキテクチャ上の決定が記録される場所です。

ADRの概念について詳しくは、この[ブログ投稿](https://product.reverb.com/documenting-architecture-decisions-the-reverb-way-a3563bb24bd0#.78xhdix6t)を参照してください。

ADRは以下を提供する必要があります。

-関連する目標の背景と現在の状況
-目標を達成するために提案された変更
-長所と短所の概要
- 参照する
-変更ログ

ADRと仕様の違いに注意してください。 ADRは、コンテキスト、直感、推論、および
構造を変更する理由、または何かの構造
新着。仕様は、すべてのコンテンツのより圧縮された単純化された要約を提供します。
今日は立っています。

欠落している決定を見つけた場合は、ディスカッションに電話し、ここに新しい決定を記録してから、一致するようにコードを変更してください。

文脈/背景は現在形で書かれるべきであることに注意してください。

## コンテンツ

### 実装

-[ADR-001:ログ](./ adr-001-logging.md)
-[ADR-002:イベントサブスクリプション](./ adr-002-event-subscription.md)
-[ADR-003:ABCI-APP-RPC](./ adr-003-abci-app-rpc.md)
-[ADR-004:履歴バリデーター](./ adr-004-historical-validators.md)
-[ADR-005:コンセンサス-パラム](./ adr-005-consensus-params.md)
-[ADR-008:Priv-Validator](./ adr-008-priv-validator.md)
-[ADR-009:ABCI-Design](./ adr-009-ABCI-design.md)
-[ADR-010:暗号化の変更](./ adr-010-crypto-changes.md)
-[ADR-011:監視](./ adr-011-monitoring.md)
-[ADR-014:Secp-Malleability](./ adr-014-secp-malleability.md)
-[ADR-015:暗号化エンコーディング](./ adr-015-crypto-encoding.md)
-[ADR-016:プロトコルバージョン](./ adr-016-protocol-versions.md)
-[ADR-017:チェーンバージョン](./ adr-017-chain-versions.md)
-[ADR-018:ABCI-Validators](./ adr-018-ABCI-Validators.md)
-[ADR-019:マルチシグ](./ adr-019-multisigs.md)
-[ADR-020:ブロックサイズ](./ adr-020-block-size.md)
-[ADR-021:ABCI-イベント](./ adr-021-abci-events.md)
-[ADR-025:送信](./ adr-025-commit.md)
-[ADR-026:General-Merkle-Proof](./ adr-026-general-merkle-proof.md)
-[ADR-033:パブリッシュおよびサブスクライブ](./ adr-033-pubsub.md)
-[ADR-034:Priv-Validator-File-Structure](./ adr-034-priv-validator-file-structure.md)
-[ADR-043:Blockchain-RiRi-Org](./ adr-043-blockchain-riri-org.md)
-[ADR-044:Lite-Client-With-Weak-Subjectivity](./ adr-044-lite-client-with-weak-subjectivity.md)
-[ADR-046:Light-Client-Implementation](./ adr-046-light-client-implementation.md)
-[ADR-047:Handling-Evidence-From-Light-Client](./ adr-047-handling-evidence-from-light-client.md)
-[ADR-051:二重署名リスクの削減](./ adr-051-double-signing-risk-reduction.md)
-[ADR-052:Tendermint-Mode](./ adr-052-tendermint-mode.md)
-[ADR-053:状態同期プロトタイプ](./ adr-053-state-sync-prototype.md)
-[ADR-054:Crypto-Encoding-2](./ adr-054-crypto-encoding-2.md)
-[ADR-055:Protobuf-Design](./ adr-055-protobuf-design.md)
-[ADR-056:Light-Client-Amnesia-Attacks](./ adr-056-light-client-amnesia-attacks.md)
-[ADR-059:エビデンスの構成とライフサイクル](./ adr-059-evidence-composition-and-lifecycle.md)
-[ADR-062:P2P-アーキテクチャ](./ adr-062-p2p-architecture.md)
-[ADR-063:Privval-gRPC](./ adr-063-privval-grpc.md)
-[ADR-066:E2E-テスト](./ adr-066-e2e-testing.md)
-[ADR-072:コメントリクエストの再開](./ adr-072-request-for-comments.md)

### 受け入れる

-[ADR-006:Trust-Metric](./ adr-006-trust-metric.md)
-[ADR-024:署名バイト](./ adr-024-sign-bytes.md)
-[ADR-035:ドキュメント](./ adr-035-documentation.md)
-[ADR-039:Peer-Behaviour](./ adr-039-peer-behaviour.md)
-[ADR-060:Go-API-Stability](./ adr-060-go-api-stability.md)
-[ADR-061:P2P-Refactor-Scope](./ adr-061-p2p-refactor-scope.md)
-[ADR-065:カスタムイベントインデックス](./ adr-065-custom-event-indexing.md)
-[ADR-068:リバース同期](./ adr-068-reverse-sync.md)
-[ADR-067:メモリプールリファクタリング](./ adr-067-mempool-refactor.md)

### ごみ

-[ADR-023:ABCI-Propose-tx](./ adr-023-ABCI-propose-tx.md)
-[ADR-029:Check-Tx-Consensus](./ adr-029-check-tx-consensus.md)
-[ADR-058:イベントハッシュ](./ adr-058-event-hashing.md)


### 提案

-[ADR-007:Trust-Metric-Usage](./ adr-007-trust-metric-usage.md)
-[ADR-012:Peer-Transport](./ adr-012-peer-transport.md)
-[ADR-013:対称暗号化](./ adr-013-symmetric-crypto.md)
-[ADR-022:ABCI-エラー](./ adr-022-abci-errors.md)
-[ADR-030:コンセンサスリファクタリング](./ adr-030-consensus-refactor.md)
-[ADR-037:Deliver-Block](./ adr-037-deliver-block.md)
-[ADR-038:ゼロ以外の開始高さ](./ adr-038-non-zero-start-height.md)
-[ADR-041:Proposer-Selection-via-ABCI](./ adr-041-proposer-selection-via-abci.md)
-[ADR-045:ABCI-エビデンス](./ adr-045-abci-evidence.md)
-[ADR-057:RPC](./ adr-057-RPC.md)
-[ADR-069:ノードの初期化](./ adr-069-flexible-node-initialization.md)
-[ADR-071:提案者のタイムスタンプに基づく](adr-071-proposer-based-timestamps.md)
