# 同期をブロックする
*以前はQuickSyncと呼ばれていました*

プルーフオブワークブロックチェーンでは、チェーンとの同期は同じです
コンセンサスとの同期を維持するプロセス:ブロックのダウンロード、および
総ワークロードが最も多いものを探します。公平性の証明では、
コンセンサスプロセスは複数のラウンドを伴うため、より複雑です
どのブロックが必要かを決定するためのノード間の通信
次にコミットします。このプロセスを使用して、ブロックチェーンと同期します
ゼロから始めるには長い時間がかかる場合があります。直接ダウンロードははるかに高速になります
リアルタイムで実行するのではなく、バリデーターのMerkelツリーをブロックして確認します
コンセンサスゴシップ合意。

## ブロック同期を使用する

より高速な同期をサポートするために、Tendermintは「blocksync」モードを提供します。
これはデフォルトで有効になっており、 `config.toml`または
`--blocksync.enable = false`。

このモードでは、Tendermintデーモンは何百回も同期します
リアルタイムのコンセンサスプロセスを使用する代わりに。追いついたら、
デーモンはブロック同期を終了し、通常のコンセンサスモードに入ります。
一定期間実行した後、ノードが「追いついた」と見なされる場合
少なくとも1つのピアがあり、その高さは少なくとも最大と同じくらい高い
報告されたピアの高さ。 [IsCaughtUpを参照してください
メソッド](https://github.com/tendermint/tendermint/blob/b467515719e686e4678e6da4e102f32a491b85a0/blockchain/pool.go#L128)。

注:BlockSyncには複数のバージョンがあります。他のバージョンはサポートされなくなったため、v0を使用してください。
  別のバージョンを使用する場合は、 `config.toml`でバージョンを変更することで使用できます。

```toml
#######################################################
###       Block Sync Configuration Connections       ###
#######################################################
[blocksync]

# If this node is many blocks behind the tip of the chain, BlockSync
# allows them to catchup quickly by downloading blocks in parallel
# and verifying their commits
enable = true

# Block Sync version to use:
#   1) "v0" (default) - the standard Block Sync implementation
#   2) "v2" - DEPRECATED, please use v0
version = "v0"
```

十分に遅れている場合は、ブロック同期に戻る必要がありますが、
これは[未解決の問題](https://github.com/tendermint/tendermint/issues/129)です。

##ブロック同期イベント
テンダーミントブロックチェーンコアが起動すると、 `block-sync`に切り替わる場合があります
モデルは、現在のネットワークの最高の高さまで状態に追いつきます。 コアが発行されます
現在の状態と同期の高さを開示するためのクイック同期イベント。 捕まえたら
ネットワークの最適な高さ、それは状態同期メカニズムに切り替わり、次に発行します
別のイベントは、高速同期の「完了」ステータスとステータス「高さ」を開示するために使用されます。

ユーザーは `EventQueryBlockSyncStatus`にサブスクライブすることでイベントをクエリできます
詳細については、[types](https://pkg.go.dev/github.com/tendermint/tendermint/types?utm_source=godoc#pkg-constants)を参照してください。

## 埋め込む

実装の詳細については、[reactor doc](./reactor.md)および[implementation doc](./implementation.md)を参照してください。
