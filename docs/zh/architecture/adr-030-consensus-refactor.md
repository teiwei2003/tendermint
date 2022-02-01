# ADR 030:共识重构

## 语境

该项目面临的最大挑战之一是证明
规范的实现是正确的，就像我们努力的那样
正式验证我们的算法和协议，我们应该努力提高
对我们程序代码的正确性充满信心.其中之一是核心
Tendermint - Consensus - 目前位于 `consensus` 包中.
随着时间的推移，由于
算法分散在一个有副作用的容器中(当前
`共识状态`).为了测试算法，一个大的对象图需要
设置，甚至比容器的非确定性部分制造
防止高确定性.理想情况下，我们有一个 1 对 1 的表示
[spec](https://github.com/tendermint/spec)，准备好且易于测试域
专家.

地址:

- [#1495](https://github.com/tendermint/tendermint/issues/1495)
- [#1692](https://github.com/tendermint/tendermint/issues/1692)

## 决定

为了解决这些问题，我们计划对
`共识`包.首先将共识算法隔离为
一个纯函数和一个有限状态机来解决最紧迫的问题
缺乏信心.这样做的同时保持包装的其余部分完好无损
并有后续的可选更改以改进关注点的分离.

### 实施更改

共识的核心可以建模为一个具有明确定义输入的函数:

* `State` - 当前回合、高度等的数据容器.
* `Event` - 网络中的重要事件

产生清晰的输出；

* `State` - 更新输入
* `Message` - 表示要执行的操作

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

跟踪相关信息以将“事件”输入函数并采取行动
输出留给`ConsensusExecutor`(以前称为`ConsensusState`).

测试的好处 很好地呈现为测试一系列事件
反对算法可以像以下示例一样简单:

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


## 共识执行器

## 共识核心
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

### 实施路线图

* 实施提议的实施
* 将当前分散在 `ConsensusState` 中的调用替换为对新的调用
   `共识`功能
* 将 `ConsensusState` 重命名为 `ConsensusExecutor` 以避免混淆
* 提出设计以改善分离和清晰的信息流
   `ConsensusExecutor` 和 `ConsensusReactor`

## 状态

草稿.

## 结果

### 积极的

- 算法的隔离实现
- 改进的可测试性 - 更容易证明正确性
- 更清晰的关注点分离 - 更容易推理

### 消极的

### 中性的
