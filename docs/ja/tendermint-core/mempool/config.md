# 構成

ここでは、メモリプールを取り巻く構成オプションについて説明します。
このドキュメントの目的上、これらは次のように説明されています。
tomlファイルにありますが、それらのいくつかは次のように使用することもできます
環境変数。

構成:

```toml
[mempool]

recheck = true
broadcast = true
wal-dir = ""

# Maximum number of transactions in the mempool
size = 5000

# Limit the total size of all txs in the mempool.
# This only accounts for raw transactions (e.g. given 1MB transactions and
# max-txs-bytes=5MB, mempool will only accept 5 transactions).
max-txs-bytes = 1073741824

# Size of the cache (used to filter transactions we saw earlier) in transactions
cache-size = 10000

# Do not remove invalid transactions from the cache (default: false)
# Set to true if it's not possible for any invalid transaction to become valid
# again in the future.
keep-invalid-txs-in-cache = false

# Maximum size of a single transaction.
# NOTE: the max size of a tx transmitted over the network is {max-tx-bytes}.
max-tx-bytes = 1048576

# Maximum size of a batch of transactions to send to a peer
# Including space needed by encoding (one varint per transaction).
# XXX: Unused due to https://github.com/tendermint/tendermint/issues/5796
max-batch-bytes = 0
```

<!-- Flag: `--mempool.recheck=false`

Environment: `TM_MEMPOOL_RECHECK=false` -->

## 再確認

再チェックして、メモリプールがすべてのハングを再チェックするかどうかを判断します
ブロックが送信された後のトランザクション。一度にブロック
送信されると、メモリプールはすべての有効なトランザクションを削除します
ブロックに正常に含まれています。

`recheck`がtrueの場合、CheckTxが再実行されます
新しいブロックステータスを持つ残りのすべてのトランザクション。

## ブロードキャスト

このノードが有効なトランザクションについてチャットしているかどうかを確認します
メモリプールに到達します。デフォルトはゴシップすべてです
checktxを介して。このオプションが無効になっている場合、トランザクションは無効になります
ゴシップですが、ローカルに保存され、次のものに追加されます
このノードが提案者にならないようにします。

## WalDir

これは、メモリプールの先行書き込みディレクトリを定義します
ログ。これらのファイルを使用して、ブロードキャストされていないファイルをリロードできます
ノードがクラッシュしたときのトランザクション。

着信ディレクトリが絶対パスの場合、walファイルは次のようになります。
そこで作成します。ディレクトリが相対パスの場合、パスは
テンダーミントプロセスのホームディレクトリに添付します
walディレクトリの絶対パスを生成します
(デフォルトの `$ HOME/ .tendermint`または` TM_HOME`または `--home`によって設定されます)

## サイズ

サイズは、メモリプールに保存されるトランザクションの合計量を定義します。デフォルト値は「5_000」ですが、任意の数に調整できます。サイズが大きいほど、ノードへの負担が大きくなります。

## 最大トランザクションバイト

トランザクションバイトの最大数は、メモリプール内のすべてのトランザクションの合計サイズを定義します。デフォルト値は1GBです。

## キャッシュサイズ

キャッシュサイズは、私たちが見たキャッシュトランザクションのサイズを決定します。キャッシュは、トランザクションが受信されるたびに `checktx`が実行されないようにするために存在します。

## 無効なトランザクションをキャッシュに保持する

無効なトランザクションをキャッシュに保持して、キャッシュ内の無効なトランザクションを削除する必要があるかどうかを判断します。ここでの無効なトランザクションは、トランザクションがブロックに含まれていない別のtxに依存している可能性があることを意味している可能性があります。

## 最大トランザクションバイト

最大トランザクションバイト数は、ノードで使用できるトランザクションの最大サイズを定義します。ノードでより小さなトランザクションのみを追跡する場合は、このフィールドを変更する必要があります。デフォルトは1MBです。

## 最大バッチバイト

最大バッチバイト数は、ノードがピアに送信するバイト数を定義します。デフォルト値は0です。

>注:https://github.com/tendermint/tendermint/issues/5796のため使用されていません
