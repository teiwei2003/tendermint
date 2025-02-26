# ADR 068:逆同期

## 変更ログ

-2021年4月20日:最初のドラフト(@cmwaters)

## ステータス

受け入れられました

## 環境

状態の同期とブロックのプルーニングの出現により、完全なブロック履歴を必要とせずに、完全なノードがコンセンサスに参加する機会が提供されます.これはまた、証拠の取り扱いに問題を引き起こします.プルーフ時代にすべてのブロックを所有していなかったノードはプルーフを検証できないため、プルーフがチェーンに送信されると停止します.

[RFC005](https://github.com/tendermint/spec/blob/master/rfc/005-reverse-sync.md)がこの問題に対応してリリースされ、最小のブロック履歴不変条件を追加するように仕様が変更されました. .これは主に、状態の同期を拡張して、「Header」、「Commit」、「ValidatorSet」(基本的には「LightBlock」)の最後の「n」の高さを取得して保存できるようにするためです.ここで、「n」はFromの計算に基づいています.証拠の時代.

このADRは、この状態同期拡張機能の設計、ライトクライアントプロバイダーの変更、およびtmストレージのマージについて説明することを目的としています.

## 決定

状態同期リアクターは、2つの新しいP2Pメッセージ(および新しいチャネル)を導入することによって拡張されます.

```protobuf
message LightBlockRequest {
  uint64 height = 1;
}

message LightBlockResponse {
  tendermint.types.LightBlock light_block = 1;
}
```

これは、ノードがコンセンサスに安全に参加できるように、以前のライトブロックを取得、検証、および保存する「逆同期」プロトコルによって使用されます.

さらに、これにより、新しいライトクライアントプロバイダーは、RPCの代わりに基盤となるP2Pスタックを使用する機能を `StateProvider`に提供できます.

## 詳細設計

このセクションでは、最初に独立したプロトコルとしての逆同期(ここでは「バックフィル」と呼びます)メカニズムに焦点を当て、次にそれが状態同期リアクターに統合される方法と、新しいp2pライトクライアントプロバイダーを定義する方法について説明します.

```go
// Backfill fetches, verifies, and stores necessary history
// to participate in consensus and validate evidence.
func (r *Reactor) backfill(state State) error {}
```

`State`は、戻る距離を計算するために使用されます.つまり、次の特性を持つすべてのライトブロックが必要です.
-高さ: `h> = state.LastBlockHeight-state.ConsensusParams.Evidence.MaxAgeNumBlocks`
-時間: `t> = state.LastBlockTime-state.ConsensusParams.Evidence.MaxAgeDuration`

逆同期は、「Dispatcher」と「BlockQueue」の2つのコンポーネントに依存しています. `Dispatcher`は、同様の[PR](https://github.com/tendermint/tendermint/pull/4508)から取得したモデルです. 「LightBlockChannel」に接続し、ピアのリンクリスト内を移動することでブロック要求を許可および発行します.この種の抽象化は非常に優れた品質を備えており、P2Pに基づくライトクライアントのライトプロバイダーのアレイとしても使用できます.

「BlockQueue」は、複数のワーカーが軽量ブロックを取得し、メインスレッド用にシリアル化できるようにするデータ構造です.メインスレッドは、キューの最後からそれらを選択し、ハッシュを検証して永続化します.

### 状態同期との統合

逆同期は、同期状態の直後で、高速同期またはコンセンサスに移行する前に実行されるブロッキングプロセスです.

以前は、状態同期サービスはどのデータベースにも接続しませんでしたが、状態をノードに戻しました.逆同期の場合、状態同期には `StateStore`と` BlockStore`へのアクセスが許可され、 `Header`、` Commit`、 `ValidatorSet`に書き込み、それらを読み取って他の状態同期ピアにサービスを提供できるようになります.

これは、これらのそれぞれのストアに新しいメソッドを追加して、それらを維持することも意味します

### P2Pライトクライアントプロバイダー

前述のように、「ディスパッチャ」は複数のピアへの要求を処理できます.したがって、各ピアに割り当てられたblockProviderインスタンスを簡単に取り除くことができます.チェーンIDを指定することで、 `blockProvider`はライトブロックをクライアントに返す前に基本的な検証を実行できます.

状態同期は証拠チャネルにアクセスできないため、ライトクライアントが証拠を報告することを許可できないため、「ReportEvidence」は無効であることに注意してください.これは逆同期の問題ではありませんが、純粋なp2pライトクライアントでは解決する必要があります.

### プルーン

最後の小さなメモは剪定です.このADRは、アプリケーションが証拠の時代にブロックをプルーニングすることを許可しない変更を導入します.

## 将来の仕事

このADRは、拡張状態同期の範囲内にとどまろうとしますが、行われた変更により、フォローアップが必要ないくつかの領域への扉が開かれます.
-p2pメッセージングをライトクライアントパッケージに正しく統合します.これには、ライトクライアントが証拠を報告できるように証拠チャネルを追加する必要があります.また、プロバイダーモデルを再検討する必要がある場合もあります(つまり、現在のプロバイダーは起動時にのみ追加されます)
-ミントストレージ(ステータス、ブロック、証拠)を統合してクリーンアップします.このADRは、ヘッダー、送信、およびバリデーターセットを保存するための状態およびブロックストレージに新しいメソッドを追加します.これは現在の構造には適していません(つまり、 `Header`ではなく` BlockMeta`のみが保存されます).アトミック性とバッチ処理の機会については、この統合のポイントを調査する必要があります.ブロックパーツの保管方法など、他にも変更があります.詳細については、[こちら](https://github.com/tendermint/tendermint/issues/5383)および[こちら](https://github.com/tendermint/tendermint/issues/4630)をご覧ください.
-逆同期の機会を探ります.技術的に言えば、証拠が観察されない場合は、逆同期は必要ありません.私は、適切と思われる場合にエビデンスパッケージに転送できるようにプロトコルを設計しようとしました.したがって、必要なデータがない証拠が見つかった場合にのみ、逆同期を実行します.問題は、コンセンサスに到達し、最後の10,000ブロックを最初に取得して検証するように求める証拠が表示されると仮定します.ノードはこの操作を(順番に)実行して、ラウンドが終了する前に投票することはできません.さらに、無効な証拠にペナルティを課さないため、悪意のあるノードがチェーンにスパムを簡単に送信して、一連の「ステートレス」ノードに大量の無用な作業を実行させることができます.
-完全な逆同期を調べます.現在、ライトブロックのみを取得しています.ブロック全体を取得して永続化することは、特にこの操作を実行するようにアプリケーションに制御を与える場合、将来的に有益になる可能性があります.

## 結果

### ポジティブ

-すべてのノードには、すべてのタイプの証拠を検証するのに十分な履歴が必要です
-状態同期ノードは、状態ライトクライアントの検証にp2pレイヤーを使用できます.これはユーザーエクスペリエンスが向上し、高速になる可能性がありますが、ベンチマークは行いませんでした.

### ネガティブ

-より多くのコードを導入する=より多くのメンテナンス

### ニュートラル

## 参照する

-[リバース同期RFC](https://github.com/tendermint/spec/blob/master/rfc/005-reverse-sync.md)
-[元の問題](https://github.com/tendermint/tendermint/issues/5617)
