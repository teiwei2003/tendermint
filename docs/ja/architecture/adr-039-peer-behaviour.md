# ADR 039:ピアツーピア動作インターフェース

## 変更ログ
* 07-03-2019:最初のドラフト
* 14-03-2019:フィードバックの更新

## 環境

ピアの行動に信号を送り、行動するための単一責任の欠如
独自のコンポーネントであり、ネットワークスタック[<sup> 1 </ sup>](#references)と緊密に結合されています.原子炉
彼らが呼び出していた「p2p.Switch」への参照を維持する
ピアが誤動作しているときの `switch.StopPeerForError(...)`
`switch.MarkAsGood(...)`ピアが意味のある方法で貢献したとき.
スイッチは内部で `StopPeerForError`を処理しますが、` MarkAsGood`
メソッドは別のコンポーネント `p2p.AddrBook`に委任されます.この委員会
クロススイッチは、ピアツーピアの動作を処理する責任を覆い隠します
また、テスト中にリアクターをより大きな依存関係グラフにバンドルします.

## 決定

「PeerBehaviour」インターフェースと特定の実装を紹介します
原子炉が直接なしでピアツーピアの振る舞いを通知する方法を提供する
`p2p.Switch`をカップルします.提供するErrorBehaviourPeerを導入します
ピアをブロックする具体的な理由. GoodBehaviourPeerを導入して提供する
ピアが貢献する特定の方法.

### 変更を実装する

PeerBehaviourは、ピアエラー信号を送信するためのインターフェイスにもなります.
ピアを「良い」とマークすることに関して.

```go
type PeerBehaviour interface {
    Behaved(peer Peer, reason GoodBehaviourPeer)
    Errored(peer Peer, reason ErrorBehaviourPeer)
}
```

何らかの理由で停止するようにピアに通知する代わりに:
`理由インターフェース{}`

特定のエラータイプErrorBehaviourPeerを紹介します.
```go
type ErrorBehaviourPeer int

const (
    ErrorBehaviourUnknown = iota
    ErrorBehaviourBadMessage
    ErrorBehaviourMessageOutofOrder
    ...
)
```

ピアがどのように貢献するかについての詳細を提供するために、
GoodBehaviourPeerタイプ.

```go
type GoodBehaviourPeer int

const (
    GoodBehaviourVote = iota
    GoodBehaviourBlockPart
    ...
)
```

最初の反復として、ラップする具体的な実装を提供します
スイッチ:
```go
type SwitchedPeerBehaviour struct {
    sw *Switch
}

func (spb *SwitchedPeerBehaviour) Errored(peer Peer, reason ErrorBehaviourPeer) {
    spb.sw.StopPeerForError(peer, reason)
}

func (spb *SwitchedPeerBehaviour) Behaved(peer Peer, reason GoodBehaviourPeer) {
    spb.sw.MarkPeerAsGood(peer)
}

func NewSwitchedPeerBehaviour(sw *Switch) *SwitchedPeerBehaviour {
    return &SwitchedPeerBehaviour{
        sw: sw,
    }
}
```

通常、リアクターの単体テストは困難です[<sup> 2 </ sup>](#references)リアクターによって生成された信号を
製造シーン:

```go
type ErrorBehaviours map[Peer][]ErrorBehaviourPeer
type GoodBehaviours map[Peer][]GoodBehaviourPeer

type StorePeerBehaviour struct {
    eb ErrorBehaviours
    gb GoodBehaviours
}

func NewStorePeerBehaviour() *StorePeerBehaviour{
    return &StorePeerBehaviour{
        eb: make(ErrorBehaviours),
        gb: make(GoodBehaviours),
    }
}

func (spb StorePeerBehaviour) Errored(peer Peer, reason ErrorBehaviourPeer) {
    if _, ok := spb.eb[peer]; !ok {
        spb.eb[peer] = []ErrorBehaviours{reason}
    } else {
        spb.eb[peer] = append(spb.eb[peer], reason)
    }
}

func (mpb *StorePeerBehaviour) GetErrored() ErrorBehaviours {
    return mpb.eb
}


func (spb StorePeerBehaviour) Behaved(peer Peer, reason GoodBehaviourPeer) {
    if _, ok := spb.gb[peer]; !ok {
        spb.gb[peer] = []GoodBehaviourPeer{reason}
    } else {
        spb.gb[peer] = append(spb.gb[peer], reason)
    }
}

func (spb *StorePeerBehaviour) GetBehaved() GoodBehaviours {
    return spb.gb
}
```

## ステータス

受け入れられました

## 結果

### ポジティブ

      *信号を同等の動作の動作から分離します.
      *リアクトルとスイッチおよびネットワーク間の結合を減らす
        ヒープ
      *ピアの動作を管理する責任はに移すことができます
        スイッチとスイッチを分割するのではなく、個々のコンポーネント
        住所録.

### ネガティブ

      *最初の反復では、スイッチをラップして、
        間接レベル.

### ニュートラル

## 参照する

1.問題[#2067](https://github.com/tendermint/tendermint/issues/2067):P2Pリファクタリング
2. PR:[#3506](https://github.com/tendermint/tendermint/pull/3506):ADR 036:ブロックチェーンリアクターの再構築
