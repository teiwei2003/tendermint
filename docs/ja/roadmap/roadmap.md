# テンダーミントロードマップ

*最終更新日:2021年10月8日金曜日*

このドキュメントは、より広いTendermintコミュニティに、Tendermint Coreの開発計画と優先順位、および機能の提供を予定している時期を理解させることを目的としています.これは、アプリケーション開発者、ノードオペレーター、インテグレーター、エンジニアリングおよび研究チームを含む、Tendermintのすべてのユーザーに広く通知することを目的としています.

提案された作業をこのロードマップの一部にしたい場合は、仕様の[issue](https://github.com/tendermint/spec/issues/new/choose)を開いてください.バグレポートやその他の実装の問題は、[コアリポジトリ](https://github.com/tendermint/tendermint)で発生する必要があります.

ロードマップは、タイムラインと成果物へのコミットメントではなく、計画と優先順位の高レベルのガイドと見なす必要があります.ロードマップの前にある機能は、通常、後ろにある機能よりも具体的で詳細です.このドキュメントは、現在の状況を反映するために定期的に更新されます.

アップグレードは2つの部分に分かれています.リリースする機能を定義し、リリースの時間を大幅に決定する** Epic **と、規模が小さく優先度の低い機能である** Minor **です. 、隣接するバージョンで表示される場合があります.

## V0.35(2021年の第3四半期に完成)
### 優先メモリプール

トランザクションの前に、それらはメモリプールに到着した順序でブロックに追加されます. 「CheckTx」を介して優先度フィールドを追加すると、アプリケーションはどのトランザクションをブロックに入れるかをより適切に制御できます.これは、取引コストがある場合に重要です. [詳細](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-067-mempool-refactor.md)

### P2Pフレームワークのリファクタリング

Tendermint P2Pシステムは、パフォーマンスと信頼性を向上させるために大規模な再設計が行われています.この再設計の最初のフェーズは0.35に含まれています.この段階では、抽象化をクリーンアップして切り離し、ピアライフサイクル管理、ピアアドレス処理を改善し、プラグ可能な送信を可能にします.以前の実装プロトコルと互換性があるように実装されています. [詳細](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-062-p2p-architecture.md)

### 状態の同期の改善

状態同期の初期バージョンの後、いくつかの改善が行われました.証拠処理の追加に必要な[ReverseSync](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-068-reverse-sync.md)を含み、[P2P状態プロバイダー]( https://github.com/tendermint/tendermint/pull/6807)RPCエンドポイントの代替として、スループットを調整するための新しい構成パラメーター、およびいくつかのバグ修正.

### カスタムイベントインデックス+ PSQLインデクサー

新しい「EventSink」インターフェースが追加され、Tendermint独自のトランザクションインデクサーを置き換えることができます.また、豊富なSQLベースのインデックスクエリを可能にするPostgreSQLインデクサー実装を追加しました. [詳細](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-065-custom-event-indexing.md)

### 小作品

-いくつかのGoパッケージが再編成され、パブリックAPIと実装の詳細の違いが明確になりました.
-ブロックインデクサーは、開始ブロックイベントと終了ブロックイベントにインデックスを付けます. [詳細](https://github.com/tendermint/tendermint/pull/6226)
-ブロック、ステータス、証拠、およびライトストレージキーは、辞書の順序を維持するように再設計されました.この変更には、データベースの移行が必要です. [詳細](https://github.com/tendermint/tendermint/pull/5771)
-テンダーミントモードの紹介.この変更の一部には、PEXリアクターのみを実行する別のシードノードを実行する可能性が含まれます. [詳細](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-052-tendermint-mode.md)

## V0.36(2022年の第1四半期に予定)

### ABCI ++

アプリケーションとコンセンサスの間の既存のインターフェイスを完全に改革して、アプリケーションがブロック構造をより適切に制御できるようにします. ABCI ++は、トランザクションがブロックに入る前にトランザクションを変更し、投票する前にブロックを検証し、投票に署名情報を挿入し、合意後にブロックをよりコンパクトに配信できるようにする新しいフックを追加します(同時実行を可能にします). [詳細](https://github.com/tendermint/spec/blob/master/rfc/004-abci%2B%2B.md)

###提案者のタイムスタンプに基づく

提案者に基づくタイムスタンプは、[BFT時間](https://docs.tendermint.com/master/spec/consensus/bft-time.html)の代替手段であり、提案者がタイムスタンプを選択し、バリデーターは次の場合、ブロックの投票タイムスタンプは*タイムリー*と見なされます.これにより、正確なローカルクロックへの依存度が高まりますが、その代わりに、ブロック時間の信頼性が高まり、障害に対する耐性が高まります.これには、ライトクライアント、IBCリピーター、CosmosHubインフレーション、および署名の集約を可能にする重要なユースケースがあります. [詳細](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-071-proposer-based-timestamps.md)

### ソフトアップグレード

ノードオペレーターとアプリケーション開発者が新しいバージョンのTendermintにすばやく安全にアップグレードできるように、一連のツールとモデルを開発しています. [詳細](https://github.com/tendermint/spec/pull/222)

### 小作品

-「レガシー」P2Pフレームワークを削除し、P2Pパッケージをクリーンアップします. [詳細](https://github.com/tendermint/tendermint/issues/5670)
-ローカルABCIクライアントからグローバルミューテックスを削除して、アプリケーション制御の同時実行を有効にします. [詳細](https://github.com/tendermint/tendermint/issues/7073)
-ライトクライアントのP2Pサポートを有効にする
-サービスノードのオーケストレーション+ノードの初期化と構成可能性
-複数のデータ構造の冗長性を削除します.ブロック同期v2リアクター、RPCレイヤーのgRPC、ソケットベースのリモート署名者などの未使用のコンポーネントを削除します.
-より多くのインジケーターを導入することにより、ノードの可視性を向上させます

## V0.37(2022年の第3四半期に予定)

### 完全なP2Pリファクタリング

P2Pシステムの最終段階を完了します.進行中の調査と計画では、[QUIC](https://en.wikipedia.org/wiki/QUIC)などの「MConn」送信方法の代わりに[libp2p](https://libp2p.io/)を使用するかどうかを決定しています. )および[Noise](https://noiseprotocol.org/)などのハンドシェイク/認証プロトコル.より高度なゴシップテクニックを研究します.

### ストレージエンジンを簡素化する

Tendermintには現在、複数のデータベースバックエンドをサポートできるようにする抽象化があります.この一般性により、メンテナンスオーバーヘッドが発生し、Tendermintが使用できるアプリケーション固有の最適化(ACID保証など)が妨げられる可能性があります. 1つのデータベースに集中し、Tendermintストレージエンジンを簡素化する予定です. [詳細](https://github.com/tendermint/tendermint/pull/6897)

### プロセス間通信を評価する

Tendermintノードには現在、他のプロセス(ABCI、リモート署名者、P2P、JSONRPC、WebSocket、イベントなど)との複数の通信フィールドがあります.これらの多くには複数の実装があり、そのうちの1つで十分です. IPCを統合してクリーンアップします. [詳細](https://github.com/tendermint/tendermint/blob/master/docs/rfc/rfc-002-ipc-ecosystem.md)

### ヒント

-健忘症の攻撃処理. [詳細](https://github.com/tendermint/tendermint/issues/5270)
-コンセンサスWALを削除/更新します. [詳細](https://github.com/tendermint/tendermint/issues/6397)
-署名の集約. [詳細](https://github.com/tendermint/tendermint/issues/1319)
-gogoprotoの依存関係を削除します. [詳細](https://github.com/tendermint/tendermint/issues/5446)

## V1.0(2022年の第4四半期に予定)

V0.37と同じ機能セットを備えていますが、テスト、プロトコルの正確性、および製品の安定性を確保するための微調整に重点を置いています.これらの取り組みには、[コンセンサステストフレームワーク](https://github.com/tendermint/tendermint/issues/5920)の拡張、カナリア/長寿命テストネットの使用、およびより大規模な統合テストが含まれる場合があります.

## リリース1.0の作業

-ブロックの伝播を改善するために、イレイジャーコーディングやコンパクトブロックを使用します. [詳細](https://github.com/tendermint/spec/issues/347)
-リファクタリングされたコンセンサスエンジン
-双方向ABCI
-ランダムリーダー選出
-ZKプルーフ/その他の暗号化プリミティブ
-マルチチェーンテンダーミント
