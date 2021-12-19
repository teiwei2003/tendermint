# 反应堆

共识反应器为共识服务定义了一个反应器。它包含 ConsensusState 服务
管理 Tendermint 共识内部状态机的状态。
当 Consensus Reactor 启动时，它会启动 Broadcast Routine，启动 ConsensusState 服务。
此外，对于添加到共识反应器的每个对等点，它创建(并管理)已知的对等点状态
(在八卦例程中广泛使用)并为对等 p 启动以下三个例程:
Gossip Data Routine、Gossip Votes Routine 和 QueryMaj23Routine。最后，Consensus Reactor 负责
用于解码从对等方接收的消息，并根据消息的类型和内容对消息进行适当的处​​理。
处理通常包括更新已知的对等状态和一些消息
(`ProposalMessage`、`BlockPartMessage` 和 `VoteMessage`)也将消息转发到 ConsensusState 模块
作进一步处理。在下面的文本中，我们指定了这些单独的执行单元的核心功能
是共识反应堆的一部分。

## 共识状态服务

共识状态处理 Tendermint BFT 共识算法的执行。它处理投票和提案，
并在达成一致后，将块提交到链中并针对应用程序执行它们。
内部状态机接收来自对等方、内部验证器和定时器的输入。

在 Consensus State 中，我们有以下执行单元:Timeout Ticker 和 Receive Routine。
Timeout Ticker 是一个计时器，它根据处理的高度/轮次/步长安排超时
通过接收例程。

### 接收 ConsensusState 服务的例程

ConsensusState 的接收例程处理可能导致内部共识状态转换的消息。
它是更新包含内部共识状态的 RoundState 的唯一例程。
更新(状态转换)发生在超时、完整提案和 2/3 多数时。
它接收来自对等方、内部验证器和 Timeout Ticker 的消息
并调用相应的处理程序，可能会更新 RoundState。
接收例程实现的协议细节(连同正确性的正式证明)是
在单独的文件中讨论。为了理解本文档
理解接收例程管理和更新 RoundState 数据结构就足够了
然后被 gossip 程序广泛使用来确定应该将哪些信息发送到对等进程。

## 圆形状态

RoundState 定义了内部共识状态。它包含高度、圆形、圆形步长、当前验证器集、
当前轮次的提议和提议区块，锁定轮次和区块(如果某个区块被锁定)，一组
收到选票并设置最后一次提交和最后一次验证器。

```go
type RoundState struct {
 Height             int64
 Round              int
 Step               RoundStepType
 Validators         ValidatorSet
 Proposal           Proposal
 ProposalBlock      Block
 ProposalBlockParts PartSet
 LockedRound        int
 LockedBlock        Block
 LockedBlockParts   PartSet
 Votes              HeightVoteSet
 LastCommit         VoteSet
 LastValidators     ValidatorSet
}
```

在内部，共识将作为具有以下状态的状态机运行:

- RoundStepNewHeight
- RoundStepNewRound
- RoundStepPropose
- RoundStepProposeWait
- RoundStepPrevote
- RoundStepPrevoteWait
- RoundStepPrecommit
- RoundStepPrecommitWait
- RoundStepCommit

## 对等轮状态

对等轮状态包含对等方的已知状态。 它正在被 Receive 例程更新
共识反应器和八卦例程在向对等方发送消息时。

```golang
type PeerRoundState struct {
 Height                   int64               // Height peer is at
 Round                    int                 // Round peer is at, -1 if unknown.
 Step                     RoundStepType       // Step peer is at
 Proposal                 bool                // True if peer has proposal for this round
 ProposalBlockPartsHeader PartSetHeader
 ProposalBlockParts       BitArray
 ProposalPOLRound         int                 // Proposal's POL round. -1 if none.
 ProposalPOL              BitArray            // nil until ProposalPOLMessage received.
 Prevotes                 BitArray            // All votes peer has for this round
 Precommits               BitArray            // All precommits peer has for this round
 LastCommitRound          int                 // Round of commit for last height. -1 if none.
 LastCommit               BitArray            // All commit precommits of commit for last height.
 CatchupCommitRound       int                 // Round that we have commit for. Not necessarily unique. -1 if none.
 CatchupCommit            BitArray            // All commit precommits peer has for this height & CatchupCommitRound
}
```

## Consensus reactor的接收方法

共识反应器的入口点是一个接收方法。 当一条消息
从对等体 p 接收，通常更新对等体轮次状态
相应地，一些消息被传递以供进一步处理，对于
以 ConsensusState 服务为例。 我们现在指定消息的处理
每种消息类型的共识反应器的接收方法。 在下面的
消息处理程序，`rs` 和`prs` 表示`RoundState` 和`PeerRoundState`，
分别。

### NewRoundStepMessage 处理程序

