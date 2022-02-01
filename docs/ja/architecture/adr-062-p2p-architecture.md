# ADR 062:P2Pアーキテクチャと抽象化

## 変更ログ

-2020-11-09:初期バージョン(@erikgrinaker)

-2020年11月13日:ストリームIDを削除し、ピアエラーをチャネルに移動し、PEXをコアに移動することに注意してください(@erikgrinaker)

-2020-11-16:推奨される原子炉実施モードの説明、ADRの承認(@erikgrinaker)

-2021年2月4日:新しいP2PコアとトランスポートAPIの変更で更新されました(@erikgrinaker).

## 環境

[ADR 061](adr-061-p2p-refactor-scope.md)では、ピアツーピア(P2P)ネットワークスタックをリファクタリングすることにしました.最初の段階は、プロトコルの互換性を可能な限り維持しながら、内部P2Pアーキテクチャを再設計および再構築することです.

## 代替方法

たとえば、メッセージを渡す代わりにインターフェイスメソッドを呼び出す(現在のアーキテクチャのように)、チャネルとストリームをマージする、内部のピアツーピアデータ構造をリアクタに公開する、任意のパスを渡すなど、提案された設計のいくつかのバリエーションが検討されました.コーデックとメッセージフォーマット何もありません、待ってください.この設計が選択されたのは、結合が非常に緩く、推論が簡単で使いやすく、競合状態や内部データ構造のロック競合を回避し、reactorがメッセージの順序付けと処理のセマンティクスをより適切に制御できるようにし、QoSディスパッチを可能にするためです.そして非常に自然な方法で背圧.

