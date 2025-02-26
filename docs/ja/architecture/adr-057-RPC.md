# ADR 057:RPC

## 変更ログ

-19-05-2020:作成

## 環境

現在、TendermintのRPCレイヤーは、JSON-RPCプロトコルのバリアントを使用しています.このADRは、JSON-RPCの可能な代替案と長所と短所のリストとして使用することを目的としています.

現在、gRPCとJSON-RPCの2つのオプションが検討されています.

### JSON-RPC

JSON-RPCは、JSONに基づくRPCプロトコルです. Tendermintは、[JSON-RPC 2.0仕様](https://www.jsonrpc.org/specification)と互換性のない独自のJSON-RPCバリアントを実装しています.

**アドバンテージ:**

-使いやすく、実装も簡単(デフォルト)
-ユーザーとインテグレーターに知られ、理解されている
-Webインフラストラクチャ(プロキシ、APIゲートウェイ、サービスメッシュ、キャッシュなど)とうまく統合できます
-人間が読めるエンコーディング(デフォルト)

**欠点:**

-モードサポートなし
-RPCクライアントは手書きである必要があります
-ストリーミングはプロトコルに組み込まれていません
-不特定のタイプ(例:番号とタイムスタンプ)
-Tendermintには独自の実装があります(非標準、メンテナンスオーバーヘッド)
  -これに関連する高いメンテナンスコスト
-Stdlib`jsonrpc`パッケージはJSON-RPC1.0のみをサポートし、JSON-RPC2.0のメインパッケージはサポートしません
-ドキュメント/仕様に関するツール(Swaggerなど)の方が優れている可能性があります
-JSONデータが大きい(HTTP圧縮によるオフセット)
-シリアル化は非常に遅い([〜100％マーシャル、〜400％アンマーシャル](https://github.com/alectomas/go_serialization_benchmarks));絶対値は重要ではありません.
-仕様は2013年に最後に更新されましたが、これはSwagger/OpenAPIよりもはるかに遅れています.

### gRPC + gRPCゲートウェイ(REST + Swagger)

gRPCは高性能RPCフレームワークです.それは多くのユーザーによって実際の戦闘でテストされており、無数の大企業によって非常に依存され、維持されています.

**アドバンテージ:**

-ユーザー、合理化されたクライアント、その他のプロトコルのための効率的なデータ検索
-サポートされている言語(Go、Dart、JS、TS、rust、Elixir、Haskellなど)での簡単な実装
-よりリッチな型システム(プロトコルバッファ)を使用した定義モード
-共通のパターンとタイプは、すべてのプロトコルとデータストレージ(RPC、ABCI、ブロックなど)で使用できます.
-下位互換性と下位互換性の規則を確立する
-双方向ストリーミング
-サーバーとクライアントは複数の言語で自動的に生成されます(例:Tendermint-rs)
-RESTAPIのSwaggerドキュメントを自動的に生成します
-プロトコルレベルで実施される下位互換性と上位互換性の保証.
-さまざまなコーデック(JSON、CBOR、...)で利用可能

**欠点:**

-クロスランゲージモード、コード生成、カスタムプロトコルを含む複雑なシステム
-型システムは必ずしも母国語の型システムに明確に対応しているわけではありません.統合のジレンマ
-多くの一般的なタイプにはProtobufプラグインが必要です(タイムスタンプや期間など)
-生成されたコードは慣用的ではなく、使いにくい場合があります
-移行は混乱を招き、骨の折れる作業になります

## 決定

>このセクションでは、実装の詳細を含む、提案されたソリューションのすべての詳細について説明します.
>また、その一部として変更する必要があるかもしれない影響/必要性についても説明する必要があります.
>提案された変更が大きい場合は、レビューを容易にするためにどのように変更が行われたかについても説明してください.
>(たとえば、別々のPR間で行われることの最良の分割)

## ステータス

>合意がない場合は、決定を「提案」するか、合意に達したら「受け入れる」ことができます.後続のADRが決定を変更または撤回した場合は、それを「非推奨」または「置き換え」としてマークし、その置き換えについて言及することができます.

{非推奨|推奨|承認済み}

## 結果

>このセクションでは、決定を適用した場合の結果について説明します. 「ポジティブ」な結果だけでなく、すべての結果をここに要約する必要があります.

### ポジティブ

### ネガティブ

### ニュートラル

## 参照する

>関連するPRコメント、この問題の原因となった問題、または特定の設計を選択した理由に関する参考記事はありますか？もしそうなら、ここにそれらをリンクしてください！

-{参照リンク}
