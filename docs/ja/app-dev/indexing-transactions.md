# インデックストランザクション

Tendermintを使用すると、トランザクションとブロックにインデックスを付けてから、クエリまたは
結果を購読します。 トランザクションは「TxResult.Events」と
ブロックには、「Response(Begin | End)Block.Events」というインデックスが付けられます。 ただし、トランザクション
また、トランザクションハッシュとマッピングを含む主キーによってインデックスが作成されます
対応する「TxResult」を保存します。 主キーでインデックス付けされたブロック
これにはブロックの高さが含まれ、ブロックの高さにマップされて格納されます。
ブロック自体が保存されることはありません。

各イベントには、キーと値のペアであるタイプと属性のリストが含まれています
メソッドの実行中に何が起こったかを表します。 もっと
「イベント」の詳細については、を参照してください。
[ABCI](https://github.com/tendermint/spec/blob/master/spec/abci/abci.md#events)
書類。

`Event`には複合キーが関連付けられています。 「複合キー」は
タイプとキーはドットで区切られています。

例えば:

```json
"jack": [
  "account.number": 100
]
```

`jack.account.number`のキーの組み合わせと同じになります。

デフォルトでは、Tendermintはすべてのトランザクションをそれぞれのハッシュでインデックス付けします
そして、ブロックの高さと高さ。

## 構成

オペレーターは、 `[tx_index]`セクションを介してインデックスを構成できます。 `インデクサ`
このフィールドは、サポートされている一連のインデクサーを使用します。 `null`が含まれている場合、インデックスは
提供された他の値に関係なく、それは閉じられます。

```toml
[tx-index]

# The backend database list to back the indexer.
# If list contains null, meaning no indexer service will be used.
#
# The application will set which txs to index. In some cases a node operator will be able
# to decide which txs to index based on configuration set in the application.
#
# Options:
#   1) "null"
#   2) "kv" (default) - the simplest possible indexer, backed by key-value storage (defaults to levelDB; see DBBackend).
#     - When "kv" is chosen "tx.height" and "tx.hash" will always be indexed.
#   3) "psql" - the indexer services backed by PostgreSQL.
# indexer = []
```

### サポートされているインデクサー

#### KV

`kv`インデクサータイプは、サポートされている主要な組み込みKey-Valueストアです
基盤となるTendermintデータベース。 `kv`インデクサータイプを使用すると、クエリを実行できます
TendermintのRPCブロックおよびトランザクションイベントを直接ターゲットにするために使用されます。 でもそれは
クエリ構文が制限されているため、このインデクサータイプは非推奨または削除される可能性があります
完全に将来。

#### PostgreSQL

`psql`インデクサータイプにより、オペレーターはブロックイベントとトランザクションイベントを有効にできます
イベントを許可する外部PostgreSQLインスタンスにプロキシしてインデックスを作成します
リレーショナルモデルに格納されます。 イベントはRDBMSに保存されるため、オペレーターは
SQLを使用して、一連のリッチで複雑なクエリを実行できます
`kv`インデクサータイプのサポート。 演算子はSQLを直接使用できるため、
TendermintのRPCが `psql`インデクサータイプの検索を有効にできませんでした-任意
このようなクエリは失敗します。

SQLスキーマは `state/indexer/sink/psql/schema.sql`と演算子に格納されていることに注意してください
Tendermintを起動して有効にする前に、関係を明示的に作成する必要があります
`psql`インデクサータイプ。

例:

```shell
$ psql ... -f state/indexer/sink/psql/schema.sql
```

## デフォルトのインデックス

Tendermintトランザクションおよびブロックイベントインデクサーは、いくつかの選択された保持イベントにインデックスを付けます
デフォルトでは。

### トレード

次のインデックスはデフォルトでインデックスが付けられます。

-`tx.height`
-`tx.hash`

### ピース

次のインデックスはデフォルトでインデックスが付けられます。

-`block.height`

## イベントを追加

アプリケーションは、インデックスを作成するイベントを自由に定義できます。 テンダーミント
インデックスを作成するイベントと無視するイベントを定義する関数を公開します。 存在
アプリケーションの `DeliverTx`メソッド、` Events`フィールドのペアを追加します
UTF-8でエンコードされた文字列(例: "transfer.sender": "Bob"、 "transfer.recipient":
「アリス」、「transfer.balance」:「100」)。

例:

```go
func (app *KVStoreApplication) DeliverTx(req types.RequestDeliverTx) types.Result {
    //...
    events := []abci.Event{
        {
            Type: "transfer",
            Attributes: []abci.EventAttribute{
                {Key: []byte("sender"), Value: []byte("Bob"), Index: true},
                {Key: []byte("recipient"), Value: []byte("Alice"), Index: true},
                {Key: []byte("balance"), Value: []byte("100"), Index: true},
                {Key: []byte("note"), Value: []byte("nothing"), Index: true},
            },
        },
    }
    return types.ResponseDeliverTx{Code: code.CodeTypeOK, Events: events}
}
```

インデクサーが `null`でない場合、トランザクションはインデックス付けされます。 すべてのイベントは
"{eventType}。{eventAttribute} = {eventValue}"の形式の複合キーインデックスを使用します、
たとえば、 `transfer.sender = bob`です。

## トランザクションイベントのクエリ

イベントを呼び出すことにより、一連のページングトランザクションをクエリできます
`/tx_search` RPCエンドポイント:

```bash
curl "localhost:26657/tx_search?query=\"message.sender='cosmos1...'\"&prove=true"
```

[APIドキュメント](https://docs.tendermint.com/master/rpc/#/Info/tx_search)を表示する
クエリ構文およびその他のオプションに関する詳細情報。

## サブスクリプショントランザクション

クライアントは、WebSocketを介して特定のタグを使用してトランザクションをサブスクライブできます
`/subscribe`RPCエンドポイントへのクエリ。

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "message.sender='cosmos1...'"
  }
}
```

查看 [API 文档](https://docs.tendermint.com/master/rpc/#subscribe) 了解更多信息
关于查询语法和其他选项。

## ブロックイベントのクエリ

イベントを呼び出すことにより、ページのグループのブロックをクエリできます
`/block_search` RPCエンドポイント:

```bash
curl "localhost:26657/block_search?query=\"block.height > 10 AND val_set.num_changed > 0\""
```

[APIドキュメント]を表示(https://docs.tendermint.com/master/rpc/#/Info/block_search)
クエリ構文およびその他のオプションに関する詳細情報。
