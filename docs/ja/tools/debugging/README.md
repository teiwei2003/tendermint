# デバッグ

## Tendermintデバッグキル

Tendermintには、リアルタイムで強制終了できる `debug`サブコマンドが付属しています.
Tendermintは、圧縮されたアーカイブに有用な情報を収集しながら処理します.
情報には、使用された構成、コンセンサスステータス、ネットワークが含まれます
ステータス、ノードステータス、WAL、さらにはプロセススタックトレース
終了する前に. これらのファイルは、障害をデバッグするときにチェックするために使用できます
テンダーミントプロセス.

```bash
tendermint debug kill <pid> </path/to/out.zip> --home=</path/to/app.d>
```


デバッグ情報を圧縮アーカイブに書き込みます. アーカイブには
以下:

```sh
├── config.toml
├── consensus_state.json
├── net_info.json
├── stacktrace.out
├── status.json
└── wal
```

舞台裏では、 `/status`、`/net_info`からの `debug kill`、
`/dump_consensus_state` HTTPエンドポイント、および` -6`を使用してプロセスを終了します.
ゴールーチンダンプをキャプチャします.

## Tendermintデバッグダンプ

さらに、 `debug dump`サブコマンドを使用すると、デバッグデータをにダンプできます.
定期的にファイルを圧縮します. これらのファイルにはゴルーチンが含まれています
コンセンサスステータス、ネットワーク情報、およびノー​​ドに加えて、ヒープ構成ファイルもあります
ステータス、さらにはWAL.

```bash
tendermint debug dump </path/to/out> --home=</path/to/app.d>
```

ただし、ノードと
デバッグデータをにダンプします
指定されたターゲットディレクトリ. 各アーカイブには以下が含まれます.

```sh
├── consensus_state.json
├── goroutine.out
├── heap.out
├── net_info.json
├── status.json
└── wal
```

注:goroutine.outとheap.outは、構成ファイルのアドレスが
提供して実行します. このコマンドはブロックされており、エラーをログに記録します.

## テンダーミントチェック

Tendermintには、Tendermintの状態ストレージとブロックを照会するための `inspect`コマンドが含まれています
TendermintRPCを介して保存されます.

Tendermintコンセンサスエンジンが不整合な状態を検出すると、クラッシュします
テンダーミントプロセス全体.
この一貫性のない状態では、Tendermintコンセンサスエンジンを実行しているノードは起動しません.
`inspect`コマンドは、ブロックストレージをクエリするためにTendermintRPCエンドポイントのサブセットのみを実行します
そしてステータスストア.
`inspect`を使用すると、オペレーターはステージの読み取り専用ビューを照会できます.
`inspect`はコンセンサスエンジンをまったく実行しないため、デバッグに使用できます
一貫性のない状態が原因でクラッシュしたプロセス.


「チェック」プロセスを開始するには、
```bash
tendermint inspect
```

### RPCエンドポイント
使用可能なRPCエンドポイントのリストは、RPCポートに要求を行うことで見つけることができます.
`127.0.0.1:26657`で実行されている` inspect`プロセスの場合、ブラウザを次の場所に移動します
`http://127.0.0.1:26657/`を使用して、有効なRPCエンドポイントのリストを取得します.

Tendermint RPCエンドポイントの詳細については、[rpcドキュメント](https://docs.tendermint.com/master/rpc)を参照してください.
