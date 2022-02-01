# ADR 043:Blockhchain Reactor Riri-Org

## 変更ログ

-18-06-2019:最初のドラフト
-08-07-2019:監査
-29-11-2019:実装
-14-02-2020:実装の詳細を更新

## 環境

ブロックチェーンリアクターは、ピアからのブロックの送受信と、ブロックをすばやく同期してアップノードのはるか後ろに追いつくという2つの高レベルのプロセスを担当します. [ADR-40](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain-reactor-refactor.md)目標は、ビジネスを分離することにより、これら2つをリファクタリングすることです.純粋な `handle *`関数へのgo-channelsに現在含まれているフローのロジック. ADRは、リアクターの最終的な形式がどのように見えるかを指定しますが、中間ステップの実装に関するガイダンスが不足しています.
次の図は、[blockchain-reorg](https://github.com/tendermint/tendermint/pull/3561)リアクターの状態を示しています.これは「v1」と呼ばれます.

！[v1ブロックチェーンリアクタアーキテクチャ
図)(https://github.com/tendermint/tendermint/blob/f9e556481654a24aeb689b​​ dadaf5eab3ccd66829/docs/architecture/img/blockchain-reactor-v1.png)

ブロックチェーンリアクターの「v1」は、並行性モデルを単純化する上で大幅な改善を示していますが、現在のPRにはいくつかの障害があります.

-現在の広報活動は大きく、レビューが困難です.
-ブロックゴシップと高速同期プロセスは、共有「プール」データ構造と高度に結合されています.
-ピアツーピア通信は複数のコンポーネントに分散され、テスト中にシミュレートする必要がある複雑な依存関係グラフを作成します.
-ステートフルコードとしてモデル化されたタイムアウトは、テストに不確実性をもたらします

このADRは、実装[ADR-40](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain-reactor-refactor.MD)を指定することを目的としています.

## 決定

ブロックチェーンリアクターの責任は、イベントとの通信専用の一連のコンポーネントに分割されます.イベントにはタイムスタンプが含まれ、各コンポーネントが内部状態として時間を追跡できるようにします.内部状態は、イベントを生成する一連の `handle *`によって変更されます.コンポーネント間の統合はリアクター内で発生し、リアクターテストはコンポーネント間の統合テストになります.このデザインは「v2」と呼ばれます.

！[v2ブロックチェーンリアクターアーキテクチャ
図)(https://github.com/tendermint/tendermint/blob/584e67ac3fac220c5c3e0652e3582eca8231e8​​14/docs/architecture/img/blockchain-reactor-v2.png)

### 関連する通信チャネルをすばやく同期する

次の図は、高速同期ルーチンと、相互の通信に使用されるチャネルとキューのタイプを示しています.
さらに、sendRoutineがPeerMConnectionを介してメッセージを送信するために使用する各リアクタチャネルが表示されます.

！[v2ブロックチェーンチャネルとキュー
図)(https://github.com/tendermint/tendermint/blob/5cf570690f989646fb3b615b734da503f038891f/docs/architecture/img/blockchain-v2-channels.png)

### リアクターの詳細の変更

リアクタには、独立した処理のために各メッセージを各サブルーチンに送信する逆多重化プログラムが含まれます.次に、各サブルーチンは関心のあるメッセージを選択し、[ADR-40](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain)で指定されたハンドルを呼び出します. function-reactor-refactor.md). demuxRoutineは「ペースメーカー」として機能し、イベントを処理するための予想時間を設定します.

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

## ライフサイクル管理

各プロセスの一連のルーチンにより、プロセスを明確なライフサイクル管理と並行して実行できます. 現在reactorに存在する `Start`、` Stop`、および `AddPeer`フックはサブルーチンに委任され、reactorにさらに結合することなく内部状態を独立して管理できるようになります.

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

## IO処理

リアクターのIO処理ルーチンは、ピアツーピア通信を分離します. ioRoutineを介したメッセージは通常、 `p2p`APIを使用する方法です. 「trySend」などの「p2p」APIがエラーを返す場合、ioRoutineはこれらのメッセージを集約してdemuxRoutineに戻し、他のルーチンに配布できます. たとえば、ioRoutineからのエラーは、より適切なピア選択の実装を通知するためにスケジューラーによって消費される可能性があります.

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

### プロセッサ内部

プロセッサは、ブロックの並べ替え、検証、および実行を担当します. プロセッサは、最後に処理されたブロックを参照して、内部カーソルの「高さ」を維持します. ブロックのグループが順不同で到着すると、プロセッサは次のブロックを処理するために必要な「高さ+1」があるかどうかを確認します. プロセッサは、ピアから高さへのマッピング「blockPeers」も維持して、「高さ」でブロックを提供したピアを追跡します. `blockPeers`を` handleRemovePeer(...) `で使用して、障害のあるピアによって提供されたすべての未処理のブロックを再配置できます.

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

## スケジュール

Scheduleは、いくつかのスケジューリングアルゴリズムに従って、blockRequestMessagesをスケジューリングするための内部状態を維持します. スケジュールは、次の領域で維持する必要があります.

-各ブロックの状態 `blockState`はmaxHeightの高さに達しているようです
-ピアのセットとそのピア状態 `peerState`
-どのピアがどのブロックを持っているか
-どのブロックがどのピアから要求されたか

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

## スケジューラ

スケジューラーは、飛行中のターゲット `n`を維持するように構成されています
メッセージと `_blockResponseMessage`からのフィードバックを使用します.
`_statusResponseMessage`と` _peerError`が最適な割り当てを生成します
`timeCheckEv`のすべてのscBlockRequestMessageで.

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

## ピア

ピアは、スケジューラが受信したメッセージに従って、各ピアのステータスを保存します.

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

## ステータス

実装

## 結果

### ポジティブ

-テストは決定論的になります
-シミュレーションは一時的になります:壁掛けの時間がタイムアウトするのを待つ必要はありません
-ピアの選択は、個別にテスト/シミュレーションできます
-原子炉をリファクタリングするための一般的な方法を開発する

### ネガティブ

### ニュートラル

### 気付く

-スケジューラー、テストスケジューラー、レビュー再スケジューラーを実装します
-実装プロセッサ、テストプロセッサ、レビュープロセッサ
-スプリッターの実装、統合テストの作成、統合テストのレビュー

## 参照する

-[ADR-40](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-040-blockchain-reactor-refactor.md):元のブロックチェーンリアクター再構築の提案
-[Blockchain re-org](https://github.com/tendermint/tendermint/pull/3561):ブロックチェーンreactor re-org(v1)の現在の実装
