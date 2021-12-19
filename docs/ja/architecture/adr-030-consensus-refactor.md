# ADR 030:コンセンサス再構築

## 環境

プロジェクトが直面している最大の課題の1つは、証明することです
私たちが試みているように、仕様の実装は正しいです
アルゴリズムとプロトコルをフォーマル検証し、改善に努める必要があります
プログラムコードの正確さに自信を持ってください。それらの1つはコアです
Tendermint-コンセンサス-現在 `コンセンサス`パッケージに含まれています。
時間の経過とともに、
アルゴリズムは副作用のあるコンテナに散在しています(現在
`コンセンサスステータス`)。アルゴリズムをテストするには、ラージオブジェクトグラフが必要です
コンテナの非決定論的な部品製造以上のセットアップ
高い確実性を防ぎます。理想的には、1対1の表現があります
[spec](https://github.com/tendermint/spec)、ドメインをテストする準備ができて簡単
エキスパート。

住所:

-[#1495](https://github.com/tendermint/tendermint/issues/1495)
-[#1692](https://github.com/tendermint/tendermint/issues/1692)

## 決定

これらの問題を解決するために、
`コンセンサス`パッケージ。まず、コンセンサスアルゴリズムを次のように分離します
最も差し迫った問題を解決するための純粋関数と有限状態マシン
自信の欠如。残りのパッケージをそのままにして、これを行います
そして、関心の分離を改善するために、その後のオプションの変更があります。

### 変更を実装する

コンセンサスのコアは、明確に定義された入力を持つ関数としてモデル化できます。

* `State`-現在のラウンド、高さなどのデータコンテナ。
* `イベント`-ネットワーク内の重要なイベント

明確な出力を生成します。

* `State`-入力を更新
* `メッセージ`-実行するアクションを示します

```go
type Event int

const (
	EventUnknown Event = iota
	EventProposal
	Majority23PrevotesBlock
	Majority23PrecommitBlock
	Majority23PrevotesAny
	Majority23PrecommitAny
	TimeoutNewRound
	TimeoutPropose
	TimeoutPrevotes
	TimeoutPrecommit
)

type Message int

const (
	MeesageUnknown Message = iota
	MessageProposal
	MessageVotes
	MessageDecision
)

type State struct {
	height      uint64
	round       uint64
	step        uint64
	lockedValue interface{} // TODO: Define proper type.
	lockedRound interface{} // TODO: Define proper type.
	validValue  interface{} // TODO: Define proper type.
	validRound  interface{} // TODO: Define proper type.
	// From the original notes: valid(v)
	valid       interface{} // TODO: Define proper type.
	// From the original notes: proposer(h, r)
	proposer    interface{} // TODO: Define proper type.
}

func Consensus(Event, State) (State, Message) {
	// Consolidate implementation.
}
```

関連情報を追跡して、機能に「イベント」を入力し、アクションを実行します
出力は `ConsensusExecutor`(以前は` ConsensusState`と呼ばれていました)に任されています。

テストの利点は、一連のイベントのテストとして十分に示されています
異議申し立てアルゴリズムは、次の例のように単純にすることができます。

``` go
func TestConsensusXXX(t *testing.T) {
	type expected struct {
		message Message
		state   State
	}

	// Setup order of events, initial state and expectation.
	var (
		events = []struct {
			event Event
			want  expected
		}{
		// ...
		}
		state = State{
		// ...
		}
	)

	for _, e := range events {
		sate, msg = Consensus(e.event, state)

		// Test message expectation.
		if msg != e.want.message {
			t.Fatalf("have %v, want %v", msg, e.want.message)
		}

		// Test state expectation.
		if !reflect.DeepEqual(state, e.want.state) {
			t.Fatalf("have %v, want %v", state, e.want.state)
		}
	}
}
```


## コンセンサスエグゼキュータ

## コンセンサスコア
```go
type Event interface{}

type EventNewHeight struct {
    Height           int64
    ValidatorId      int
}

type EventNewRound HeightAndRound

type EventProposal struct {
    Height           int64
    Round            int
    Timestamp        Time
    BlockID          BlockID
    POLRound         int
    Sender           int
}

type Majority23PrevotesBlock struct {
    Height           int64
    Round            int
    BlockID          BlockID
}

type Majority23PrecommitBlock struct {
    Height           int64
    Round            int
    BlockID          BlockID
}

type HeightAndRound struct {
    Height           int64
    Round            int
}

type Majority23PrevotesAny HeightAndRound
type Majority23PrecommitAny HeightAndRound
type TimeoutPropose HeightAndRound
type TimeoutPrevotes HeightAndRound
type TimeoutPrecommit HeightAndRound


type Message interface{}

type MessageProposal struct {
    Height           int64
    Round            int
    BlockID          BlockID
    POLRound         int
}

type VoteType int

const (
	VoteTypeUnknown VoteType = iota
	Prevote
	Precommit
)


type MessageVote struct {
    Height           int64
    Round            int
    BlockID          BlockID
    Type             VoteType
}


type MessageDecision struct {
    Height           int64
    Round            int
    BlockID          BlockID
}

type TriggerTimeout struct {
    Height           int64
    Round            int
    Duration         Duration
}


type RoundStep int

const (
	RoundStepUnknown RoundStep = iota
	RoundStepPropose
	RoundStepPrevote
	RoundStepPrecommit
	RoundStepCommit
)

type State struct {
	Height           int64
	Round            int
	Step             RoundStep
	LockedValue      BlockID
	LockedRound      int
	ValidValue       BlockID
	ValidRound       int
	ValidatorId      int
	ValidatorSetSize int
}

func proposer(height int64, round int) int {}
func getValue() BlockID {}

func Consensus(event Event, state State) (State, Message, TriggerTimeout) {
    msg = nil
    timeout = nil
	switch event := event.(type) {
    	case EventNewHeight:
    		if event.Height > state.Height {
    		    state.Height = event.Height
    		    state.Round = -1
    		    state.Step = RoundStepPropose
    		    state.LockedValue = nil
    		    state.LockedRound = -1
    		    state.ValidValue = nil
    		    state.ValidRound = -1
    		    state.ValidatorId = event.ValidatorId
    		}
    	    return state, msg, timeout

    	case EventNewRound:
    		if event.Height == state.Height and event.Round > state.Round {
               state.Round = eventRound
               state.Step = RoundStepPropose
               if proposer(state.Height, state.Round) == state.ValidatorId {
                   proposal = state.ValidValue
                   if proposal == nil {
                   	    proposal = getValue()
                   }
                   msg =  MessageProposal { state.Height, state.Round, proposal, state.ValidRound }
               }
               timeout = TriggerTimeout { state.Height, state.Round, timeoutPropose(state.Round) }
            }
    	    return state, msg, timeout

    	case EventProposal:
    		if event.Height == state.Height and event.Round == state.Round and
    	       event.Sender == proposal(state.Height, state.Round) and state.Step == RoundStepPropose {
    	       	if event.POLRound >= state.LockedRound or event.BlockID == state.BlockID or state.LockedRound == -1 {
    	       		msg = MessageVote { state.Height, state.Round, event.BlockID, Prevote }
    	       	}
    	       	state.Step = RoundStepPrevote
            }
    	    return state, msg, timeout

    	case TimeoutPropose:
    		if event.Height == state.Height and event.Round == state.Round and state.Step == RoundStepPropose {
    		    msg = MessageVote { state.Height, state.Round, nil, Prevote }
    			state.Step = RoundStepPrevote
            }
    	    return state, msg, timeout

    	case Majority23PrevotesBlock:
    		if event.Height == state.Height and event.Round == state.Round and state.Step >= RoundStepPrevote and event.Round > state.ValidRound {
    		    state.ValidRound = event.Round
    		    state.ValidValue = event.BlockID
    		    if state.Step == RoundStepPrevote {
    		    	state.LockedRound = event.Round
    		    	state.LockedValue = event.BlockID
    		    	msg = MessageVote { state.Height, state.Round, event.BlockID, Precommit }
    		    	state.Step = RoundStepPrecommit
    		    }
            }
    	    return state, msg, timeout

    	case Majority23PrevotesAny:
    		if event.Height == state.Height and event.Round == state.Round and state.Step == RoundStepPrevote {
    			timeout = TriggerTimeout { state.Height, state.Round, timeoutPrevote(state.Round) }
    		}
    	    return state, msg, timeout

    	case TimeoutPrevote:
    		if event.Height == state.Height and event.Round == state.Round and state.Step == RoundStepPrevote {
    			msg = MessageVote { state.Height, state.Round, nil, Precommit }
    			state.Step = RoundStepPrecommit
    		}
    	    return state, msg, timeout

    	case Majority23PrecommitBlock:
    		if event.Height == state.Height {
    		    state.Step = RoundStepCommit
    		    state.LockedValue = event.BlockID
    		}
    	    return state, msg, timeout

    	case Majority23PrecommitAny:
    		if event.Height == state.Height and event.Round == state.Round {
    			timeout = TriggerTimeout { state.Height, state.Round, timeoutPrecommit(state.Round) }
    		}
    	    return state, msg, timeout

    	case TimeoutPrecommit:
            if event.Height == state.Height and event.Round == state.Round {
            	state.Round = state.Round + 1
            }
    	    return state, msg, timeout
	}
}

func ConsensusExecutor() {
	proposal = nil
	votes = HeightVoteSet { Height: 1 }
	state = State {
		Height:       1
		Round:        0
		Step:         RoundStepPropose
		LockedValue:  nil
		LockedRound:  -1
		ValidValue:   nil
		ValidRound:   -1
	}

	event = EventNewHeight {1, id}
	state, msg, timeout = Consensus(event, state)

	event = EventNewRound {state.Height, 0}
	state, msg, timeout = Consensus(event, state)

	if msg != nil {
		send msg
	}

	if timeout != nil {
		trigger timeout
	}

	for {
		select {
		    case message := <- msgCh:
		    	switch msg := message.(type) {
		    	    case MessageProposal:

		    	    case MessageVote:
		    	    	if msg.Height == state.Height {
		    	    		newVote = votes.AddVote(msg)
		    	    		if newVote {
		    	    			switch msg.Type {
                                	case Prevote:
                                		prevotes = votes.Prevotes(msg.Round)
                                		if prevotes.WeakCertificate() and msg.Round > state.Round {
                                			event = EventNewRound { msg.Height, msg.Round }
                                			state, msg, timeout = Consensus(event, state)
                                			state = handleStateChange(state, msg, timeout)
                                		}

                                		if blockID, ok = prevotes.TwoThirdsMajority(); ok and blockID != nil {
                                		    if msg.Round == state.Round and hasBlock(blockID) {
                                		    	event = Majority23PrevotesBlock { msg.Height, msg.Round, blockID }
                                		    	state, msg, timeout = Consensus(event, state)
                                		    	state = handleStateChange(state, msg, timeout)
                                		    }
                                		    if proposal != nil and proposal.POLRound == msg.Round and hasBlock(blockID) {
                                		        event = EventProposal {
                                                        Height: state.Height
                                                        Round:  state.Round
                                                        BlockID: blockID
                                                        POLRound: proposal.POLRound
                                                        Sender: message.Sender
                                		        }
                                		        state, msg, timeout = Consensus(event, state)
                                		        state = handleStateChange(state, msg, timeout)
                                		    }
                                		}

                                		if prevotes.HasTwoThirdsAny() and msg.Round == state.Round {
                                			event = Majority23PrevotesAny { msg.Height, msg.Round, blockID }
                                			state, msg, timeout = Consensus(event, state)
                                            state = handleStateChange(state, msg, timeout)
                                		}

                                	case Precommit:

		    	    		    }
		    	    	    }
		    	        }
		    case timeout := <- timeoutCh:

		    case block := <- blockCh:

		}
	}
}

func handleStateChange(state, msg, timeout) State {
	if state.Step == Commit {
		state = ExecuteBlock(state.LockedValue)
	}
	if msg != nil {
		send msg
	}
	if timeout != nil {
		trigger timeout
	}
}

```

### 実装ロードマップ

*提案された実装の実装
*現在 `ConsensusState`に散在している通話を新しい通話に置き換えます
    `コンセンサス`機能
*混乱を避けるために、 `ConsensusState`の名前を` ConsensusExecutor`に変更しました
*分離を改善し、情報の流れを明確にするための設計を提案する
    `ConsensusExecutor`と` ConsensusReactor`

## ステータス

下書き。

## 結果

### ポジティブ

-アルゴリズムの分離された実装
-改善されたテスト容易性-正当性を証明するのがより簡単
-関心の分離の明確化-推論の容易化

### ネガティブ

### ニュートラル
