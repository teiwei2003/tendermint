# プロのようにデバッグする

## 導入する

Tendermint Coreは、非常に強力なBFTレプリケーションエンジンです。 残念ながら、他のソフトウェアと同様に、失敗することがあります。 問題は、システムが期待される動作から逸脱したときに「何をするか」です。

最初の反応は通常、ログを確認することです。 デフォルトでは、Tendermintはログを標準出力¹に書き込みます。

```sh
I[2020-05-29|03:03:16.145] Committed state                              module=state height=2282 txs=0 appHash=0A27BC6B0477A8A50431704D2FB90DB99CBFCB67A2924B5FBF6D4E78538B67C1I[2020-05-29|03:03:21.690] Executed block                               module=state height=2283 validTxs=0 invalidTxs=0I[2020-05-29|03:03:21.698] Committed state                              module=state height=2283 txs=0 appHash=EB4E409D3AF4095A0757C806BF160B3DE4047AC0416F584BFF78FC0D44C44BF3I[2020-05-29|03:03:27.994] Executed block                               module=state height=2284 validTxs=0 invalidTxs=0I[2020-05-29|03:03:28.003] Committed state                              module=state height=2284 txs=0 appHash=3FC9237718243A2CAEE3A8B03AE05E1FC3CA28AEFE8DF0D3D3DCE00D87462866E[2020-05-29|03:03:32.975] enterPrevote: ProposalBlock is invalid       module=consensus height=2285 round=0 err="wrong signature (#35): C683341000384EA00A345F9DB9608292F65EE83B51752C0A375A9FCFC2BD895E0792A0727925845DC13BA0E208C38B7B12B2218B2FE29B6D9135C53D7F253D05"
```

本番環境でバリデーターを実行している場合は、filebeatまたは同様のツールを使用して、分析のためにログを転送することをお勧めします。 さらに、エラーが発生したときに通知を設定できます。

ログは、何が起こったかの基本的な理解を与えるはずです。 最悪の場合、ノードは停止し、ログを生成しません(または単にパニックになります)。

次のステップは、/status、/net_info、/consensus_state、および/dump_consensus_stateRPCエンドポイントを呼び出すことです。
```sh
curl http://<server>:26657/status$ curl http://<server>:26657/net_info$ curl http://<server>:26657/consensus_state$ curl http://<server>:26657/dump_consensus_state
```

ノードが停止している場合、/consensus_stateと/dump_consensus_stateは結果を返さない可能性があることに注意してください(コンセンサスミューテックスを取得しようとしているため)。

これらのエンドポイントの出力には、開発者がノードの状態を理解するために必要なすべての情報が含まれています。 ノードがネットワークの背後にあるかどうか、接続されているピアの数、および最新のコンセンサス状態が通知されます。

この時点で、ノードが停止していて再起動したい場合は、-6信号でノードを強制終了するのが最善の方法です。
```sh
kill -6 <PID>
```

これにより、現在実行中のgoroutineのリストがダンプされます。 このリストは、デッドロックをデバッグするときに非常に役立ちます。

`PID`はTendermintのプロセスIDです。 `ps -a |を実行することで見つけることができます grep mint | awk '{print $ 1}' `

## Tendermintデバッグキル

さまざまなデータフラグメントを収集する負担を軽減するために、Tendermint Core(v0.33以降)はTendermintのデバッグおよび強制終了ツールを提供します。これにより、上記のすべての手順が完了し、すべてのコンテンツが適切なアーカイブファイルにパックされます。
```sh
tendermint debug kill <pid> </path/to/out.zip> — home=</path/to/app.d>
```

これは公式のドキュメントページです-<https://docs.tendermint.com/master/tools/debugging>

systemdなどのプロセスマネージャーを使用すると、Tendermintが自動的に再起動します。 本番環境にインストールすることを強くお勧めします。 そうでない場合は、ノードを手動で再起動する必要があります。

Tendermintを使用したデバッグのもう1つの利点は、ソフトウェアに問題があると思われる場合に、同じアーカイブファイルをTendermintコア開発者に提供できることです。

## Tendermintデバッグダンプ

さて、ノードが停止せず、その状態が時間の経過とともに低下した場合はどうなりますか？ テンダーミントのデバッグダンプが救いの手を差し伸べます！

```sh
tendermint debug dump </path/to/out> — home=</path/to/app.d>
```

ノードを強制終了することはありませんが、上記のすべてのデータを収集してアーカイブファイルにパックします。 さらに、ヒープダンプも実行します。これは、Tendermintのメモリリークが発生した場合に役立ちます。

この時点で、ダウングレードの重大度によっては、プロセスを再開する必要がある場合があります。

## テンダーミントチェック

コンセンサス状態に一貫性がないためにTendermintノードを開始できない場合はどうなりますか？

Tendermintコンセンサスエンジンを実行しているノードが不整合な状態を検出した場合
Tendermintプロセス全体がクラッシュします。
Tendermintコンセンサスエンジンはこの一貫性のない状態では実行できないため、ノードは
その結果、起動しません。
この場合、TendermintRPCサーバーはデバッグに役立つ情報を提供できます。
Tendermintの `inspect`コマンドは、TendermintRPCサーバーのサブセットを実行します
これは、一貫性のない状態をデバッグする場合に役立ちます。

### チェックを実行

次のコマンドを使用して、Tendermintがクラッシュしたマシンで検査ツールを起動します。
```bash
tendermint inspect --home=</path/to/app.d>
```

`inspect`は、Tendermint構成ファイルで指定されたデータディレクトリを使用します。
`inspect`は、Tendermint構成ファイルで指定されたアドレスでRPCサーバーも実行します。

### チェックを使用

`inspect`サーバーを実行すると、非常に重要なRPCエンドポイントにアクセスできます
デバッグに使用されます。
`/status`、` /consensus_state`、および `/dump_consensus_state`RPCエンドポイントを呼び出します
テンダーミントのコンセンサスステータスに関する有用な情報を返します。

## 終わり

これらのテンダーミントツールが事故への最初の対応になることを願っています。

これまでの経験を教えてください！ 「tendermintdebug」または「tendermintinspect」を試す機会はありますか？

[discordチャット](https://discord.gg/cosmosnetwork)に参加して、現在の問題と将来の改善について話し合ってください。

—

[1]:もちろん、Tendermintの出力をファイルにリダイレクトしたり、別のサーバーに転送したりすることは自由です。
