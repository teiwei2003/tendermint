# 状態の同期


状態の同期により、新しいノードは検出、取得、
そして、ステートマシンのスナップショットを復元します。詳細については、[状態同期ABCIセクション](https://docs.tendermint.com/master/spec/abci/abci.html#state-sync)を参照してください。

状態同期リアクトルには、2つの主な責任があります。

*ローカルABCIアプリケーションによって取得されたステートマシンスナップショットを、新しく追加されたノードに提供します
  インターネット。

*既存のスナップショットを見つけて、空のローカルアプリケーションのスナップショットブロックを取得します
  案内されます。

新しいノードをガイドするために使用される状態同期プロセスについては、リンクされたセクションで詳しく説明されています。
より多い。技術的にはreactorの一部ですが( `statesync/syncer.go`および関連コンポーネントを参照)、
このドキュメントでは、P2Pリアクターコンポーネントのみを取り上げます。

ABCIメソッドとデータ型の詳細については、[ABCIドキュメント](https://docs.tendermint.com/master/spec/abci/)を参照してください。

状態同期の構成方法に関する情報は、[ノードセクション](../../nodes/state-sync.md)にあります。

## 状態同期P2Pプロトコル

新しいノードが状態の同期を開始すると、遭遇したすべてのピアが
利用可能なスナップショット:

```go
type snapshotsRequestMessage struct{}
```

受信者は、ListSnapshotsを介してローカルABCIアプリケーションにクエリを実行し、メッセージを送信します
最後の10個のスナップショット(4 MBに制限)のそれぞれのスナップショットメタデータが含まれます。

```go
type snapshotsResponseMessage struct {
 Height   uint64
 Format   uint32
 Chunks   uint32
 Hash     []byte
 Metadata []byte
}
```

ステータス同期を実行しているノードは、次の方法でこれらのスナップショットをローカルABCIアプリケーションに提供します
`OfferSnapshot` ABCIは、どのピアにどのスナップショットが含まれているかを呼び出して追跡します。 スナップショット
受け入れられると、状態シンクロナイザーは適切なピアからスナップショットブロックを要求します。

```go
type chunkRequestMessage struct {
 Height uint64
 Format uint32
 Index  uint32
}
```

受信者は、「LoadSnapshotChunk」を介してローカルアプリケーションから要求されたチャンクをロードします。
そしてそれに応答します(16MBに制限されています):

```go
type chunkResponseMessage struct {
 Height  uint64
 Format  uint32
 Index   uint32
 Chunk   []byte
 Missing bool
}
```

ここで、「Missing」は、空であるためにブロックがピアで見つからないことを示すために使用されます
チャンクは有効な(可能性は低いですが)応答です。

返されたブロックは、スナップショットまで「ApplySnapshotChunk」を介してABCIアプリケーションに提供されます
復元されました。 ブロック応答が一定期間内に返されない場合、それは再要求されます、
異なるピアから来る可能性があります。

ABCIプロトコルの一部として、ABCIアプリケーションはピアの禁止を要求し、再取得をブロックできます。

状態の同期が進行中でない場合(つまり、通常の操作中)、一方的な応答メッセージ
捨てる。