[multiaddr](https://github.com/multiformats/multiaddr)は、通常のURLを介した送信とは関係のないピアツーピアアドレス形式と見なされますが、広く採用されているようには見えず、プロトコルのカプセル化やトンネリングなどの高度な機能は、すぐには役に立たないようです.

独自のP2Pスタックを維持する代わりにLibP2Pを使用するためのいくつかの提案もあります.これらの提案は、[ADR 061](adr-061-p2p-refactor-scope.md)で(現在)拒否されています.

このADRの初期バージョンにはバイト指向のマルチストリーム送信APIがありますが、既存のメッセージ指向のMConnectionプロトコルとの下位互換性を維持するために、放棄/延期する必要があります.詳細については、[tendermint/spec#227](https://github.com/tendermint/spec/pull/227)で拒否されたRFCを参照してください.

## 決定

P2Pスタックは、メッセージ指向アーキテクチャとして再設計され、主に通信とスケジューリングをGoチャネルに依存します.単一のピア、双方向のピアツーピアアドレス可能チャネルを使用したバイナリメッセージへのメッセージ指向の送信を使用して、Protobufメッセージ、リアクターとピア間でメッセージをルーティングするルーターを送受信し、定期的な情報のピアライフピアマネージャーを管理します.メッセージ配信は、最大1回の配信と非同期です.

## 詳細設計

ADRは、実装の詳細ではなく、主にP2Pスタックのアーキテクチャとインターフェイスに焦点を当てています.したがって、ここで説明するインターフェイスは、完全な最終設計ではなく、大まかなアーキテクチャの概要と見なす必要があります.

主な設計目標は次のとおりです.

*コンポーネント間の結合を緩めて、よりシンプルで堅牢でテストに適したアーキテクチャを実現します.
*プラガブル伝送(必ずしもネットワーク化されている必要はありません).
*より良いメッセージスケジューリング、改善された優先度、バックプレッシャーおよびパフォーマンス.
*一元化されたピアツーピアのライフサイクルと接続管理.
*ピアアドレスの検出、アドバタイズ、交換が改善されました.
*障害物が多すぎることが判明しない限り、現在のP2Pネットワークプロトコルとの回線レベルの下位互換性.

新しいスタックの主な抽象化は次のとおりです.

* `Transport`:` Connection`を介してピアとバイナリメッセージを交換するための任意のメカニズム.
* `Channel`:ノードIDを使用して、ピアとのProtobufメッセージの非同期交換用の双方向チャネルをアドレス指定します.
* `Router`:関連するピアおよびルートチャネルメッセージとの送信接続を維持します.
* `PeerManager`:ストレージに「peerStore」を使用して、呼び出すピアや呼び出すタイミングの決定など、ピアのライフサイクル情報を管理します.
* Reactor:「チャネルをリッスンしてメッセージに反応するもの」として大まかに定義されたデザインパターン.

これらの抽象化は、以下の図(ノードAの内部構造を表す)に示され、以下で詳細に説明されています.

！[P2Pアーキテクチャ図](img/adr-062-architecture.svg)

### 交通

トランスポートは、バイナリメッセージをピアと交換するために使用されるメカニズムです.たとえば、gRPC送信はTCP/IPを介してピアに接続し、gRPCプロトコルを使用してデータを送信しますが、メモリ内送信は内部Goチャネルを使用して別のゴルーチンで実行されているピアと通信する場合があります.トランスポート自体には「ピア」または「ノード」の概念がないことに注意してください.代わりに、任意のエンドポイントアドレス(IPアドレスやポート番号など)間の接続を確立して、残りのP2Pスタックから分離します.

輸送は次の要件を満たしている必要があります.

*コネクション型で、インバウンド接続の監視とエンドポイントアドレスを使用したアウトバウンド接続の確立をサポートします.

*異なるチャネルIDでのバイナリメッセージの送信をサポートします(チャネルとチャネルIDは、ルーターのセクションで説明されている高レベルのアプリケーションプロトコルの概念ですが、トランスポート層を介してスレッド化され、既存のMConnectionプロトコルと下位互換性があります).

*ノードハンドシェイクを介してMConnection`NodeInfo`と公開鍵を交換し、トラフィックを適切に暗号化または署名する場合があります.

最初の送信は、Tendermintによって現在使用されている現在のMConnectionプロトコルのポートであり、回線レベルで下位互換性がある必要があります.テスト用のメモリ内送信も実装されています. MConnectionプロトコルに取って代わる可能性のあるQUIC送信を調査する計画があります.

`Transport`のインターフェースは次のとおりです.

```go
// Transport is a connection-oriented mechanism for exchanging data with a peer.
type Transport interface {
    // Protocols returns the protocols supported by the transport. The Router
    // uses this to pick a transport for an Endpoint.
    Protocols() []Protocol

    // Endpoints returns the local endpoints the transport is listening on, if any.
    // How to listen is transport-dependent, e.g. MConnTransport uses Listen() while
    // MemoryTransport starts listening via MemoryNetwork.CreateTransport().
    Endpoints() []Endpoint

    // Accept waits for the next inbound connection on a listening endpoint, blocking
    // until either a connection is available or the transport is closed. On closure,
    // io.EOF is returned and further Accept calls are futile.
    Accept() (Connection, error)

    // Dial creates an outbound connection to an endpoint.
    Dial(context.Context, Endpoint) (Connection, error)

    // Close stops accepting new connections, but does not close active connections.
    Close() error
}
```

送信がリッスンするように構成されている方法は、送信によって異なり、インターフェイスには含まれていません. これは通常、トランスポートの構築中に発生します.トランスポートインスタンスが作成され、ルーターに渡される前に適切なネットワークインターフェイスでリッスンするように設定されます.

#### 終点

「エンドポイント」は、送信エンドポイント(IPアドレスやポートなど)を示します. 接続には常に2つのエンドポイントがあります.1つはローカルノードに、もう1つはリモートノードにあります. リモートエンドポイントへのアウトバウンド接続は `Dial()`を介して確立され、リスニングエンドポイントへのインバウンド接続は `Accept()`を介して返されます.

`Endpoint`構造は次のとおりです.

```go
// Endpoint represents a transport connection endpoint, either local or remote.
//
// Endpoints are not necessarily networked (see e.g. MemoryTransport) but all
// networked endpoints must use IP as the underlying transport protocol to allow
// e.g. IP address filtering. Either IP or Path (or both) must be set.
type Endpoint struct {
    // Protocol specifies the transport protocol.
    Protocol Protocol

    // IP is an IP address (v4 or v6) to connect to. If set, this defines the
    // endpoint as a networked endpoint.
    IP net.IP

    // Port is a network port (either TCP or UDP). If 0, a default port may be
    // used depending on the protocol.
    Port uint16

    // Path is an optional transport-specific path or identifier.
    Path string
}

// Protocol identifies a transport protocol.
type Protocol string
```

エンドポイントは任意の送信固有のアドレスですが、ネットワークに接続されている場合はIPアドレスを使用する必要があるため、基本的なパケットルーティングプロトコルとしてIPに依存しています.これにより、アドレスの検出、アドバタイズ、および交換の戦略が可能になります.たとえば、プライベート「192.168.0.0/24」IPアドレスは、そのIPネットワーク上のピアにのみアドバタイズする必要がありますが、パブリックアドレス「8.8.8.8」はすべての同僚にアドバタイズできます. .同様に、NATゲートウェイなどの自動構成に[UPnP](https://en.wikipedia.org/wiki/Universal_Plug_and_Play)を使用するには、任意のポート番号がTCPおよび/またはUDPポート番号を表す必要があります.

ネットワークに接続されていないエンドポイント(IPアドレスなし)はローカルと見なされ、同じプロトコルを介して接続されている他のピアにのみアドバタイズされます.たとえば、テストに使用されるメモリ転送では、ノード「foo」のアドレスとして「Endpoint {Protocol: "memory"、Path: "foo"}」が使用されます.これは、「Protocol:」を使用して他のノードにのみアドバタイズする必要があります.メモリ "`.

#### 接続

接続は、2つのエンドポイント(つまり、2つのノード)間で確立された伝送接続を表し、論理チャネルID(ルーターで使用される上位レベルのチャネルIDに対応)とバイナリメッセージを交換するために使用できます. `Transport.Dial()`(アウトバウンド)または `Transport.Accept()`(インバウンド)を介して接続を確立します.

接続が確立されたら、「Transport.Handshake()」を呼び出してノードハンドシェイクを実行し、ノード情報と公開鍵を交換してノードのIDを確認する必要があります.ノードハンドシェイクは、実際にはトランスポート層の一部であってはなりません(これはアプリケーションプロトコルの問題です).これは、既存のMConnectionプロトコルとの下位互換性のためであり、2つを混同します. `NodeInfo`は既存のMConnectionプロトコルの一部ですが、仕様に記載されていないようです.詳細については、Goコードベースを参照してください.

`Connection`インターフェースを以下に示します.レガシーP2Pスタックとの下位互換性のために現在実装されている特定の追加を省略し、最終バージョンの前にそれらを削除する予定です.

```go
// Connection represents an established connection between two endpoints.
type Connection interface {
    // Handshake executes a node handshake with the remote peer. It must be
    // called once the connection is established, and returns the remote peer's
    // node info and public key. The caller is responsible for validation.
    Handshake(context.Context, NodeInfo, crypto.PrivKey) (NodeInfo, crypto.PubKey, error)

    // ReceiveMessage returns the next message received on the connection,
    // blocking until one is available. Returns io.EOF if closed.
    ReceiveMessage() (ChannelID, []byte, error)

    // SendMessage sends a message on the connection. Returns io.EOF if closed.
    SendMessage(ChannelID, []byte) error

    // LocalEndpoint returns the local endpoint for the connection.
    LocalEndpoint() Endpoint

    // RemoteEndpoint returns the remote endpoint for the connection.
    RemoteEndpoint() Endpoint

    // Close closes the connection.
    Close() error
}
```

このADRは当初、バイト指向のマルチストリーム接続APIを提案しました.これは、より一般的なネットワークAPI規則に従います(たとえば、他のライブラリと簡単に組み合わせることができる `io.Reader`および` io.Writer`インターフェイスを使用します).これにより、メッセージフレーム、ノードハンドシェイク、およびトラフィックスケジューリングの責任を、送信間で再実装するのではなく、パブリックルーターに転送し、QUICなどのマルチストリームプロトコルをより適切に使用できるようになります.ただし、これには、拒否されたMConnectionプロトコルにマイナーおよびメジャーの変更を加える必要があります.詳細については、[tendermint/spec#227](https://github.com/tendermint/spec/pull/227)を参照してください. QUICトランスポートの作業を開始するときは、これを再検討する必要があります.

### ピア管理

ピアは他のTendermintノードです.各ピアは、(ノードの秘密鍵に関連付けられた)一意の「NodeID」によって識別されます.

```go
// NodeID is a hex-encoded crypto.Address. It must be lowercased
// (for uniqueness) and of length 40.
type NodeID string

// NodeAddress is a node address URL. It differs from a transport Endpoint in
// that it contains the node's ID, and that the address hostname may be resolved
// into multiple IP addresses (and thus multiple endpoints).
//
// If the URL is opaque, i.e. of the form "scheme:opaque", then the opaque part
// is expected to contain a node ID.
type NodeAddress struct {
    NodeID   NodeID
    Protocol Protocol
    Hostname string
    Port     uint16
    Path     string
}

// ParseNodeAddress parses a node address URL into a NodeAddress, normalizing
// and validating it.
func ParseNodeAddress(urlString string) (NodeAddress, error)

// Resolve resolves a NodeAddress into a set of Endpoints, e.g. by expanding
// out a DNS hostname to IP addresses.
func (a NodeAddress) Resolve(ctx context.Context) ([]Endpoint, error)
```

#### ピアマネージャー

P2Pスタックは、アドレス、接続ステータス、優先度、可用性、障害、再試行など、ピアに関する多くの内部ステータスを追跡する必要があります.この責任は、「ルーター」のこの状態を追跡する「PeerManager」に分離されています(ただし、ルーターの責任である実際の伝送接続自体は維持されません).

`PeerManager`は同期ステートマシンであり、すべての状態遷移がシリアル化されます(同期メソッド呼び出しとして実装され、排他的なミューテックスロックを保持します).ピア状態のほとんどは意図的に内部に保持され、適切に永続化する「peerStore」データベースに保存されます.外部インターフェイスは、ルーターのゴルーチン間で状態を共有しないようにするために必要な最小限の情報を送信します.この設計はモデルを大幅に簡素化し、P2Pネットワークのコアにモデルを配置するよりも推論とテストが容易であり、非同期で並行している必要があります.ピアツーピアのライフサイクルイベントは比較的少ないと予想されるため、これがパフォーマンスに大きな影響を与えることはありません.

`Router`は` PeerManager`を使用して、どのピアにダイヤルして削除するかを要求し、接続、切断、障害などのピアライフサイクルイベントを報告します.マネージャは、エラーを返すことでこれらのイベントを拒否できます(たとえば、インバウンド接続を拒否します).これは次のように発生します.

* `Transport.Dial`を介したアウトバウンド接続:
    * `DialNext()`:ダイヤル用のピアアドレスを返すか、使用可能になるまでブロックします.
    * `DialFailed()`:反対側でのダイヤルの失敗を報告します.
    * `Dialed()`:ピアが正常にダイヤルしたことを報告します.
    * `Ready()`:ピアルーティングと準備状況を報告します.
    * `Disconnected()`:ピアが切断されたことを報告します.

* `Transport.Accept`を介したインバウンド接続:
    * `Accepted()`:インバウンドピアツーピア接続を報告します.
    * `Ready()`:ピアルーティングと準備状況を報告します.
    * `Disconnected()`:ピアが切断されたことを報告します.

*追放するには、 `Connection.Close`を使用します.
    * `EvictNext()`:切断するピアを返すか、使用可能になるまでブロックします.
    * `Disconnected()`:ピアが切断されたことを報告します.

これらの呼び出しには、次のインターフェイスがあります.

```go
// DialNext returns a peer address to dial, blocking until one is available.
func (m *PeerManager) DialNext(ctx context.Context) (NodeAddress, error)

// DialFailed reports a dial failure for the given address.
func (m *PeerManager) DialFailed(address NodeAddress) error

// Dialed reports a successful outbound connection to the given address.
func (m *PeerManager) Dialed(address NodeAddress) error

// Accepted reports a successful inbound connection from the given node.
func (m *PeerManager) Accepted(peerID NodeID) error

// Ready reports the peer as fully routed and ready for use.
func (m *PeerManager) Ready(peerID NodeID) error

// EvictNext returns a peer ID to disconnect, blocking until one is available.
func (m *PeerManager) EvictNext(ctx context.Context) (NodeID, error)

// Disconnected reports a peer disconnection.
func (m *PeerManager) Disconnected(peerID NodeID) error
```

内部的には、「PeerManager」は数値のピアスコアを使用して、ピアノードの優先度を決定します.たとえば、次に呼び出すピアを決定する場合などです. スコアリングポリシーはまだ実装されていませんが、たとえば、「persistent_peers」などのノード構成、稼働時間と接続の障害、パフォーマンスなどを考慮に入れる必要があります. より適切なノードが使用可能な場合(たとえば、中断後に永続ノードがオンラインに戻った場合)、マネージャーは、スコアの低いノードを削除することにより、より高いノードに自動的にアップグレードしようとします.

`PeerManager`には、スコアに影響を与えるリアクターからのピアの動作を報告するAPIも必要です(たとえば、ブロックに署名するとスコアが上がり、二重投票するとスコアが下がり、ピアリングが禁止されます)が、これは設計されておらず、実装されました.

さらに、 `PeerManager`は` PeerUpdates`サブスクリプションを提供します.これは、ピアステータスが大幅に変更されるたびに `PeerUpdate`イベントを受信します. リアクタは、これらを使用して、たとえば、ピアが接続または切断されたことを認識し、適切な対策を講じることができます. これは現在非常に小さいです:

```go
// Subscribe subscribes to peer updates. The caller must consume the peer updates
// in a timely fashion and close the subscription when done, to avoid stalling the
// PeerManager as delivery is semi-synchronous, guaranteed, and ordered.
func (m *PeerManager) Subscribe() *PeerUpdates

// PeerUpdate is a peer update event sent via PeerUpdates.
type PeerUpdate struct {
    NodeID NodeID
    Status PeerStatus
}

// PeerStatus is a peer status.
type PeerStatus string

const (
    PeerStatusUp   PeerStatus = "up"   // Connected and ready.
    PeerStatusDown PeerStatus = "down" // Disconnected.
)

// PeerUpdates is a real-time peer update subscription.
type PeerUpdates struct { ... }

// Updates returns a channel for consuming peer updates.
func (pu *PeerUpdates) Updates() <-chan PeerUpdate

// Close closes the peer updates subscription.
func (pu *PeerUpdates) Close()
```

`PeerManager`は、他のノードによってゴシップされる可能性のあるPEXリアクターにピアツーピア情報を提供する役割も果たします. これには、ピアアドレスとセルフアドレスの信頼性の高い検出や、同じネットワーク上の他のピアにのみプライベートアドレスを送信するなど、改善されたピアアドレス検出およびアドバタイズシステムが必要ですが、このシステムはまだ完全に設計および実装されていません.

### チャネル

低レベルのデータ交換は「送信」を通じて行われますが、高レベルのAPIは、「NodeID」によってアドレス指定されたProtobufメッセージを送受信できる双方向の「チャネル」に基づいています. チャネルは任意の「ChannelID」識別子によって識別され、特定のタイプのProtobufメッセージを交換できます(マーシャリングされないタイプは事前定義されている必要があるため). メッセージ配信は非同期で、多くても1回です.

このチャネルは、無効な情報や悪意のある情報を受信した場合など、ピアツーピアエラーを報告するためにも使用できます. 「PeerManager」ポリシーによると、これによりピアが切断または禁止される可能性がありますが、より広範なピア動作APIに置き換える必要があります.これにより、良好な動作も報告される可能性があります.

`Channel`には次のインターフェースがあります.

```go
// ChannelID is an arbitrary channel ID.
type ChannelID uint16

// Channel is a bidirectional channel to exchange Protobuf messages with peers.
type Channel struct {
    ID          ChannelID        // Channel ID.
    In          <-chan Envelope  // Inbound messages (peers to reactors).
    Out         chan<- Envelope  // outbound messages (reactors to peers)
    Error       chan<- PeerError // Peer error reporting.
    messageType proto.Message    // Channel's message type, for e.g. unmarshaling.
}

// Close closes the channel, also closing Out and Error.
func (c *Channel) Close() error

// Envelope specifies the message receiver and sender.
type Envelope struct {
    From      NodeID        // Sender (empty if outbound).
    To        NodeID        // Receiver (empty if inbound).
    Broadcast bool          // Send to all connected peers, ignoring To.
    Message   proto.Message // Message payload.
}

// PeerError is a peer error reported via the Error channel.
type PeerError struct {
    NodeID   NodeID
    Err      error
}
```

チャネルは接続されている任意のピアに到達でき、Protobufメッセージを自動的に(キャンセル)マーシャルします. メッセージのスケジューリングとキューイングは「ルーター」実装の問題であり、FIFO、ループ、優先キューなど、任意の数のアルゴリズムを使用できます. メッセージの配信は保証されないため、必要に応じて、受信メッセージと送信メッセージの両方が破棄、バッファリング、並べ替え、またはブロックされる場合があります.

チャネルは単一のタイプのメッセージしか交換できないため、含めることができる内部メッセージタイプのセットを指定するProtobuf`oneof`フィールドなどのラッパーメッセージタイプを使用すると便利なことがよくあります. 外部メッセージタイプが `Wrapper`インターフェースを実装している場合([Reactor Example](#reactor-example)の例を参照)、チャネルはこの(アン)ラッピングを自動的に実行できます.

```go
// Wrapper is a Protobuf message that can contain a variety of inner messages.
// If a Channel's message type implements Wrapper, the channel will
// automatically (un)wrap passed messages using the container type, such that
// the channel can transparently support multiple message types.
type Wrapper interface {
    proto.Message

    // Wrap will take a message and wrap it in this one.
    Wrap(proto.Message) error

    // Unwrap will unwrap the inner message contained in this message.
    Unwrap() (proto.Message, error)
}
```

### ルーター

ルーターはノードのP2Pネットワークを実装し、「PeerManager」から命令を取得して「PeerManager」にイベントを報告し、ピアとの伝送接続を維持し、チャネルとピアの間でメッセージをルーティングします.

実際、P2Pスタック内のすべての同時実行性はルーターとリアクターに移動され、他の多くの責任は「Transport」や「PeerManager」などの個別のコンポーネントに移動されました.これらのコンポーネントは大幅に同期を保つことができます. ". 同時実行を単一のコアコンポーネントに制限すると、同時実行構造が1つだけであり、残りのコンポーネントをシリアル化でき、単純で、テストが容易になるため、推論が容易になります.

`Router`のAPIは非常に小さいため、主に` PeerManager`イベントと `Transport`イベントによって駆動されます.

```go
// Router maintains peer transport connections and routes messages between
// peers and channels.
type Router struct {
    // Some details have been omitted below.

    logger          log.Logger
    options         RouterOptions
    nodeInfo        NodeInfo
    privKey         crypto.PrivKey
    peerManager     *PeerManager
    transports      []Transport

    peerMtx         sync.RWMutex
    peerQueues      map[NodeID]queue

    channelMtx      sync.RWMutex
    channelQueues   map[ChannelID]queue
}

// OpenChannel opens a new channel for the given message type. The caller must
// close the channel when done, before stopping the Router. messageType is the
// type of message passed through the channel.
func (r *Router) OpenChannel(id ChannelID, messageType proto.Message) (*Channel, error)

// Start starts the router, connecting to peers and routing messages.
func (r *Router) Start() error

// Stop stops the router, disconnecting from all peers and stopping message routing.
func (r *Router) Stop() error
```

すべてのGoチャネルは「ルーター」で送信され、リアクターはブロックされます(ルーターは信号チャネルを閉じることも選択します). メッセージのスケジューリング、優先順位付け、バックプレッシャ、およびロードシェディングの責任は、競合ポイント(つまり、すべてのピアから単一のチャネルへ、およびすべてのチャネルから単一のペアへ)に使用されるコア「キュー」インターフェイスに集中します. .待つ):

```go
// queue does QoS scheduling for Envelopes, enqueueing and dequeueing according
// to some policy. Queues are used at contention points, i.e.:
// - Receiving inbound messages to a single channel from all peers.
// - Sending outbound messages to a single peer from all channels.
type queue interface {
    // enqueue returns a channel for submitting envelopes.
    enqueue() chan<- Envelope

    // dequeue returns a channel ordered according to some queueing policy.
    dequeue() <-chan Envelope

    // close closes the queue. After this call enqueue() will block, so the
    // caller must select on closed() as well to avoid blocking forever. The
    // enqueue() and dequeue() channels will not be closed.
    close()

    // closed returns a channel that's closed when the scheduler is closed.
    closed() <-chan struct{}
}
```

現在の実装は「fifoQueue」です.これは、メッセージを受信した順序で配信し、メッセージが配信されるまでブロックする単純なバッファなしのロスレスキューです(つまり、Goチャネルです). ルーターには、より複雑なキューイング戦略が必要になりますが、これはまだ実装されていません.

内部の `Router`ゴルーチンの構造と設計は、` Router` GoDocで説明されており、参考のために以下が含まれています.

```go
// On startup, three main goroutines are spawned to maintain peer connections:
//
//   dialPeers(): in a loop, calls PeerManager.DialNext() to get the next peer
//   address to dial and spawns a goroutine that dials the peer, handshakes
//   with it, and begins to route messages if successful.
//
//   acceptPeers(): in a loop, waits for an inbound connection via
//   Transport.Accept() and spawns a goroutine that handshakes with it and
//   begins to route messages if successful.
//
//   evictPeers(): in a loop, calls PeerManager.EvictNext() to get the next
//   peer to evict, and disconnects it by closing its message queue.
//
// When a peer is connected, an outbound peer message queue is registered in
// peerQueues, and routePeer() is called to spawn off two additional goroutines:
//
//   sendPeer(): waits for an outbound message from the peerQueues queue,
//   marshals it, and passes it to the peer transport which delivers it.
//
//   receivePeer(): waits for an inbound message from the peer transport,
//   unmarshals it, and passes it to the appropriate inbound channel queue
//   in channelQueues.
//
// When a reactor opens a channel via OpenChannel, an inbound channel message
// queue is registered in channelQueues, and a channel goroutine is spawned:
//
//   routeChannel(): waits for an outbound message from the channel, looks
//   up the recipient peer's outbound message queue in peerQueues, and submits
//   the message to it.
//
// All channel sends in the router are blocking. It is the responsibility of the
// queue interface in peerQueues and channelQueues to prioritize and drop
// messages as appropriate during contention to prevent stalls and ensure good
// quality of service.
```

### リアクターの例

リアクターは現在のP2Pスタックのファーストクラスの概念ですが(つまり、明確な `p2p.Reactor`インターフェースがあります)、それらは新しいスタックの単なるデザインパターンであり、大まかに「チャネルでリッスンするもの」として定義されます. 、およびニュースに応答する」.

原子炉に形式的な制約があることはめったにないので、それらはさまざまな方法で達成することができます. 現在、このADRでの過度の仕様と範囲の広がりを回避するために推奨されるリアクター実装モデルはありません. ただし、プロトタイプの設計と原子炉モデルの開発は、「チャネル」インターフェースを使用して構築された原子炉が利便性、決定論的テスト、および信頼性の要件を満たすことができるように、実装プロセスのできるだけ早い段階で完了する必要があります.

以下は、関数として実装された単純なエコーリアクターの簡単な例です. リアクタは、次のProtobufメッセージを交換します.

```protobuf
message EchoMessage {
    oneof inner {
        PingMessage ping = 1;
        PongMessage pong = 2;
    }
}

message PingMessage {
    string content = 1;
}

message PongMessage {
    string content = 1;
}
```

`EchoMessage`に` Wrapper`インターフェースを実装すると、チャネルを介した `PingMessage`と` PongMessage`の透過的な送信が可能になり、自動的に(キャンセル) `EchoMessage`にラップされます.

```go
func (m *EchoMessage) Wrap(inner proto.Message) error {
    switch inner := inner.(type) {
    case *PingMessage:
        m.Inner = &EchoMessage_PingMessage{Ping: inner}
    case *PongMessage:
        m.Inner = &EchoMessage_PongMessage{Pong: inner}
    default:
        return fmt.Errorf("unknown message %T", inner)
    }
    return nil
}

func (m *EchoMessage) Unwrap() (proto.Message, error) {
    switch inner := m.Inner.(type) {
    case *EchoMessage_PingMessage:
        return inner.Ping, nil
    case *EchoMessage_PongMessage:
        return inner.Pong, nil
    default:
        return nil, fmt.Errorf("unknown message %T", inner)
    }
}
```

リアクター自体は、たとえば次のように実装されます.

```go
// RunEchoReactor wires up an echo reactor to a router and runs it.
func RunEchoReactor(router *p2p.Router, peerManager *p2p.PeerManager) error {
    channel, err := router.OpenChannel(1, &EchoMessage{})
    if err != nil {
        return err
    }
    defer channel.Close()
    peerUpdates := peerManager.Subscribe()
    defer peerUpdates.Close()

    return EchoReactor(context.Background(), channel, peerUpdates)
}

// EchoReactor provides an echo service, pinging all known peers until the given
// context is canceled.
func EchoReactor(ctx context.Context, channel *p2p.Channel, peerUpdates *p2p.PeerUpdates) error {
    ticker := time.NewTicker(5 * time.Second)
    defer ticker.Stop()

    for {
        select {
        // Send ping message to all known peers every 5 seconds.
        case <-ticker.C:
            channel.Out <- Envelope{
                Broadcast: true,
                Message:   &PingMessage{Content: "👋"},
            }

        // When we receive a message from a peer, either respond to ping, output
        // pong, or report peer error on unknown message type.
        case envelope := <-channel.In:
            switch msg := envelope.Message.(type) {
            case *PingMessage:
                channel.Out <- Envelope{
                    To:      envelope.From,
                    Message: &PongMessage{Content: msg.Content},
                }

            case *PongMessage:
                fmt.Printf("%q replied with %q\n", envelope.From, msg.Content)

            default:
                channel.Error <- PeerError{
                    PeerID: envelope.From,
                    Err:    fmt.Errorf("unexpected message %T", msg),
                }
            }

        // Output info about any peer status changes.
        case peerUpdate := <-peerUpdates:
            fmt.Printf("Peer %q changed status to %q", peerUpdate.PeerID, peerUpdate.Status)

        // Exit when context is canceled.
        case <-ctx.Done():
            return nil
        }
    }
}
```

## ステータス

部分的な実装([#5670](https://github.com/tendermint/tendermint/issues/5670))

## 結果

### ポジティブ

*結合を減らし、インターフェースを単純化すると、理解しやすく、信頼性が高くなり、テストが増えるはずです.

* Goチャネルを介して配信されるメッセージを使用して、バックプレッシャーとサービス品質のスケジューリングをより適切に制御します.

*ピアのライフサイクルと接続管理は単一のエンティティに一元化されているため、推論が容易になります.

*ノードアドレスの検出、通知、交換が改善されます.

*追加の送信(QUICなど)を実装して、既存のMConnプロトコルと並行して使用できます.

*可能であれば、初期バージョンではP2Pプロトコルが壊れることはありません.

### ネガティブ

*期待どおりに新しい設計を完全に実現するには、ある時点でP2Pプロトコルに大幅な変更が必要になる場合がありますが、最初の実装では必要ありません.

*既存のスタックを段階的に移行して下位互換性を維持することは、単にスタック全体を置き換えるよりも手間がかかります.

*実装が成熟するにつれて、P2P内部構造を徹底的に検査すると、一時的なパフォーマンスの低下やエラーが発生する可能性があります.

*「PeerManager」でピア管理情報を非表示にすると、設計を簡素化し、結合を減らし、競合状態とロックの競合を回避するためのトレードオフとして、特定の機能が妨げられたり、情報交換のための追加の意図的なインターフェイスが必要になる場合があります.

### ニュートラル

*ピア管理、メッセージスケジューリング、ピアおよびエンドポイントのアドバタイズなどの実装の詳細はまだ決定されていません.

## 参照する

* [ADR 061:P2Pリファクタリングスコープ](adr-061-p2p-refactor-scope.md)
* [#5670 p2p:内部リファクタリングとアーキテクチャの再設計](https://github.com/tendermint/tendermint/issues/5670)
