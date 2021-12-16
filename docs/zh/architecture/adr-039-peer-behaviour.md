# ADR 039:对等行为接口

## 变更日志
* 07-03-2019:初稿
* 14-03-2019:反馈更新

## 语境

对同伴行为发出信号并采取行动的责任缺乏单一的
拥有组件并与网络堆栈[<sup>1</sup>](#references) 紧密耦合。反应堆
维护对他们用来调用的“p2p.Switch”的引用
`switch.StopPeerForError(...)` 当对等端行为不端时
`switch.MarkAsGood(...)` 当对等方以某种有意义的方式做出贡献时。
虽然开关在内部处理 `StopPeerForError`，`MarkAsGood`
方法委托给另一个组件，`p2p.AddrBook`。这个委托方案
跨 Switch 掩盖了处理对等行为的责任
并在测试时将反应器捆绑在更大的依赖图中。

## 决定

引入“PeerBehaviour”接口和具体实现
为反应器提供方法来在没有直接的情况下向对等行为发出信号
耦合`p2p.Switch`。引入一个 ErrorBehaviourPeer 来提供
阻止同行的具体原因。引入 GoodBehaviourPeer 提供
同行做出贡献的具体方式。

### 实施变更

PeerBehaviour 然后也成为用于发送对等错误信号的接口
至于将同行标记为“好”。

```go
type PeerBehaviour interface {
    Behaved(peer Peer, reason GoodBehaviourPeer)
    Errored(peer Peer, reason ErrorBehaviourPeer)
}
```

而不是以任意原因通知对等点停止:
`原因接口{}`

我们引入一个具体的错误类型 ErrorBehaviourPeer:
```go
type ErrorBehaviourPeer int

const (
    ErrorBehaviourUnknown = iota
    ErrorBehaviourBadMessage
    ErrorBehaviourMessageOutofOrder
    ...
)
```

为了提供有关对等方贡献方式的更多信息，我们引入
GoodBehaviourPeer 类型。

```go
type GoodBehaviourPeer int

const (
    GoodBehaviourVote = iota
    GoodBehaviourBlockPart
    ...
)
```

作为第一次迭代，我们提供了一个具体的实现，它包装了
开关:
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

Reactor 通常难以进行单元测试[<sup>2</sup>](#references) 可以使用一种实现，该实现将反应器产生的信号暴露在
制造场景:

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

## 状态

公认

## 结果

### 积极的

     * 将信号与对等行为的行为分离开来。
     * 减少电抗器与交换机与网络的耦合
       堆
     * 管理同伴行为的责任可以迁移到
       单个组件，而不是在开关和开关之间拆分
       地址簿。

### 消极的

     * 第一次迭代将简单地包装 Switch 并引入一个
       间接级别。

### 中性的

## 参考

1.问题[#2067](https://github.com/tendermint/tendermint/issues/2067):P2P重构
2. PR:[#3506](https://github.com/tendermint/tendermint/pull/3506):ADR 036:区块链反应器重构
