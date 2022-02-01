# ADR 061:P2P再構築スコープ

## 変更ログ

-2020年10月30日:初期バージョン(@erikgrinaker)

## 環境

ピアツーピアネットワークを担当する「p2p」パッケージは非常に古く、密結合、抽象リーク、テストの欠如、DoSの脆弱性、パフォーマンスの低下、カスタムプロトコル、不正な動作など、多くの弱点があります.リファクタリングは数年前から議論されてきました([#2067](https://github.com/tendermint/tendermint/issues/2067)).

Informal Systemsは、TendermintのRust実装[Tendermint-rs](https://github.com/informalsystems/tendermint-rs)も構築しており、来年はP2Pネットワークサポートを実装する予定です.この作業の一環として、Tendermintプログラムで現在使用されているカスタムアプリケーションを実装する代わりに、トランスポートプロトコルとして[QUIC](https://datatracker.ietf.org/doc/draft-ietf-quic-transport/)などを要求しました. -レベルの `MConnection`ストリーム多重化プロトコル.

このADRは、P2P再構築の範囲に関する利害関係者との最近の議論をまとめたものです.特定の設計と実装は、個別のADRとして提出されます.

## 代替方法

独自のP2Pネットワークスタックを維持する代わりに[LibP2P](https://libp2p.io)を採用する提案が繰り返されています([#3696](https://github.com/tendermint/tendermint/issues/3696を参照) ).これは原則としては良い考えのように思えますが、非常に破壊的なプロトコルの変更になります.LibP2Pをフォークして変更しなければならない兆候があり、使用される抽象化について懸念があります.

Informal Systemsとの話し合いでは、現在のP2Pスタックを段階的に改善し、プラグイン可能な伝送のサポートを追加することから始め、その後、徐々にLibP2Pを伝送層として試し始めました.これが成功した場合は、後で上位レベルのコンポーネントに使用することを検討できます.

## 決定

P2Pスタックは、いくつかの段階でリファクタリングおよび改善されます.

* **フェーズ1:**プロトコルの互換性を可能な限り維持するためのコードとAPIのリファクタリング.

* **フェーズ2:**追加の送信と増分プロトコルの改善.

* **フェーズ3:**破壊的な合意の変更.

フェーズ2とフェーズ3の範囲はまだ不明です.前のフェーズが完了したら、ニーズと課題をよりよく理解できるようになるため、フェーズ2とフェーズを再検討します.

## 詳細設計

調査とプロトタイピングの後、各段階で特定の設計と変更のために個別のADRが提出されます.以下は、優先順位の高い順に目標です.

### フェーズ1:コードとAPIのリファクタリング

この段階では、p2pパッケージの内部抽象化と実装の改善に焦点を当てます.後方互換性のない方法でP2Pプロトコルを変更しないようにしてください.

* `Reactor`、` Switch`、 `Peer`などのより明確で分離された抽象化. [#2067](https://github.com/tendermint/tendermint/issues/2067)[#5287](https://github.com/tendermint/tendermint/issues/5287)[#3833](https:/ //github.com/tendermint/tendermint/issues/3833)
    * Reactorは、別のゴルーチンまたはバッファーチャネルを介してメッセージを受信する必要があります. [#2888](https://github.com/tendermint/tendermint/issues/2888)
*ピアライフサイクル管理が改善されました. [#3679](https://github.com/tendermint/tendermint/issues/3679)[#3719](https://github.com/tendermint/tendermint/issues/3719)[#3653](https:///github.com/tendermint/tendermint/issues/3653)[#3540](https://github.com/tendermint/tendermint/issues/3540)[#3183](https://github.com/tendermint/テンダーミント/ issues/3183)[#3081](https://github.com/tendermint/tendermint/issues/3081)[#1356](https://github.com/tendermint/tendermint/issues/1356)
    *ピアの優先順位. [#2860](https://github.com/tendermint/tendermint/issues/2860)[#2041](https://github.com/tendermint/tendermint/issues/2041)
*実装として `MConnection`を使用したプラグ可能な送信. [#5587](https://github.com/tendermint/tendermint/issues/5587)[#2430](https://github.com/tendermint/tendermint/issues/2430)[#805](https:/ //github.com/tendermint/tendermint/issues/805)
*ピアアドレス処理が改善されました.
    *名簿の再構築. [#4848](https://github.com/tendermint/tendermint/issues/4848)[#2661](https://github.com/tendermint/tendermint/issues/2661)
    *送信とは関係のないピアツーピアアドレス指定. [#5587](https://github.com/tendermint/tendermint/issues/5587)[#3782](https://github.com/tendermint/tendermint/issues/3782)[#3692](https:/ //github.com/tendermint/tendermint/issues/3692)
    *自分のアドレスの検出とアドバタイズメントを改善しました. [#5588](https://github.com/tendermint/tendermint/issues/5588)[#4260](https://github.com/tendermint/tendermint/issues/4260)[#3716](https:///github.com/tendermint/tendermint/issues/3716)[#1727](https://github.com/tendermint/tendermint/issues/1727)
    *各ピアは複数のIPをサポートします. [#1521](https://github.com/tendermint/tendermint/issues/1521)[#2317](https://github.com/tendermint/tendermint/issues/2317)

リファクタリングは、テスト容易性、可観測性、パフォーマンス、セキュリティ、サービス品質、バックプレッシャー、およびDoS復元力という2番目の目標に対処するように努める必要があります.これらのほとんどは、フェーズ2の明確な目標として再検討されます.

理想的には、リファクタリングは徐々に実行し、数週間ごとに定期的に「マスター」にマージする必要があります.全体として、これには時間がかかり、内部Go APIに頻繁に大きな変更が加えられますが、ブランチドリフトが減少し、コードがより高速になり、より広範囲にテストされます.

### フェーズ2:追加の送信とプロトコルの改善

このフェーズでは、プロトコルの改善とその他の主要な変更に焦点を当てます.以下は、リファクタリングの完了後に個別に評価する必要がある推奨事項です.他の提案はフェーズ1で追加される可能性があります.

* QUIC送信. [#198](https://github.com/tendermint/spec/issues/198)
*シークレット接続ハンドシェイクのノイズプロトコル. [#5589](https://github.com/tendermint/tendermint/issues/5589)[#3340](https://github.com/tendermint/tendermint/issues/3340)
*接続ハンドシェイクのピアID. [#5590](https://github.com/tendermint/tendermint/issues/5590)
*ピアとサービスの検出(RPCノード、状態同期スナップショットなど). [#5481](https://github.com/tendermint/tendermint/issues/5481)[#4583](https://github.com/tendermint/tendermint/issues/4583)
*レート制限、バックプレッシャ、およびQoSスケジューリング. [#4753](https://github.com/tendermint/tendermint/issues/4753)[#2338](https://github.com/tendermint/tendermint/issues/2338)
*圧縮. [#2375](https://github.com/tendermint/tendermint/issues/2375)
*改善されたインジケーターと追跡. [#3849](https://github.com/tendermint/tendermint/issues/3849)[#2600](https://github.com/tendermint/tendermint/issues/2600)
*簡略化されたP2P構成オプション.
###フェーズ3:破壊的なプロトコルの変更

この段階では、明確に定義されておらず、非常に不確実な投機的で幅広い提案を取り上げます.最初のいくつかの段階が完了すると、それらが評価されます.

* LibP2Pを採用します. [#3696](https://github.com/tendermint/tendermint/issues/3696)
*クロスリアクター通信を許可します.チャネルがない場合があります.
*リアクターが有効/無効になっているため、動的チャネル広告. [#4394](https://github.com/tendermint/tendermint/issues/4394)[#1148](https://github.com/tendermint/tendermint/issues/1148)
*ネットワークトポロジとモードをパブリッシュおよびサブスクライブします.
*同じネットワークで複数のチェーンIDをサポートします.

## ステータス

受け入れられました

## 結果

### ポジティブ

*より簡潔でシンプルなアーキテクチャであり、推論とテストが容易であるため、エラーを減らしたいと考えています.

*パフォーマンスと堅牢性が向上しました.

* QUICやNoiseなどの標準化されたプロトコルを採用することで、メンテナンスの負担を軽減し、相互運用性を向上させます.

*可用性の向上、可観測性の向上、構成の簡素化、自動化(ピア/サービス/アドレスの検出、レート制限、バックプレッシャーなど).

### ネガティブ

*独自のP2Pネットワークスタックを維持するには、リソースを大量に消費します.

*基になる送信を抽象化すると、高度な送信機能を使用できなくなる可能性があります.

* APIとプロトコルに大きな変更を加えると、ユーザーに損害を与える可能性があります.

## 参照する

上記の質問リンクを参照してください.

-[#2067:P2Pリファクタリング](https://github.com/tendermint/tendermint/issues/2067)

-[P2Pリファクタリングブレーンストーミングドキュメント](https://docs.google.com/document/d/1FUTADZyLnwA9z7ndayuhAdAFRKujhh_y73D0ZFdKiOQ/edit?pli=1#)
