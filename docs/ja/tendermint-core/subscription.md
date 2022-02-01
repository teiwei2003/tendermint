# Websocketを介してイベントをサブスクライブする

Tendermintはさまざまなイベントを発行します.次の方法でサブスクライブできます
[Websocket](https://en.wikipedia.org/wiki/WebSocket). これは便利です
サードパーティのアプリケーション(分析用)またはステータスの確認に使用されます.

[イベントリスト](https://godoc.org/github.com/tendermint/tendermint/types#pkg-constants)

CLIからWebSocketを介してノードに接続するには、次のようなものを使用できます.
[wscat](https://github.com/websockets/wscat)そして実行:

```sh
wscat ws://127.0.0.1:26657/websocket
```

`subscribe` RPCを呼び出すことで、上記のイベントのいずれかをサブスクライブできます.
Websocketと効果的なクエリによるメソッド.

```json
{
    "jsonrpc": "2.0",
    "method": "subscribe",
    "id": 0,
    "params": {
        "query": "tm.event='NewBlock'"
    }
}
```

[APIドキュメント](https://docs.tendermint.com/master/rpc/)を表示する
クエリ構文およびその他のオプションに関する詳細情報.

DeliverTxにタグを含めていれば、タグを使用することもできます.
応答、トランザクション結果を照会します. [インデックス]を参照してください
詳細については、トランザクション](../app-dev/indexing-transactions.md)を参照してください.

## バリデーターセットの更新

バリデーターセットが変更されると、ValidatorSetUpdatesイベントが通知されます. この
このイベントには、公開鍵と電源のペアのリストが含まれています. リストは同じです
テンダーミントはABCIアプリケーションから受信されます([EndBlockを参照]
パート)(https://github.com/tendermint/spec/blob/master/spec/abci/abci.md#endblock)
ABCI仕様).

返事:

```json
{
    "jsonrpc": "2.0",
    "id": 0,
    "result": {
        "query": "tm.event='ValidatorSetUpdates'",
        "data": {
            "type": "tendermint/event/ValidatorSetUpdates",
            "value": {
              "validator_updates": [
                {
                  "address": "09EAD022FD25DE3A02E64B0FE9610B1417183EE4",
                  "pub_key": {
                    "type": "tendermint/PubKeyEd25519",
                    "value": "ww0z4WaZ0Xg+YI10w43wTWbBmM3dpVza4mmSQYsd0ck="
                  },
                  "voting_power": "10",
                  "proposer_priority": "0"
                }
              ]
            }
        }
    }
}
```