```go
handleMessage(msg):
    if msg is from smaller height/round/step then return
    // Just remember these values.
    prsHeight = prs.Height
    prsRound = prs.Round
    prsCatchupCommitRound = prs.CatchupCommitRound
    prsCatchupCommit = prs.CatchupCommit

    Update prs with values from msg
    if prs.Height or prs.Round has been updated then
        reset Proposal related fields of the peer state
    if prs.Round has been updated and msg.Round == prsCatchupCommitRound then
        prs.Precommits = psCatchupCommit
    if prs.Height has been updated then
        if prsHeight+1 == msg.Height && prsRound == msg.LastCommitRound then
            prs.LastCommitRound = msg.LastCommitRound
         prs.LastCommit = prs.Precommits
        } else {
            prs.LastCommitRound = msg.LastCommitRound
         prs.LastCommit = nil
        }
        Reset prs.CatchupCommitRound and prs.CatchupCommit
```

### NewValidBlockMessage handler

```go
handleMessage(msg):
    if prs.Height != msg.Height then return

    if prs.Round != msg.Round && !msg.IsCommit then return

    prs.ProposalBlockPartsHeader = msg.BlockPartsHeader
    prs.ProposalBlockParts = msg.BlockParts
```

The number of block parts is limited to 1601 (`types.MaxBlockPartsCount`) to
protect the node against DOS attacks.

### HasVoteMessage handler

```go
handleMessage(msg):
    if prs.Height == msg.Height then
        prs.setHasVote(msg.Height, msg.Round, msg.Type, msg.Index)
```

### VoteSetMaj23Message handler

```go
handleMessage(msg):
    if prs.Height == msg.Height then
        Record in rs that a peer claim to have ⅔ majority for msg.BlockID
        Send VoteSetBitsMessage showing votes node has for that BlockId
```

### ProposalMessage handler

```go
handleMessage(msg):
    if prs.Height != msg.Height || prs.Round != msg.Round || prs.Proposal then return
    prs.Proposal = true
    if prs.ProposalBlockParts == empty set then // otherwise it is set in NewValidBlockMessage handler
      prs.ProposalBlockPartsHeader = msg.BlockPartsHeader
    prs.ProposalPOLRound = msg.POLRound
    prs.ProposalPOL = nil
    Send msg through internal peerMsgQueue to ConsensusState service
```

### ProposalPOLMessage handler

```go
handleMessage(msg):
    if prs.Height != msg.Height or prs.ProposalPOLRound != msg.ProposalPOLRound then return
    prs.ProposalPOL = msg.ProposalPOL
```

The number of votes is limited to 10000 (`types.MaxVotesCount`) to protect the
node against DOS attacks.

### BlockPartMessage handler

```go
handleMessage(msg):
    if prs.Height != msg.Height || prs.Round != msg.Round then return
    Record in prs that peer has block part msg.Part.Index
    Send msg trough internal peerMsgQueue to ConsensusState service
```

### VoteMessage handler

```go
handleMessage(msg):
    Record in prs that a peer knows vote with index msg.vote.ValidatorIndex for particular height and round
    Send msg trough internal peerMsgQueue to ConsensusState service
```

### VoteSetBitsMessage handler

```go
handleMessage(msg):
    Update prs for the bit-array of votes peer claims to have for the msg.BlockID
```

投票数限制在 10000 (`types.MaxVotesCount`) 以保护
节点抵御 DOS 攻击。

## 八卦数据例程

它用于向对等方发送以下消息:`BlockPartMessage`、`ProposalMessage` 和
DataChannel 上的`ProposalPOLMessage`。 gossip 数据例程基于本地 RoundState (`rs`)
和已知的 PeerRoundState (`prs`)。 该例程永远重复如下所示的逻辑:

```go
1a) if rs.ProposalBlockPartsHeader == prs.ProposalBlockPartsHeader and the peer does not have all the proposal parts then
        Part = pick a random proposal block part the peer does not have
        Send BlockPartMessage(rs.Height, rs.Round, Part) to the peer on the DataChannel
        if send returns true, record that the peer knows the corresponding block Part
     Continue

1b) if (0 < prs.Height) and (prs.Height < rs.Height) then
        help peer catch up using gossipDataForCatchup function
        Continue

1c) if (rs.Height != prs.Height) or (rs.Round != prs.Round) then
        Sleep PeerGossipSleepDuration
        Continue

//  at this point rs.Height == prs.Height and rs.Round == prs.Round
1d) if (rs.Proposal != nil and !prs.Proposal) then
        Send ProposalMessage(rs.Proposal) to the peer
        if send returns true, record that the peer knows Proposal
     if 0 <= rs.Proposal.POLRound then
     polRound = rs.Proposal.POLRound
        prevotesBitArray = rs.Votes.Prevotes(polRound).BitArray()
        Send ProposalPOLMessage(rs.Height, polRound, prevotesBitArray)
        Continue

2)  Sleep PeerGossipSleepDuration
```

### 八卦数据追赶

