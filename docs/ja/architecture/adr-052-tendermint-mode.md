# ADR 052:テンダーミントモード

## 変更ログ

* 27-11-2019:ADR-051からの最初のドラフト
* 13-01-2020:ADRTendermintモードをADR-051から分離
* 29-03-2021:デフォルト値に関する情報を更新

## 環境

-フルモード:フルモードには、バリデーターになる機能はありません。
-バリデーターモード:このモードは、既存のステートマシンとまったく同じ動作をします。同時に投票せずにコンセンサスを作成し、完全に同期したときにコンセンサスに参加する
-シードモード:軽量シードノードはアドレスブックを維持します。p2pは[TenderSeed](https://gitlab.com/polychainlabs/tenderseed)に似ています。

## 決定

テンダーミントパターンの簡単な抽象化を提案したいと思います。これらのパターンはバイナリファイルに存在し、ノードを初期化するときに、ユーザーは作成するノードを指定できます。

-各ノードに含まれるリアクターとコンポーネント
    - 満杯
        -切り替え、輸送
        -原子炉
          -メモリプール
          -コンセンサス
          - 証拠
          -ブロックチェーン
          -p2p/pex
          -状態の同期
        -rpc(安全な接続のみ)
        -* ~~ privValidator(priv_validator_key.json、priv_validator_state.json)なし~~ *
    -オーセンティケーター
        -切り替え、輸送
        -原子炉
          -メモリプール
          -コンセンサス
          - 証拠
          -ブロックチェーン
          -p2p/pex
          -状態の同期
        -rpc(安全な接続のみ)
        -privValidator(priv_validator_key.json、priv_validator_state.json)を使用します
    -シード
        -切り替え、輸送
        -原子炉
           -p2p/pex
-構成、cliコマンド
    -`config.toml`とcliに `mode`パラメータを導入することをお勧めします
    -<span v-pre> `mode =" {{.BaseConfig.Mode}} "` </ span> in `config.toml`
    -CLIの `tendermint start --modevalidator`
    -フル|バリデーター|シードノード
    -デフォルト値はありません。ユーザーは、 `tendermintinit`をいつ実行するかを指定する必要があります
-RPCの変更
    -`ホスト:26657 /ステータス `
        -フルモードで空の `validator_info`を返します
    -シードノードにrpcサーバーがありません
-コードベースの変更された場所
    -`node/node.go:DefaultNewNode`に `config.Mode`のスイッチを追加します
    -`config.Mode == validator`の場合、デフォルトの `NewNode`(現在のロジック)を呼び出します
    -`config.Mode == full`の場合は、 `NewNode`と` nil` `privValidator`を呼び出します(ロードまたは生成しないでください)
        -関連する関数に `nil``privValidator`の例外ルーチンを追加する必要があります
    -`config.Mode == seed`の場合、 `NewSeedNode`(` node/node.go:NewNode`のシードノードバージョン)を呼び出します
        -`nil`、 `reactor`、` component`の関連関数に例外ルーチンを追加する必要があります

## ステータス

実装

## 結果

### ポジティブ

-ノードオペレータは、ノードの目的に応じてステートマシンの実行モードを選択できます。
-ユーザーはフラグを介して実行するモードを指定する必要があるため、モードはエラーを防ぐことができます。 (たとえば、ユーザーがバリデーターノードを実行したい場合、ユーザーはバリデーターをパターンとして明示的に記述する必要があります)
-異なるモデルでは、効率的なリソース利用を実現するために異なるリアクターが必要です。

### ネガティブ

-ユーザーは、各モードがどのように機能し、どのような機能を備えているかを調べる必要があります。

### ニュートラル

## 参照する

-問題[#2237](https://github.com/tendermint/tendermint/issues/2237):テンダーミントの「モード」
-[TenderSeed](https://gitlab.com/polychainlabs/tenderseed):軽量のTendermintシードノード。
