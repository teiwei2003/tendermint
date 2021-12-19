# ADR 058:イベントハッシュ

## 変更ログ

-2020-07-17:初期バージョン
-2020-07-27:IsmailとEthanのコメントを修正しました
-2020-07-27:辞退

## 環境

[PR#4845](https://github.com/tendermint/tendermint/pull/4845)の前、
`Header#LastResultsHash`は、` DeliverTx`から構築されたマークルツリーのルートです。
結果。 「情報」と「ログ」のため、「コード」フィールドと「データ」フィールドのみが含まれます
フィールドは未定義です。

ある時点で、 `ResponseBeginBlock`、` ResponseEndBlock`、にイベントを追加しました。
また、 `ResponseDeliverTx`は、アプリケーションに追加のメソッドを提供します
ブロック/トランザクション情報。

それ以来、多くのアプリケーションがそれらを使い始めたようです。

ただし、[PR#4845](https://github.com/tendermint/tendermint/pull/4845)の前
特定のイベントが結果の一部であることを証明する方法はありません
(_アプリケーション開発者がそれらを状態ツリーに含めない限り_)。

したがって、[PR#4845](https://github.com/tendermint/tendermint/pull/4845)は
開ける。その中で、ハッシュには `GasWanted`と` GasUsed`が含まれています
`DeliverTx`の結果。さらに、 `BeginBlock`、` EndBlock`、および `DeliverTx`からのイベント
結果は、以下に示すように `LastResultsHash`にハッシュされます。

-`BeginBlock`と `EndBlock`に多くのイベントを含めたくないので、
  これらはProtobufによってエンコードされ、Merkleツリーに葉として含まれます。
-したがって、 `LastResultsHash`は、3枚の葉を持つマークルツリーのルートハッシュです。
  プロトタイプでコード化された `ResponseBeginBlock#Events`、Merkelツリー構築のルートハッシュ
  `ResponseDeliverTx`からの応答(ログ、情報、コードスペースのフィールドは
  無視)、プロトタイプは `ResponseEndBlock#Events`をコーディングしました。
-イベントのシーケンスは変更されません-ABCIアプリケーションから受信したものと同じです。

[仕様PR](https://github.com/tendermint/spec/pull/97/files)

もちろん良いことは証明できますが、新しいイベントを紹介します
または、「LastResultsHash」が破棄されるため、このクラスの削除が困難になります。それ
イベントを追加、削除、または更新するたびに、
ハードフォーク。これは、進化するアプリケーションにとって間違いなく悪いことです
安定した一連のイベントはありません。

## 決定

妥協案として、提案は増加することです
`Block#LastResultsEvents`コンセンサスパラメータはすべてのイベントのリストです
ヘッダーでハッシュされます。
```
@ proto/tendermint/abci/types.proto:295 @ message BlockParams {
  int64 max_bytes = 1;
  // Note: must be greater or equal to -1
  int64 max_gas = 2;
  // List of events, which will be hashed into the LastResultsHash
  repeated string last_results_events = 3;
}
```

最初はリストは空です。 ABCIアプリケーションは `InitChain`を介してそれを変更することができます
または `EndBlock`。

例:

```go
func (app *MyApp) DeliverTx(req types.RequestDeliverTx) types.ResponseDeliverTx {
    //...
    events := []abci.Event{
        {
            Type: "transfer",
            Attributes: []abci.EventAttribute{
                {Key: []byte("sender"), Value: []byte("Bob"), Index: true},
            },
        },
    }
    return types.ResponseDeliverTx{Code: code.CodeTypeOK, Events: events}
}
```

「送信」イベントをハッシュするには、 `LastResultsEvents`に
文字列「転送」。

## ステータス

ごみ

**より多くの安定性/動機/ユースケース/要件があるまで、決定は
この完全にアプリケーション側を宣伝します。イベントが必要なアプリケーションのみを宣伝します。
それらをアプリケーション側のMerkelツリーに挿入することが証明できます。もちろん
これにより、アプリケーションの状態にさらに圧力がかかり、インシデントが証明されます
アプリケーションに固有ですが、ユースケースの認識を高めるのに役立つ場合があります
そして、テンダーミントが最終的にそれをどのように行うべきか。 ****

## 結果

### ポジティブ

1.ネットワークは、新しいイベントが追加されたときにこのリストを更新するためのパラメーター変更の提案を実装できます
2.ネットワークがハードフォークを回避できるようにします
3.イベントは、何も中断することなく、アプリケーションに自由に追加できます。

### ネガティブ

1.別のコンセンサスパラメータ
2.テンダーミント状態でより多くのものを追跡する

## 参照する

-[ADR 021](./ adr-021-abci-events.md)
-[インデックス作成トランザクション](../ app-dev / indexing-transactions.md)

## 付録A.代替案

別の提案は、次のように、 `Hashbool`フラグを` Event`に追加することです。
`Index bool`EventAttributeフィールド。 trueの場合、Tendermintはそれを次のようにハッシュします
`LastResultsEvents`。欠点は、ロジックが暗黙的であり、依存していることです。
これは主に、実行するアプリケーションコードを決定するノードのオペレーターに依存します。この
上記の提案により、(論理的に)明確で簡単にアップグレードできます
ガバナンス。
