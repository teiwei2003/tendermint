# ADR 50:信頼できるピアの相互接続の改善

##変更ログ
* 22-10-2019:最初のドラフト
* 05-11-2019: `maximum-dial-period`を` persistent-peers-max-dial-period`に変更

## 環境

ノードの `max-num-inbound-peers`または` max-num-outbound-peers`に到達すると、ノードはどのピアにもそれ以上のタイムスロットを割り当てることができません。
インバウンドまたはアウトバウンドを渡します。したがって、一定期間の切断後、重要なピアツーピア接続が無期限に失われる可能性があります
すべてのタイムスロットが他のピアによって消費されるため、ノードはピアにダイヤルしようとしなくなります。

この状況には、指数バックオフと信頼できるピアの無条件ピアリング機能の欠如という2つの理由があります。


## 決定

`config.toml`に` unconditional-peer-ids`と2つのパラメータを導入することでこの問題を解決することをお勧めします
「永続的なピアツーピアの最大ダイヤル期間」。

1) `無条件のピアID`

ノードオペレータは、インバウンド接続またはアウトバウンド接続を許可するピアノードのIDのリストを入力します。
ユーザーノードの「max-num-inbound-peers」または「max-num-outbound-peers」が到着したかどうか。

2) `persistent-peers-max-dial-period`

指数バックオフ期間中、各ダイヤルアップから各永続ピアまでの期間は、「persistent-peers-max-dial-period」を超えることはありません。
したがって、 `dial-period` = min(` persistent-peers-max-dial-period`、 `exponential-backoff-dial-period`)

別の方法

Persistent-peerはアウトバウンドにのみ使用されるため、「unconditional-peer-ids」の完全なユーティリティをカバーするだけでは不十分です。
@ creamers158(https://github.com/Creamers158)は、IDのみのプロジェクトを永続的なピアに配置することを提案しています。
`unconditional-peer-ids`ですが、永続的なピアノードで異なる構造を持つアイテムを処理するには、非常に複雑な構造上の例外が必要です。
したがって、このユースケースを個別にカバーするために「unconditional-peer-ids」を使用することにしました。

## ステータス

提案

## 結果

### ポジティブ

ノードオペレータは、 `config.toml`で2つの新しいパラメータを設定できるため、tendermintが接続を許可していることを確認できます。
`unconditional-peer-ids`のピアから/へ。さらに、彼/彼女は各永続的なピアが少なくとも
`persistent-peers-max-dial-period`用語。信頼できるピアに対して、より安定した永続的なピアツーピア相互接続を実現します。

### ネガティブ

この新機能により、 `config.toml`に2つの新しいパラメーターが導入され、ノードオペレーターの説明が必要になります。

### ニュートラル

## 参照する

* 2つのp2p機能拡張の提案(https://github.com/tendermint/tendermint/issues/4053)
