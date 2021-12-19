# ADR 012:PeerTransport

## 環境

p2pの現在のアーキテクチャに関するより明白な問題の1つ
異なるパッケージ間で関心の分離は明確ではありません
コンポーネント。最も注目すべきは、「スイッチ」が現在物理的に接続されていることです
対処する。 1つのアーティファクトは、スイッチオンの依存関係です
`[config.P2PConfig`](https://github.com/tendermint/tendermint/blob/05a76fb517f50da27b4bfcdc7b4cf185fc61eff6/config/config.go#L272-L339)。

住所:

-[#2046](https://github.com/tendermint/tendermint/issues/2046)
-[#2047](https://github.com/tendermint/tendermint/issues/2047)

[#2067](https://github.com/tendermint/tendermint/issues/2067)の最初のイテレーション

## 決定

送信の問題は、新しいコンポーネント( `PeerTransport`)によって処理されます。
ピアは、その境界で発信者に提供されます。次に、 `Switch`は
この新しいコンポーネントは、新しい「ピア」を受け入れ、「NetAddress」に従ってダイヤルします。

### PeerTransport

ピアの起動と接続を担当します。 `Peer`の実装
トランスミッションに任されています。これは、選択されたトランスミッションが決定することを意味します
実装された機能は「スイッチ」に返されます。各
トランスポートの実装は、ピア固有を確立するためのフィルタリングを担当します
そのドメインに対して、デフォルトの多重化実装の場合、次のようになります
申し込み:

-私たち自身のノードからの接続
-ハンドシェイクに失敗しました
-シークレット接続へのアップグレードに失敗しました
-重複するIPを防止します
-IDの重複を防ぐ
-nodeinfoは互換性がありません

```go
// PeerTransport proxies incoming and outgoing peer connections.
type PeerTransport interface {
	// Accept returns a newly connected Peer.
	Accept() (Peer, error)

	// Dial connects to a Peer.
	Dial(NetAddress) (Peer, error)
}

// EXAMPLE OF DEFAULT IMPLEMENTATION

// multiplexTransport accepts tcp connections and upgrades to multiplexted
// peers.
type multiplexTransport struct {
	listener net.Listener

	acceptc chan accept
	closec  <-chan struct{}
	listenc <-chan struct{}

	dialTimeout      time.Duration
	handshakeTimeout time.Duration
	nodeAddr         NetAddress
	nodeInfo         NodeInfo
	nodeKey          NodeKey

	// TODO(xla): Remove when MConnection is refactored into mPeer.
	mConfig conn.MConnConfig
}

var _ PeerTransport = (*multiplexTransport)(nil)

// NewMTransport returns network connected multiplexed peers.
func NewMTransport(
	nodeAddr NetAddress,
	nodeInfo NodeInfo,
	nodeKey NodeKey,
) *multiplexTransport
```

### 変化する

今後、Switchは完全に設定された「PeerTransport」に依存します
相手を検索/連絡します。 より低レベルの注意が向けられるにつれて
送信プロセス中に、スイッチへの `config.P2PConfig`の受け渡しを省略できます。

```go
func NewSwitch(transport PeerTransport, opts ...SwitchOption) *Switch
```

## ステータス

審査中です。

## 結果

### ポジティブ

-伝送の問題からの自由な切り替え-より簡単な実装
-プラガブルトランスミッションの実装-よりシンプルなテストセットアップ
-スイッチのP2PConfigへの依存を削除します-テストが簡単

### ネガティブ

-Switchに依存するテストのその他の設定

### ニュートラル

-多重化がデフォルトの実装になります

[0]これらのガードは、次のようにプラグ可能に拡張できる可能性があります。
さまざまな構成に対するさまざまな懸念を表現するミドルウェア
周囲。
