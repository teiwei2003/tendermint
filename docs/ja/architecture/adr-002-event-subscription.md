# ADR 2:イベントサブスクリプション

## 環境

ライトクライアント(または他のクライアント)では、ユーザーは**サブスクライブすることができます
`/ subscribe？event = X`トランザクションサブセット**を使用します(すべてではありません). にとって
たとえば、特定のトランザクションに関連するすべてのトランザクションをサブスクライブしたい
アカウント. フェッチについても同じことが言えます. ユーザーは
一部のフィルター**(すべてのブロックを取得する代わりに). たとえば、私は取得したい
過去2週間の特定のアカウントのすべてのトランザクション( `txのブロック時間> = '2017-06-05'`).

今では、テンダーミントの「すべてのトランザクション」を購読することさえできません.

目標は、これを行うための使いやすいAPIです.

！[送信フローチャート](img/tags1.png)

## 決定

ABCIアプリケーションはラベルを返します(_for
さて、後で別のフィールドを作成するかもしれません_). タグは、キーと値のペアのリストです.
protobufエンコーディング.

サンプルデータ:

```json
{
  "abci.account.name": "Igor",
  "abci.account.address": "0xdeadbeef",
  "tx.gas": 7
}
```

### トランザクションイベントをサブスクライブする

ユーザーがトランザクションのサブセットのみを受け取りたい場合、ABCIアプリは
「DeliverTx」の応答を持つタグのリストを返します. これらのタグは解析され、
現在のクエリ(サブスクライバー)と一致します. クエリがタグと一致する場合、
サブスクライバーはトランザクションイベントを取得します.

```
/subscribe?query="tm.event = Tx AND tx.hash = AB0023433CF0334223212243BDD AND abci.account.invoice.number = 22"
```

現在の `events`パッケージを置き換えるために、新しいパッケージを開発する必要があります. それ
顧客は、将来、さまざまな種類のイベントにサブスクライブできるようになります.

```
/subscribe?query="abci.account.invoice.number = 22"
/subscribe?query="abci.account.invoice.owner CONTAINS Igor"
```

### トランザクションを取得する

これは少し注意が必要です.a)多くのインデクサーをサポートしたいからです.
さまざまなAPIがありますb)タグで十分になる時期がわかりません
ほとんどのアプリケーションで(私は見ると思います).

```
/txs/search?query="tx.hash = AB0023433CF0334223212243BDD AND abci.account.owner CONTAINS Igor"
/txs/search?query="abci.account.owner = Igor"
```

For historic queries we will need a indexing storage (Postgres, SQLite, ...).

### 問題

-https://github.com/tendermint/tendermint/issues/376
-https://github.com/tendermint/tendermint/issues/287
-https://github.com/tendermint/tendermint/issues/525(関連)

## ステータス

実装

## 結果

### ポジティブ

-イベント通知と検索APIの同じ形式
-十分に強力なクエリ

### ネガティブ

-`match`関数のパフォーマンス(クエリ/サブスクライバーが多すぎます)
-データベースにtxが多すぎます

### ニュートラル
