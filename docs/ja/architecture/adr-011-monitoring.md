# ADR 011:モニタリング

## 変更ログ

2018年8月6日:最初のドラフト
2018年11月6日:@xlaコメント後の再編成
2018年6月13日:ラベルの使用を明確にする

## 環境

Tendermintの可視性を高めるために、レポートを作成する必要があります
インジケーター.将来、トランザクションとRPCクエリの痕跡が残る可能性があります.見て
https://github.com/tendermint/tendermint/issues/986.

いくつかの解決策が検討されました.

1. [プロメテウス](https://prometheus.io)
   a)Prometheus API
   b)[go-kitメトリクスパッケージ](https://github.com/go-kit/kit/tree/master/metrics)インターフェースとしてPrometheusを追加
   c)[Telegraf](https://github.com/influxdata/telegraf)
   d)新しいサービス.pubsubによって送信されたイベントをリッスンし、メトリックをレポートします
2. [OpenCensus](https://opencensus.io/introduction/)

### 1.プロメテウス

Prometheusは最も人気のある監視製品のようです.それは持っています
Goクライアントライブラリ、強力なクエリ、アラート.

** a)Prometheus API **

TendermintでPrometheusを使用することを約束できますが、Tendermintユーザーは
彼らがより適切であると考える監視ツールを自由に選択する必要があります
彼らのニーズ(既存のニーズがない場合).だから私たちは試してみるべきです
人々がPrometheusやその他を使用できるように十分に抽象的なインターフェース
同様のツール.

** b)インターフェースとしてのGo-kitインジケーターパッケージ**

メトリックパッケージは、サービス用の統一されたインターフェイスのセットを提供します
人気のあるインジケーターパッケージのアダプターを検出して提供します.

https://godoc.org/github.com/go-kit/kit/metrics#pkg-subdirectories

Prometheus APIと比較すると、カスタマイズ性と制御性は失われていますが、
抽出することを考えると、上記のリストから任意の楽器を選択する自由
インジケーターは別の関数で作成されます(node/node.goの「プロバイダー」を参照).

** c)電報**

すでに説明したオプションとは異なり、telegrafはTendermintを変更する必要はありません
ソースコード.入力プラグインと呼ばれるものを作成し、ポーリングします
Tendermint RPCは毎秒実行され、インジケーター自体を計算します.

良さそうに聞こえますが、報告したい指標の一部が合格しませんでした
RPCまたはpubsubであるため、外部からアクセスすることはできません.

** d)サービス、pubsubを聞く**

上記と同じ問題.

### 2.国勢調査を開きます

opencensusは、測定と追跡を提供します.
将来.そのAPIはgo-kitやPrometheusとは異なって見えますが、よく似ています
必要なものすべてをカバーします.

残念ながら、OpenCensusgoクライアントは何も定義していません
インターフェースなので、インジケーターを抽象化したい場合は、
インターフェイスは自分で作成する必要があります.

### インジケーターのリスト

|     | Name                                 | Type   | Description                                                                   |
| --- | ------------------------------------ | ------ | ----------------------------------------------------------------------------- |
| A   | consensus_height                     | Gauge  |                                                                               |
| A   | consensus_validators                 | Gauge  | Number of validators who signed                                               |
| A   | consensus_validators_power           | Gauge  | Total voting power of all validators                                          |
| A   | consensus_missing_validators         | Gauge  | Number of validators who did not sign                                         |
| A   | consensus_missing_validators_power   | Gauge  | Total voting power of the missing validators                                  |
| A   | consensus_byzantine_validators       | Gauge  | Number of validators who tried to double sign                                 |
| A   | consensus_byzantine_validators_power | Gauge  | Total voting power of the byzantine validators                                |
| A   | consensus_block_interval             | Timing | Time between this and last block (Block.Header.Time)                          |
|     | consensus_block_time                 | Timing | Time to create a block (from creating a proposal to commit)                   |
|     | consensus_time_between_blocks        | Timing | Time between committing last block and (receiving proposal creating proposal) |
| A   | consensus_rounds                     | Gauge  | Number of rounds                                                              |
|     | consensus_prevotes                   | Gauge  |                                                                               |
|     | consensus_precommits                 | Gauge  |                                                                               |
|     | consensus_prevotes_total_power       | Gauge  |                                                                               |
|     | consensus_precommits_total_power     | Gauge  |                                                                               |
| A   | consensus_num_txs                    | Gauge  |                                                                               |
| A   | mempool_size                         | Gauge  |                                                                               |
| A   | consensus_total_txs                  | Gauge  |                                                                               |
| A   | consensus_block_size                 | Gauge  | In bytes                                                                      |
| A   | p2p_peers                            | Gauge  | Number of peers node's connected to                                           |

`A`-最初に実装されます.

**提案された解決策**

## ステータス

実装

## 結果

### ポジティブ

可視性の向上、さまざまな監視バックエンドのサポート

### ネガティブ

監査用の別のライブラリは、インジケーターレポートコードをビジネスドメインと混同します.

### ニュートラル

-
