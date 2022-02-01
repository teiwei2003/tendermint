# ADR 012:PeerTransport

## 语境

p2p 中当前架构比较明显的问题之一
包是不同的之间没有明确的关注点分离
组件.最值得注意的是，“Switch”目前正在进行物理连接
处理.一个工件是 Switch on 的依赖
`[config.P2PConfig`](https://github.com/tendermint/tendermint/blob/05a76fb517f50da27b4bfcdc7b4cf185fc61eff6/config/config.go#L272-L339).

地址:

- [#2046](https://github.com/tendermint/tendermint/issues/2046)
- [#2047](https://github.com/tendermint/tendermint/issues/2047)

[#2067](https://github.com/tendermint/tendermint/issues/2067) 中的第一次迭代

## 决定

传输问题将由一个新组件(`PeerTransport`)处理，该组件
将在其边界处向调用者提供 Peers.反过来`Switch`将使用
这个新组件接受新的“Peer”并根据“NetAddress”拨号.

### PeerTransport

负责发射和连接到 Peers. `Peer`的实现
留给传输，这意味着选择的传输决定了
实现的特性交还给“Switch”.每个
传输实现负责过滤建立对等体特定的
到它的域，对于默认的多路复用实现，以下将
申请:

- 来自我们自己节点的连接
- 握手失败
- 升级到秘密连接失败
- 防止重复ip
- 防止重复ID
- nodeinfo 不兼容

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

### 转变

从现在开始，Switch 将依赖于完全设置的“PeerTransport”
检索/联系其同行. 随着更多的低级关注被推到
在传输过程中，我们可以省略将 `config.P2PConfig` 传递给 Switch.

```go
func NewSwitch(transport PeerTransport, opts ...SwitchOption) *Switch
```

## 状态

在审查.

## 结果

### 积极的

- 从传输问题中免费切换 - 更简单的实现
- 可插拔传输实现 - 更简单的测试设置
- 移除 Switch 对 P2PConfig 的依赖 - 更容易测试

### 消极的

- 对依赖于 Switch 的测试的更多设置

### 中性的

- 多路复用将是默认实现

[0] 这些守卫可以潜在地扩展为可插拔，就像
中间件来表达不同配置所需的不同关注点
环境.
