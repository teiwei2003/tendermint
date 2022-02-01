# ADR 046:Liteクライアントの実装

## 変更ログ
* 13-02-2020:最初のドラフト
* 26-02-2020:最初の見出しをクロスチェックします
* 28-02-2020:二分アルゴリズムの詳細
* 31-03-2020:検証署名が変更されました

## 環境

`Client`構造は、単一のブロックチェーンに接続されたライトクライアントを表します.

ユーザーは `VerifyHeader`ま​​たは
`VerifyHeaderAtHeight`または` Update`メソッド. 後者の方法をダウンロードする
メインの最新のヘッダーから、現在の信頼できるヘッダーと比較します.

```go
type Client interface {
	// verify new headers
	VerifyHeaderAtHeight(height int64, now time.Time) (*types.SignedHeader, error)
	VerifyHeader(newHeader *types.SignedHeader, newVals *types.ValidatorSet, now time.Time) error
	Update(now time.Time) (*types.SignedHeader, error)

	// get trusted headers & validators
	TrustedHeader(height int64) (*types.SignedHeader, error)
	TrustedValidatorSet(height int64) (valSet *types.ValidatorSet, heightUsed int64, err error)
	LastTrustedHeight() (int64, error)
	FirstTrustedHeight() (int64, error)

	// query configuration options
	ChainID() string
	Primary() provider.Provider
	Witnesses() []provider.Provider

	Cleanup() error
}
```

新しいライトクライアントは、( `NewClient`を介して)最初から作成するか、
信頼できるストアを使用します( `NewClientFromTrustedStore`経由). あるとき
ストレージ内のデータを信頼し、「NewClient」を呼び出します.ライトクライアントはa)
保存されたヘッダーが更新されているかどうかを確認しますb)オプションでいつでもユーザーに確認します
ロールバックする必要があります(デフォルトでは確認は必要ありません).

```go
func NewClient(
	chainID string,
	trustOptions TrustOptions,
	primary provider.Provider,
	witnesses []provider.Provider,
	trustedStore store.Store,
	options ...Option) (*Client, error) {
```

( `Option`ではなく)パラメータとしての` witnesses`は意図的な選択です.
デフォルトでセキュリティが強化されています. 少なくとも1人の証人が必要です.
現在、ライトクライアントはメインの！=証人をチェックしません.
新しいタイトルを目撃者とクロスチェックするときの目撃者の最小数
必要な応答:1.最初のヘッダー( `TrustOptions.Hash`)は
また、安全性を高めるために目撃者とクロスチェックしてください.

バイナリアルゴリズムの性質上、一部のタイトルはスキップされる場合があります. 軽い場合
クライアントには、高さが「X」で「VerifyHeaderAtHeight(X)」のヘッダーがありません.
`VerifyHeader(H#X)`メソッドが呼び出され、これらのメソッドはa)逆方向に実行されます
最新のヘッダーから高さ「X」のヘッダーの検証に戻る、またはb)
最初に保存されたタイトルから高さ「X」のタイトルまでのバイナリ検証.

`TrustedHeader`、` TrustedValidatorSet`は信頼できるストアとのみ通信します.
ヘッダーが存在しない場合は、エラーが返され、
確認する必要があります.

```go
type Provider interface {
	ChainID() string

	SignedHeader(height int64) (*types.SignedHeader, error)
	ValidatorSet(height int64) (*types.ValidatorSet, error)
}
```

プロバイダーは通常、完全なノードですが、別のライトクライアントにすることもできます. その上
インターフェイスは非常に薄く、多くの実装に対応できます.

プロバイダー(プライマリーまたはウィットネス)が長期間利用できない場合
時間、スムーズな操作を確保するために削除されます.

`Client`とプロバイダーの両方がチェーンIDを開示して、同じチェーンが存在するかどうかを追跡します
鎖. チェーンをアップグレードしたり、意図的にフォークしたりすると、チェーンIDが変更されることに注意してください.

ライトクライアントは、ヘッダーとベリファイアを信頼できるストレージに保存します.
```go
type Store interface {
	SaveSignedHeaderAndValidatorSet(sh *types.SignedHeader, valSet *types.ValidatorSet) error
	DeleteSignedHeaderAndValidatorSet(height int64) error

	SignedHeader(height int64) (*types.SignedHeader, error)
	ValidatorSet(height int64) (*types.ValidatorSet, error)

	LastSignedHeaderHeight() (int64, error)
	FirstSignedHeaderHeight() (int64, error)

	SignedHeaderAfter(height int64) (*types.SignedHeader, error)

	Prune(size uint16) error

	Size() uint16
}
```

現在、唯一の実装は `db`ストレージ(KV周辺)です
データベース、Tendermintで使用). 将来的には、リモートアダプタが可能です
(例: `Postgresql`).

```go
func Verify(
	chainID string,
	trustedHeader *types.SignedHeader, // height=X
	trustedVals *types.ValidatorSet, // height=X or height=X+1
	untrustedHeader *types.SignedHeader, // height=Y
	untrustedVals *types.ValidatorSet, // height=Y
	trustingPeriod time.Duration,
	now time.Time,
	maxClockDrift time.Duration,
	trustLevel tmmath.Fraction) error {
```

`Verify`純粋関数は、ヘッダー検証に公に使用されます.同時に処理します
隣接ヘッダーと非隣接ヘッダーの場合.前者の場合、比較します
直接ハッシュ(2/3以上の署名変換).それ以外の場合は、1/3 +を検証します
( `trustLevel`)信頼できるベリファイアはまだ新しいベリファイアに存在します.

`Verify`関数は確かに便利ですが、` VerifyAdjacent`と
`VerifyNonAdjacent`は、論理エラーを回避するために最も頻繁に使用する必要があります.

###除算アルゴリズムの詳細

仕様には含まれていますが、非再帰的な二分法アルゴリズムが実装されています
再帰バージョン.主な理由は2つあります.

1)継続的なメモリ消費=> OOM(メモリ不足)例外のリスクはありません.
2)より高速な終了(図1を参照).

_示されているように. 1:再帰的および非再帰的二分法の違い_

！ [示されているように. 1](./img/adr-046-fig1.png)

非再帰的二分法の基準を見つけることができます
[こちら](https://github.com/tendermint/spec/blob/zm_non-recursive-verification/spec/consensus/light-client/non-recursive-verification.md).

## ステータス

実装

## 結果

### ポジティブ

*単一の `Client`構造、使いやすい
*ヘッダープロバイダーと信頼できるストレージ間の柔軟なインターフェイス

### ネガティブ

* `Verify`は現在の仕様と一致している必要があります

### ニュートラル

* `Verify`関数は誤用される可能性があります(
  (誤って実装されたシーケンス検証)
