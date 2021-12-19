# 本番環境で実行

本番環境で使用するためにソースからTendermintを構築している場合は、ブランチではなく、適切なGitタグを確認してください。

## データベース

デフォルトでは、Tendermintはインプロセスとして `syndtr/goleveldb`パッケージを使用します
Key-Valueデータベース。最大のパフォーマンスが必要な場合は、インストールするのが最適です
LevelDBの実際のC実装と、それを使用するためのTendermintのコンパイル
`ビルドTENDERMINT_BUILD_OPTIONS = cleveldb`を作成します。 [インストール
詳細については、[説明](../Introduction/install.md)を参照してください。

Tendermintは、いくつかの異なるデータベースを `$ TMROOT/data`に保存します。

-`blockstore.db`:ブロックチェーン全体を保持します-ストレージブロック、
  ブロック送信とブロックメタデータ。それぞれ高さでインデックスが付けられます。新規の同期に使用
  ピア。
-`evidence.db`:不正行為の検証済みの証拠をすべて保存します。
-`state.db`:現在のブロックチェーンの状態(つまり、高さ、バリデーター、
  コンセンサスパラメータ)。コンセンサスパラメータまたはバリデーターが変更された場合にのみ大きくなります。また
  ブロック処理中に中間結果を一時的に保存するために使用されます。
-`tx_index.db`:txハッシュとDeliverTx結果イベントによってtxs(およびその結果)にインデックスを付けます。

デフォルトでは、Tendermintは、DeliverTxではなく、ハッシュと高さに基づいてtxにのみインデックスを付けます。
結果イベント。 [インデックストランザクション](../app-dev/indexing-transactions.md)を参照して理解してください
詳細。

アプリケーションは、ブロックプルーニング戦略をノードオペレーターに開示できます。アプリケーションのドキュメントをお読みください
もっと詳しく知る。

アプリケーションは[statesync](state-sync.md)を使用して、ノードをすばやく起動できるようにします。

## 記録

デフォルトのログレベル( `log-level =" info "`)で十分です。
通常動作モード。これを読む
投稿](https://blog.cosmos.network/one-of-the-exciting-new-features-in-0-10-0-release-is-smart-log-level-flag-e2506b4ab756)
「ログレベル」構成変数の構成方法の詳細。いくつかの
モジュールは[ここ](logging.md#list-of-modules)にあります。もしも
Tendermintをデバッグするか、デバッグログを要求しようとしています
ロギングレベル、Tendermintを実行することで達成できます
`--log-level =" Debug "`。

### コンセンサスWAL

Tendermintは、コンセンサスに達するために先行書き込みログ(WAL)を使用します。 `consensus.wal`は、クラッシュからいつでも回復できるようにするために使用されます
コンセンサスステートマシン。すべてのコンセンサスメッセージ(タイムアウト、提案、ブロック部分、または投票)を書き込みます
単一のファイルに、それ自体からのメッセージを処理する前にディスクにフラッシュします
バリデーター。テンダーミントの検証者は、相反する投票に署名することは決してないと予想されるため、
WALは、必要がなくても常に決定論的にコンセンサスの最新状態に復元できることを保証します
ネットワークを使用するか、コンセンサスメッセージに再署名します。 WALの最大サイズは1GBであり、自動的にローテーションされることが合意されています。

`consensus.wal`が破損している場合は、[下記](#wal-corruption)を参照してください。
## DOSの露出と軽減

バリデーターは[SentryNodeに設定する必要があります
アーキテクチャ](./validators.md)
サービス拒否攻撃を防ぐため。

### ピアツーピア

Tendermintのピアツーピアシステムの中核は「MConnection」です。各
接続には、最大のパケットである `MaxPacketMsgPayloadSize`があります
サイズと制限付きの送信キューと受信キュー。一人で制限を課すことができます
各接続の送受信レート( `SendRate`、` RecvRate`)。

開いているP2P接続の数が非常に多くなり、オペレーティングシステムのオープンに影響します
ファイルの制限(TCP接続はUNIXベースのシステムではファイルとして扱われるため)。ノードは
8192など、かなり大きなオープンファイル制限がある場合は、 `ulimit -n8192`またはその他のデプロイメント固有のものを渡します。
機構。

### RPC

複数のエントリを返すエンドポイントは、デフォルトで30を返すように制限されています
要素(最大100)。 [RPCドキュメント](https://docs.tendermint.com/master/rpc/)を参照してください
詳細情報が必要です。

レート制限と認証は、保護に役立つもう1つの重要な側面です
DOS攻撃に抵抗します。バリデーターは、次のような外部ツールを使用する必要があります
[NGINX](https://www.nginx.com/blog/rate-limiting-nginx/)または
[traefik](https://docs.traefik.io/middlewares/ratelimit/)
同じ目的を達成するため。

## テンダーミントをデバッグする

Tendermintをデバッグする必要がある場合、最初にすべきことは
ログを確認してください。 [Logging](../ノード/logging.md)を参照してください。
特定のログステートメントの意味を説明します。

ログを閲覧した後も状況が不明な場合は、次は何ですか
`/status`RPCエンドポイントをクエリしてみてください。必要な情報を提供します。
ノードが同期されているかどうかに関係なく、ノードの高さなどはどのくらいですか。

```bash
curl http(s)://{ip}:{rpcPort}/status
```

`/dump_consensus_state`は、コンセンサスの詳細な概要を示します
ステータス(提案者、最新のバリデーター、ピアステータス)。 それから、あなたはできるはずです
たとえば、ネットワークがダウンしている理由を調べます。

```bash
curl http(s)://{ip}:{rpcPort}/dump_consensus_state
```

このエンドポイントの簡略化されたバージョンがあります-` /consensus_state`、これは
現在の高さで見られる投票のみ。

ログと上記のエンドポイントを調べた後でも、まだわからない場合
何が起こったのか、「tendermintdebugkill」サブコマンドの使用を検討してください。この
このコマンドは、利用可能なすべての情報を破棄し、プロセスを終了します。見て
[デバッグ](../tools /debugging /README.md)正確な形式を取得します。

生成されたアーカイブは、自分で確認することも、
[Github](https://github.com/tendermint/tendermint)。質問を開く前に
ただし、[存在しない
問題](https://github.com/tendermint/tendermint/issues)すでに。

## テンダーミントの監視

各Tendermintインスタンスには、応答する標準の `/health`RPCエンドポイントがあります
200(OK)すべてがOKの場合、500(または応答なし)-何かがある場合
正しくない。

その他の有用なエンドポイントには、前述の `/status`、` /net_info`および
`/Validator`。

Tendermintは、Prometheusメトリックをレポートおよび提供することもできます。見て
[メトリクス](./metrics.md)。

`tendermint debug dump`サブコマンドを使用して、有用なものを定期的にダンプできます
情報アーカイブ。詳細については、[デバッグ](../tools /debugging /README.md)を参照してください。
情報。

## アプリが停止するとどうなりますか

あなたは[プロセスにいるはずです
スーパーバイザー)(https://en.wikipedia.org/wiki/Process_supervision)(例:
systemdまたはrunit)。これにより、Tendermintが常に実行されていることが保証されます(ただし
考えられるエラー)。

元の質問に戻ります。アプリケーションが停止した場合は、
テンダーミントはパニックになります。プロセススーパーバイザーで再起動します
アプリケーション、Tendermintは正常に再接続できるはずです。この
再起動シーケンスは重要ではありません。

## 信号処理

SIGINTとSIGTERMをキャプチャし、クリーンアップを試みます。ほかの人のため
Goでのシグナルのデフォルトの動作を使用します:(シグナルのデフォルトの動作
Goで
プログラム](https://golang.org/pkg/os/signal/#hdr-Default_behavior_of_signals_in_Go_programs)。

## 腐敗

**注:** Tendermintデータディレクトリのバックアップがあることを確認してください。

### 考えられる理由

ほとんどの損傷はハードウェアの問題によって引き起こされることに注意してください。

-RAIDコントローラのバックアップバッテリが故障/摩耗していて、予期せず電源がオフになっている
-ライトバックキャッシュが有効になっていて、予期せず電源が切れたハードドライブ
-低価格のソリッドステートハードドライブ、不十分な電源オフ保護、予期しない電源オフ
-欠陥のあるメモリ
-CPUに欠陥があるか、過熱しています

その他の理由は次のとおりです。

-fsync = offで構成されたデータベースシステムとオペレーティングシステムのクラッシュまたは電源障害
-ファイルシステムは、書き込みバリアと書き込みバリアを無視するストレージレイヤーを使用するように構成されています。 LVMは特定の原因です。
-テンダーミントエラー
-オペレーティングシステムエラー
-管理者エラー(たとえば、Tendermintデータディレクトリの内容を直接変更する)

(出典:<https://wiki.postgresql.org/wiki/Corruption>)

### WALの破損

コンセンサスWALが最新の高さで破損していて、開始しようとしている場合
テンダーミント、パニックのためリプレイは失敗します。

データ破損からの回復は困難で時間がかかる場合があります。次の2つの方法を使用できます。

1. WALファイルを削除し、Tendermintを再起動します。他のピアとの同期を試みます。
2.WALファイルを手動で修復してみてください。

1)破損したWALファイルのバックアップを作成します。

    ```sh
    cp "$TMHOME/data/cs.wal/wal" >/tmp/corrupted_wal_backup
    ```

2) `。/scripts /wal2json`を使用して、人間が読める形式のバージョンを作成します。

    ```sh
    ./scripts/wal2json/wal2json "$TMHOME/data/cs.wal/wal" >/tmp/corrupted_wal
    ```

3)「CORRUPTEDMESSAGE」の行を検索します。
4)前のメッセージと破損したメッセージを表示する
     そして、ログを確認して、メッセージを再構築してみてください。 次の場合
     メッセージも破損としてマークされます(長さヘッダーの場合)
     破損しているか、一部の書き込みがWALに入力されていません〜切り捨てられました)、
     次に、破損した行から始まるすべての行を削除して再起動します
     肌の若返り。

    ```sh
    $EDITOR/tmp/corrupted_wal
    ```

5)編集後、次のコマンドを実行して、このファイルをバイナリ形式に変換し直します。

    ```sh
    ./scripts/json2wal/json2wal/tmp/corrupted_wal  $TMHOME/data/cs.wal/wal
    ```

## ハードウェア

### プロセッサとメモリ

実際の仕様は負荷やバリデーターの数によって異なりますが、最小
要件は次のとおりです。

-1GB RAM
-25GBのディスク容量
-1.4 GHz CPU

SSDディスクは、トランザクションスループットが高いアプリケーションに適しています。

尊敬される:

-2GBのRAM
-100GBソリッドステートドライブ
-x64 2.0 GHz 2v CPU

今のところ、テンダーミントはすべての履歴を保存しており、多くの履歴が必要になる場合があります
時間の経過とともにディスク容量を増やし、状態の同期を実現する予定です([this
問題](https://github.com/tendermint/tendermint/issues/828))。だから、すべてを保存します
過去のブロックは不要になります。

### ベリファイアは32ビットアーキテクチャ(またはARM)で署名されています

`ed25519`と` secp256k1`の実装には両方とも一定の時間が必要です
`uint64`乗算。非一定時間の暗号化がリークされる可能性があります(そしてリークされています)
`ed25519`と` secp256k1`の秘密鍵。これはハードウェアには存在しません
32ビットx86プラットフォーム([ソース](https://bearssl.org/ctmul.html))では、
それを強制するのはコンパイラに依存する一定の時間です。よくわからない
Golangコンパイラがすべての人に対してこれを正しく行うときはいつでも
達成。

** 32ビットアーキテクチャでのバリデーターの実行はサポートも推奨もされていません。
ARMパーツのアーキテクチャを評価した「VIANano2000シリーズ」
「S-」。**

### オペレーティング・システム

Goのおかげで、Tendermintはさまざまなオペレーティングシステム用にコンパイルできます
言語のリスト(\ $ OS /\ $ ARCHペアは、
[ここ](https://golang.org/doc/install/source#environment))。

オペレーティングシステムは好みませんが、より安全で安定したLinuxサーバー
ディストリビューション(Centosなど)は、デスクトップオペレーティングシステムよりも優先する必要があります
(Macオペレーティングシステムなど)。

ネイティブのWindowsサポートは提供されていません。 Windowsマシンを使用している場合は、[bashシェル](https://docs.microsoft.com/en-us/windows/wsl/install-win10)を試すことができます。

### さまざまなタイプ

注:パブリックドメインでTendermintを使用する場合は、次のことを確認してください。
[ハードウェアの推奨事項](https://cosmos.network/validators)でバリデーターを読みます
宇宙ネットワーク。

## 構成パラメーター

-`p2p.flush-throttle-timeout`
-`p2p.max-packet-msg-payload-size`
-`p2p.send-rate`
-`p2p.recv-rate`

プライベートドメインでTendermintを使用する予定で、
ピア間の専用高速ネットワーク、削減
スロットルタイムアウトを更新し、他のパラメーターを増やします。

```toml
[p2p]
send-rate=20000000 # 2MB/s
recv-rate=20000000 # 2MB/s
flush-throttle-timeout=10
max-packet-msg-payload-size=10240 # 10KB
```

-`mempool.recheck`

##ハードウェア

###プロセッサとメモリ

実際の仕様は負荷やバリデーターの数によって異なりますが、最小
要件は次のとおりです。

-1GB RAM
-25GBのディスク容量
-1.4 GHz CPU

SSDディスクは、トランザクションスループットが高いアプリケーションに適しています。

尊敬される:

-2GBのRAM
-100GBソリッドステートドライブ
-x64 2.0 GHz 2v CPU

今のところ、テンダーミントはすべての履歴を保存しており、多くの履歴が必要になる場合があります
時間の経過とともにディスク容量を増やし、状態の同期を実現する予定です([this
問題](https://github.com/tendermint/tendermint/issues/828))。だから、すべてを保存します
過去のブロックは不要になります。

###ベリファイアは32ビットアーキテクチャ(またはARM)で署名されています

`ed25519`と` secp256k1`の実装には両方とも一定の時間が必要です
`uint64`乗算。非一定時間の暗号化がリークされる可能性があります(そしてリークされています)
`ed25519`と` secp256k1`の秘密鍵。これはハードウェアには存在しません
32ビットx86プラットフォーム([ソース](https://bearssl.org/ctmul.html))では、
それを強制するのはコンパイラに依存する一定の時間です。よくわからない
Golangコンパイラがすべての人に対してこれを正しく行うときはいつでも
達成。

** 32ビットアーキテクチャでのバリデーターの実行はサポートも推奨もされていません。
ARMパーツのアーキテクチャを評価した「VIANano2000シリーズ」
「S-」。**

### オペレーティング・システム

Goのおかげで、Tendermintはさまざまなオペレーティングシステム用にコンパイルできます
言語のリスト(\ $ OS /\ $ ARCHペアは、
[ここ](https://golang.org/doc/install/source#environment))。

オペレーティングシステムは好みませんが、より安全で安定したLinuxサーバー
ディストリビューション(Centosなど)は、デスクトップオペレーティングシステムよりも優先する必要があります
(Macオペレーティングシステムなど)。

ネイティブのWindowsサポートは提供されていません。 Windowsマシンを使用している場合は、[bashシェル](https://docs.microsoft.com/en-us/windows/wsl/install-win10)を試すことができます。

###さまざまなタイプ

注:パブリックドメインでTendermintを使用する場合は、次のことを確認してください。
[ハードウェアの推奨事項](https://cosmos.network/validators)でバリデーターを読みます
宇宙ネットワーク。

##構成パラメーター

-`p2p.flush-throttle-timeout`
-`p2p.max-packet-msg-payload-size`
-`p2p.send-rate`
-`p2p.recv-rate`

プライベートドメインでTendermintを使用する予定で、
ピア間の専用高速ネットワーク、削減
スロットルタイムアウトを更新し、他のパラメーターを増やします。

```md
kern.maxfiles=10000+2*N         # BSD
kern.maxfilesperproc=100+2*N    # BSD
kern.ipc.maxsockets=10000+2*N   # BSD
fs.file-max=10000+2*N           # Linux
net.ipv4.tcp_max_orphans=N      # Linux

# For load-generating clients.
net.ipv4.ip_local_port_range="10000  65535"  # Linux.
net.inet.ip.portrange.first=10000  # BSD/Mac.
net.inet.ip.portrange.last=65535   # (Enough for N < 55535)
net.ipv4.tcp_tw_reuse=1         # Linux
net.inet.tcp.maxtcptw=2*N       # BSD

# If using netfilter on Linux:
net.netfilter.nf_conntrack_max=N
echo $((N/8)) >/sys/module/nf_conntrack/parameters/hashsize
```

gRPC接続の数を制限するための同様のオプションがあります-
`rpc.grpc-max-open-connections`。