如果它在较小的高度(prs.Height < rs.Height)，这个函数负责帮助peer 赶上它。
该函数执行以下逻辑:

```go
    if peer does not have all block parts for prs.ProposalBlockPart then
        blockMeta =  Load Block Metadata for height prs.Height from blockStore
        if (!blockMeta.BlockID.PartsHeader == prs.ProposalBlockPartsHeader) then
            Sleep PeerGossipSleepDuration
     return
        Part = pick a random proposal block part the peer does not have
        Send BlockPartMessage(prs.Height, prs.Round, Part) to the peer on the DataChannel
        if send returns true, record that the peer knows the corresponding block Part
        return
    else Sleep PeerGossipSleepDuration
```

##八卦投票例程

它用于在 VoteChannel 上发送以下消息:`VoteMessage`。
八卦投票例程基于本地 RoundState (`rs`)
和已知的 PeerRoundState (`prs`)。 该例程永远重复如下所示的逻辑:

```go
1a) if rs.Height == prs.Height then
        if prs.Step == RoundStepNewHeight then
            vote = random vote from rs.LastCommit the peer does not have
            Send VoteMessage(vote) to the peer
            if send returns true, continue

        if prs.Step <= RoundStepPrevote and prs.Round != -1 and prs.Round <= rs.Round then
            Prevotes = rs.Votes.Prevotes(prs.Round)
            vote = random vote from Prevotes the peer does not have
            Send VoteMessage(vote) to the peer
            if send returns true, continue

        if prs.Step <= RoundStepPrecommit and prs.Round != -1 and prs.Round <= rs.Round then
          Precommits = rs.Votes.Precommits(prs.Round)
            vote = random vote from Precommits the peer does not have
            Send VoteMessage(vote) to the peer
            if send returns true, continue

        if prs.ProposalPOLRound != -1 then
            PolPrevotes = rs.Votes.Prevotes(prs.ProposalPOLRound)
            vote = random vote from PolPrevotes the peer does not have
            Send VoteMessage(vote) to the peer
            if send returns true, continue

1b)  if prs.Height != 0 and rs.Height == prs.Height+1 then
        vote = random vote from rs.LastCommit peer does not have
        Send VoteMessage(vote) to the peer
        if send returns true, continue

1c)  if prs.Height != 0 and rs.Height >= prs.Height+2 then
        Commit = get commit from BlockStore for prs.Height
        vote = random vote from Commit the peer does not have
        Send VoteMessage(vote) to the peer
        if send returns true, continue

2)   Sleep PeerGossipSleepDuration
```

## QueryMaj23Routine

它用于发送以下消息:`VoteSetMaj23Message`。 `VoteSetMaj23Message` 被发送以指示给定的
BlockID 已获得 +2/3 票。 此例程基于本地 RoundState (`rs`) 和已知的 PeerRoundState
(`prs`)。 该例程永远重复如下所示的逻辑。

```go
1a) if rs.Height == prs.Height then
        Prevotes = rs.Votes.Prevotes(prs.Round)
        if there is a ⅔ majority for some blockId in Prevotes then
        m = VoteSetMaj23Message(prs.Height, prs.Round, Prevote, blockId)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

1b) if rs.Height == prs.Height then
        Precommits = rs.Votes.Precommits(prs.Round)
        if there is a ⅔ majority for some blockId in Precommits then
        m = VoteSetMaj23Message(prs.Height,prs.Round,Precommit,blockId)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

1c) if rs.Height == prs.Height and prs.ProposalPOLRound >= 0 then
        Prevotes = rs.Votes.Prevotes(prs.ProposalPOLRound)
        if there is a ⅔ majority for some blockId in Prevotes then
        m = VoteSetMaj23Message(prs.Height,prs.ProposalPOLRound,Prevotes,blockId)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

1d) if prs.CatchupCommitRound != -1 and 0 < prs.Height and
        prs.Height <= blockStore.Height() then
        Commit = LoadCommit(prs.Height)
        m = VoteSetMaj23Message(prs.Height,Commit.Round,Precommit,Commit.BlockID)
        Send m to peer
        Sleep PeerQueryMaj23SleepDuration

2)  Sleep PeerQueryMaj23SleepDuration
```

## 广播例程

广播例程订阅内部事件总线以接收新一轮的步骤和投票消息，并在收到这些消息后向对等方广播消息
事件。
它在新一轮状态事件时广播 `NewRoundStepMessage` 或 `CommitStepMessage`。 注意
广播这些消息不依赖于 PeerRoundState； 它在 StateChannel 上发送。
收到 VoteMessage 后，它会在 StateChannel 上向其对等方广播“HasVoteMessage”消息。

## 频道

定义了 4 个通道:状态、数据、投票和vote_set_bits。 每个通道
有 `SendQueueCapacity` 和 `RecvBufferCapacity` 和
`RecvMessageCapacity` 设置为 `maxMsgSize`。

发送错误编码的数据将导致停止对等点。
