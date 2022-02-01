# ADR 043:Blockhchain Reactor Riri-Org

## 变更日志

- 18-06-2019:初稿
- 08-07-2019:审核
- 29-11-2019:实施
- 14-02-2020:更新了实施细节

## 语境

区块链反应器负责两个高级过程:发送/接收来自对等方的块和快速同步块以赶上远远落后的upnode. [ADR-40](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain-reactor-refactor.md) 的目标是通过分离业务来重构这两个流程当前包含在 go-channels 中的逻辑到纯 `handle*` 函数中.虽然 ADR 指定了反应堆的最终形式可能是什么样子，但它缺乏关于实现中间步骤的指导.
下图说明了 [blockchain-reorg](https://github.com/tendermint/tendermint/pull/3561) 反应器的状态，该反应器将被称为“v1”.

![v1 区块链反应器架构
图](https://github.com/tendermint/tendermint/blob/f9e556481654a24aeb689b​​dadaf5eab3ccd66829/docs/architecture/img/blockchain-reactor-v1.png)

虽然区块链反应器的“v1”在简化并发模型方面显示出显着改进，但当前的 PR 遇到了一些障碍.

- 当前公关大且难以审查.
- 块八卦和快速同步过程与共享的“池”数据结构高度耦合.
- 对等通信分布在多个组件上，创建复杂的依赖图，必须在测试期间模拟.
- 建模为有状态代码的超时在测试中引入了不确定性

此 ADR 旨在指定实现 [ADR-40] (https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain-reactor-refactor. MD).

## 决定

将区块链反应器的职责划分为一组专门与事件通信的组件.事件将包含时间戳，允许每个组件将时间作为内部状态进行跟踪.内部状态将被一组将产生事件的`handle*` 改变.组件之间的集成将发生在反应器中，然后反应器测试将成为组件之间的集成测试.这种设计将被称为“v2”.

![v2 区块链反应器架构
图](https://github.com/tendermint/tendermint/blob/584e67ac3fac220c5c3e0652e3582eca8231e8​​14/docs/architecture/img/blockchain-reactor-v2.png)

### 快速同步相关的通信渠道

下图显示了快速同步例程以及用于相互通信的通道和队列的类型.
此外，还显示了 sendRoutine 用于通过 Peer MConnection 发送消息的每个反应器通道.

![v2 区块链通道和队列
图](https://github.com/tendermint/tendermint/blob/5cf570690f989646fb3b615b734da503f038891f/docs/architecture/img/blockchain-v2-channels.png)

### Reactor 改动详解

反应器将包括一个多路分解程序，它将每个消息发送到每个子程序进行独立处理.然后每个子例程将选择它感兴趣的消息并调用 [ADR-40](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain) 中指定的句柄特定函数-reactor-refactor.md). demuxRoutine 充当“起搏器”，设置预期处理事件的时间.

```go
func demuxRoutine(msgs, scheduleMsgs, processorMsgs, ioMsgs) {
	timer := time.NewTicker(interval)
	for {
		select {
			case <-timer.C:
				now := evTimeCheck{time.Now()}
				schedulerMsgs <- now
				processorMsgs <- now
				ioMsgs <- now
			case msg:= <- msgs:
				msg.time = time.Now()
				// These channels should produce backpressure before
				// being full to avoid starving each other
				schedulerMsgs <- msg
				processorMsgs <- msg
				ioMesgs <- msg
				if msg == stop {
					break;
				}
		}
	}
}

func processRoutine(input chan Message, output chan Message) {
	processor := NewProcessor(..)
	for {
		msg := <- input
		switch msg := msg.(type) {
			case bcBlockRequestMessage:
				output <- processor.handleBlockRequest(msg))
			...
			case stop:
				processor.stop()
				break;
	}
}

func scheduleRoutine(input chan Message, output chan Message) {
	schelduer = NewScheduler(...)
	for {
		msg := <-msgs
		switch msg := input.(type) {
			case bcBlockResponseMessage:
				output <- scheduler.handleBlockResponse(msg)
			...
			case stop:
				schedule.stop()
				break;
		}
	}
}
```

## 生命周期管理

一组用于各个流程的例程允许流程与清晰的生命周期管理并行运行. 当前存在于反应器中的 `Start`、`Stop` 和 `AddPeer` 钩子将委托给子例程，允许它们独立管理内部状态，而无需进一步耦合到反应器.

```go
func (r *BlockChainReactor) Start() {
	r.msgs := make(chan Message, maxInFlight)
	schedulerMsgs := make(chan Message)
	processorMsgs := make(chan Message)
	ioMsgs := make(chan Message)

	go processorRoutine(processorMsgs, r.msgs)
	go scheduleRoutine(schedulerMsgs, r.msgs)
	go ioRoutine(ioMsgs, r.msgs)
	...
}

func (bcR *BlockchainReactor) Receive(...) {
	...
	r.msgs <- msg
	...
}

func (r *BlockchainReactor) Stop() {
	...
	r.msgs <- stop
	...
}

...
func (r *BlockchainReactor) Stop() {
	...
	r.msgs <- stop
	...
}
...

func (r *BlockchainReactor) AddPeer(peer p2p.Peer) {
	...
	r.msgs <- bcAddPeerEv{peer.ID}
	...
}

```

## IO 处理

反应器内的 io 处理例程将隔离对等通信. 通过 ioRoutine 的消息通常是一种方式，使用 `p2p` API. 在诸如“trySend”之类的“p2p”API 返回错误的情况下，ioRoutine 可以将这些消息汇集回 demuxRoutine 以分发给其他例程. 例如，来自 ioRoutine 的错误可以被调度程序消耗以通知更好的对等选择实现.

```go
func (r *BlockchainReacor) ioRoutine(ioMesgs chan Message, outMsgs chan Message) {
	...
	for {
		msg := <-ioMsgs
		switch msg := msg.(type) {
			case scBlockRequestMessage:
				queued := r.sendBlockRequestToPeer(...)
				if queued {
					outMsgs <- ioSendQueued{...}
				}
			case scStatusRequestMessage
				r.sendStatusRequestToPeer(...)
			case bcPeerError
				r.Swtich.StopPeerForError(msg.src)
				...
			...
			case bcFinished
				break;
		}
	}
}

```

### 处理器内部

处理器负责排序、验证和执行块. 处理器将维护一个内部光标“高度”，指的是最后一个处理的块. 当一组块无序到达时，处理器将检查它是否有处理下一个块所需的“高度+1”. 处理器还维护对等点到高度的映射“blockPeers”，以跟踪哪个对等点在“高度”处提供了块. `blockPeers` 可以在 `handleRemovePeer(...)` 中使用，以重新安排由出错的对等方提供的所有未处理的块.

```go
type Processor struct {
	height int64 // the height cursor
	state ...
	blocks [height]*Block	 // keep a set of blocks in memory until they are processed
	blockPeers [height]PeerID // keep track of which heights came from which peerID
	lastTouch timestamp
}

func (proc *Processor) handleBlockResponse(peerID, block) {
    if block.height <= height || block[block.height] {
	} else if blocks[block.height] {
		return errDuplicateBlock{}
	} else  {
		blocks[block.height] = block
	}

	if blocks[height] && blocks[height+1] {
		... = state.Validators.VerifyCommit(...)
		... = store.SaveBlock(...)
		state, err = blockExec.ApplyBlock(...)
		...
		if err == nil {
			delete blocks[height]
			height++
			lastTouch = msg.time
			return pcBlockProcessed{height-1}
		} else {
			... // Delete all unprocessed block from the peer
			return pcBlockProcessError{peerID, height}
		}
	}
}

func (proc *Processor) handleRemovePeer(peerID) {
	events = []
	// Delete all unprocessed blocks from peerID
	for i = height; i < len(blocks); i++ {
		if blockPeers[i] == peerID {
			events = append(events, pcBlockReschedule{height})

			delete block[height]
		}
	}
	return events
}

func handleTimeCheckEv(time) {
	if time - lastTouch > timeout {
		// Timeout the processor
		...
	}
}
```

## 日程

Schedule根据一些调度算法维护用于调度blockRequestMessages的内部状态. 日程表需要在以下方面保持状态:

- 每个块的状态 `blockState` 似乎达到 maxHeight 的高度
- 一组对等点及其对等状态`peerState`
- 哪些对等点有哪些块
- 从哪些对等方请求了哪些块

```go
type blockState int

const (
	blockStateNew = iota
	blockStatePending,
	blockStateReceived,
	blockStateProcessed
)

type schedule {
    // a list of blocks in which blockState
	blockStates        map[height]blockState

    // a map of which blocks are available from which peers
	blockPeers         map[height]map[p2p.ID]scPeer

    // a map of peerID to schedule specific peer struct `scPeer`
	peers              map[p2p.ID]scPeer

    // a map of heights to the peer we are waiting for a response from
	pending map[height]scPeer

	targetPending  int // the number of blocks we want in blockStatePending
	targetReceived int // the number of blocks we want in blockStateReceived

	peerTimeout        int
	peerMinSpeed       int
}

func (sc *schedule) numBlockInState(state blockState) uint32 {
	num := 0
	for i := sc.minHeight(); i <= sc.maxHeight(); i++ {
		if sc.blockState[i] == state {
			num++
		}
	}
	return num
}


func (sc *schedule) popSchedule(maxRequest int) []scBlockRequestMessage {
	// We only want to schedule requests such that we have less than sc.targetPending and sc.targetReceived
	// This ensures we don't saturate the network or flood the processor with unprocessed blocks
	todo := min(sc.targetPending - sc.numBlockInState(blockStatePending), sc.numBlockInState(blockStateReceived))
	events := []scBlockRequestMessage{}
	for i := sc.minHeight(); i < sc.maxMaxHeight(); i++ {
		if todo == 0 {
			break
		}
		if blockStates[i] == blockStateNew {
			peer = sc.selectPeer(blockPeers[i])
			sc.blockStates[i] = blockStatePending
			sc.pending[i] = peer
			events = append(events, scBlockRequestMessage{peerID: peer.peerID, height: i})
			todo--
		}
	}
	return events
}
...

type scPeer struct {
	peerID               p2p.ID
	numOustandingRequest int
	lastTouched          time.Time
	monitor              flow.Monitor
}

```

# 调度器

调度程序被配置为在飞行中维护目标`n`
消息并将使用来自`_blockResponseMessage`的反馈，
`_statusResponseMessage` 和 `_peerError` 产生最佳分配
在每个 `timeCheckEv` 的 scBlockRequestMessage.

```

func handleStatusResponse(peerID, height, time) {
	schedule.touchPeer(peerID, time)
	schedule.setPeerHeight(peerID, height)
}

func handleBlockResponseMessage(peerID, height, block, time) {
	schedule.touchPeer(peerID, time)
	schedule.markReceived(peerID, height, size(block))
}

func handleNoBlockResponseMessage(peerID, height, time) {
	schedule.touchPeer(peerID, time)
	// reschedule that block, punish peer...
    ...
}

func handlePeerError(peerID)  {
    // Remove the peer, reschedule the requests
    ...
}

func handleTimeCheckEv(time) {
	// clean peer list

    events = []
	for peerID := range schedule.peersNotTouchedSince(time) {
		pending = schedule.pendingFrom(peerID)
		schedule.setPeerState(peerID, timedout)
		schedule.resetBlocks(pending)
		events = append(events, peerTimeout{peerID})
    }

	events = append(events, schedule.popSchedule())

	return events
}
```

## 同行

Peer 根据调度程序接收到的消息存储每个对等状态.

```go
type Peer struct {
	lastTouched timestamp
	lastDownloaded timestamp
	pending map[height]struct{}
	height height // max height for the peer
	state {
		pending,   // we know the peer but not the height
		active,    // we know the height
		timeout    // the peer has timed out
	}
}
```

## 状态

实施的

## 结果

### 积极的

- 测试变得确定性
- 模拟变成临时:无需等待挂墙时间超时
- 对等选择可以独立测试/模拟
- 开发重构反应堆的通用方法

### 消极的

### 中性的

### 实现路径

- 实施调度器，测试调度器，审查重新调度器
- 实施处理器，测试处理器，审查处理器
- 实施分路器，编写集成测试，审查集成测试

## 参考

- [ADR-40](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain-reactor-refactor.md):原始区块链反应堆重组提案
- [Blockchain re-org](https://github.com/tendermint/tendermint/pull/3561):当前区块链反应堆重组实现(v1)
