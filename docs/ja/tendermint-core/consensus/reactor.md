# リアクター

コンセンサスリアクターは、コンセンサスサービス用のリアクターを定義します. ConsensusStateサービスが含まれています
Tendermintコンセンサス内部ステートマシンの状態を管理します.
Consensus Reactorが起動すると、Broadcast Routineが開始され、ConsensusStateサービスが開始されます.
さらに、コンセンサスリアクタに追加されたピアごとに、既知のピア状態を作成(および管理)します.
(ゴシップルーチンで広く使用されています)そして、ピアpに対して次の3つのルーチンを開始します.
ゴシップデータルーチン、ゴシップ投票ルーチン、およびQueryMaj23Routine.最後に、コンセンサスリアクターが責任を負います
ピアから受信したメッセージをデコードし、メッセージの種類と内容に応じてメッセージを適切に処理するために使用されます.
処理には通常、既知のピアステータスといくつかのメッセージの更新が含まれます
( `ProposalMessage`、` BlockPartMessage`および `VoteMessage`)もメッセージをConsensusStateモジュールに転送します
さらなる処理のため.以下のテキストでは、これらの個々の実行ユニットのコア機能を指定します
コンセンサスリアクターの一部です.

## コンセンサスステートサービス

コンセンサス状態は、テンダーミントBFTコンセンサスアルゴリズムの実行を扱います.投票と提案を処理し、
そして、合意に達した後、ブロックをチェーンに送信し、アプリケーションに対して実行します.
内部ステートマシンは、ピア、内部バリデーター、およびタイマーから入力を受け取ります.

コンセンサス状態では、次の実行ユニットがあります:タイムアウトティッカーと受信ルーチン.
タイムアウトティッカーは、処理の高さ/ラウンド/ステップ長に応じてタイムアウトをスケジュールするタイマーです.
受信ルーチンを介して.

### ConsensusStateサービスを受ける例

ConsensusStateの受信ルーチンは、内部コンセンサス状態遷移を引き起こす可能性のあるメッセージを処理します.
これは、内部コンセンサス状態を含むRoundStateを更新する唯一のルーチンです.
更新(状態遷移)は、タイムアウト、完全な提案、および2/3の過半数で発生します.
ピア、内部バリデーター、タイムアウトティッカーからメッセージを受信します
そして、対応するハンドラーを呼び出すと、RoundStateが更新される場合があります.
受信ルーチンによって実装されるプロトコルの詳細(および正式な正当性の証明)は次のとおりです.
別のドキュメントで話し合います.このドキュメントを理解するために
受信ルーチンがRoundStateデータ構造を管理および更新することを理解するだけで十分です.
次に、ゴシッププログラムで広く使用され、ピアプロセスに送信する情報を決定します.

## 世俗国家

RoundStateは、内部コンセンサス状態を定義します.高さ、円、円のステップ、現在のバリデーターセット、
提案の現在のラウンドと提案されたブロック、ロックラウンドとブロック(ブロックがロックされている場合)、セット
投票を受け取り、最後の提出と最後の検証者を設定します.
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

内部的には、コンセンサスは次の状態のステートマシンとして動作します.

- RoundStepNewHeight
- RoundStepNewRound
- RoundStepPropose
- RoundStepProposeWait
- RoundStepPrevote
- RoundStepPrevoteWait
- RoundStepPrecommit
- RoundStepPrecommitWait
- RoundStepCommit

## ピアホイールのステータス

ピア状態には、ピアの既知の状態が含まれます. 受信ルーチンによって更新されています
コンセンサスリアクターとゴシップルーチンは、ピアにメッセージを送信するときのものです.

```golang
type PeerRoundState struct {
 Height                   int64              //Height peer is at
 Round                    int                //Round peer is at, -1 if unknown.
 Step                     RoundStepType      //Step peer is at
 Proposal                 bool               //True if peer has proposal for this round
 ProposalBlockPartsHeader PartSetHeader
 ProposalBlockParts       BitArray
 ProposalPOLRound         int                //Proposal's POL round. -1 if none.
 ProposalPOL              BitArray           //nil until ProposalPOLMessage received.
 Prevotes                 BitArray           //All votes peer has for this round
 Precommits               BitArray           //All precommits peer has for this round
 LastCommitRound          int                //Round of commit for last height. -1 if none.
 LastCommit               BitArray           //All commit precommits of commit for last height.
 CatchupCommitRound       int                //Round that we have commit for. Not necessarily unique. -1 if none.
 CatchupCommit            BitArray           //All commit precommits peer has for this height & CatchupCommitRound
}
```

## コンセンサスリアクターの受信方法

コンセンサスリアクターのエントリポイントは、受信方法です. メッセージが
ピアpから受信し、通常はピアラウンドステータスを更新します
したがって、一部のメッセージは、さらに処理するために渡されます.
例としてConsensusStateサービスを取り上げます. メッセージの処理を指定します
メッセージタイプごとのコンセンサスリアクタの受信方法. 下
メッセージハンドラ、 `rs`および` prs`は `RoundState`および` PeerRoundState`を表します.
それぞれ.

### NewRoundStepMessageハンドラー

```go
handleMessage(msg):
    if msg is from smaller height/round/step then return
   //Just remember these values.
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
    if prs.ProposalBlockParts == empty set then//otherwise it is set in NewValidBlockMessage handler
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

保護のため、投票数は10000( `types.MaxVotesCount`)に制限されています
ノードはDOS攻撃に抵抗します.

##八卦データルーチン

これは、次のメッセージをピアに送信するために使用されます: `BlockPartMessage`、` ProposalMessage`、および
DataChannelの `ProposalPOLMessage`. ゴシップデータルーチンは、ローカルのRoundState( `rs`)に基づいています.
そして、既知のPeerRoundState( `prs`). このルーチンは、常に以下に示すロジックを繰り返します.

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

// at this point rs.Height == prs.Height and rs.Round == prs.Round
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

如果它在较小的高度(prs.Height < rs.Height)，这个函数负责帮助peer 赶上它.
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

## ゴシップ投票ルーチン

これは、VoteChannelで次のメッセージを送信するために使用されます: `VoteMessage`.
ゴシップ投票ルーチンは、ローカルのRoundState( `rs`)に基づいています.
そして、既知のPeerRoundState( `prs`). このルーチンは、常に以下に示すロジックを繰り返します.

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

次のメッセージを送信するために使用されます: `VoteSetMaj23Message`. `VoteSetMaj23Message`は、指定されたものを示すために送信されます
BlockIDは+2/3票を獲得しています. このルーチンは、ローカルのRoundState( `rs`)と既知のPeerRoundStateに基づいています
( `prs`). このルーチンは、常に以下に示すロジックを繰り返します.

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

## ブロードキャストルーチン

ブロードキャストルーチンは、内部イベントバスにサブスクライブして、新しいラウンドのステップと投票メッセージを受信し、これらのメッセージを受信した後、メッセージをピアにブロードキャストします.
イベント.
ステータスイベントの新しいラウンドで `NewRoundStepMessage`または` CommitStepMessage`をブロードキャストします. 知らせ
これらのメッセージのブロードキャストはPeerRoundStateに依存せず、StateChannelで送信されます.
投票メッセージを受信した後、StateChannel上のピアに「HasVoteMessage」メッセージをブロードキャストします.

## チャンネル

ステータス、データ、投票、vote_set_bitsの4つのチャネルが定義されています. チャネルごと
`SendQueueCapacity`と` RecvBufferCapacity`があり、
`RecvMessageCapacity`は` maxMsgSize`に設定されます.

誤ってコード化されたデータを送信すると、ピアが停止します.
