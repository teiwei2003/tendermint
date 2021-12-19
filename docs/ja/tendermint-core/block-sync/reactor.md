# リアクター

Blocksync Reactorの高レベルの責任は、それらを許可することです
現在のコンセンサス状態よりはるかに遅れて、ダウンロードしてすぐに追いつく
多くのブロックを並行して実行し、コミットを検証して実行します
ABCIアプリケーション。

Tendermintフルノードは、BlocksyncReactorをブロック提供サービスとして実行します
新しいノードへ。 新しいノードは、BlocksyncReactorを「fast_sync」モードで実行します。
同期されるまで、より多くのブロックをアクティブに要求します。
追いつくと、「fast_sync」モードが無効になり、ノードは次のように切り替わります。
コンセンサスリアクターを使用(およびオープン)します。

## アーキテクチャとアルゴリズム

Blocksyncリアクターは、一連の同時タスクとして編成されています。

-BlocksyncReactor受信ルーチン
-リクエスターのタスクを作成します
-一連のリクエスタータスクと-コントローラータスク。

！[Blocksync Reactorアーキテクチャ図](img/bc-reactor.png)

### データ構造

これらは、BlocksyncReactorロジックを提供するために必要なコアデータ構造です。

リクエスターのデータ構造は、IDが「peerID」と等しいピアへの「高さ」の位置にある「ブロック」のリクエストの割り当てを追跡するために使用されます。

```go
type Requester {
  mtx          Mutex
  block        Block
  height       int64
  peerID       p2p.ID
  redoChannel  chan p2p.ID//redo may send multi-time; peerId is used to identify repeat
}
```

プールは、最後に実行されたブロック( `height`)、ピアへのリクエストの割り当て(` requesters`)、各ピアの現在の高さ、および各ピアの保留中のリクエストの数( `peers`)を格納するコアデータ構造です。 、最大ピア高さなど。

```go
type Pool {
  mtx                Mutex
  requesters         map[int64]*Requester
  height             int64
  peers              map[p2p.ID]*Peer
  maxPeerHeight      int64
  numPending         int32
  store              BlockStore
  requestsChannel    chan<- BlockRequest
  errorsChannel      chan<- peerError
}
```

ピアデータ構造には、各ピアの現在の「高さ」と、ピアに送信された保留中のリクエストの数(「numPending」)などが格納されます。

```go
type Peer struct {
  id           p2p.ID
  height       int64
  numPending   int32
  timeout      *time.Timer
  didTimeout   bool
}
```

BlockRequestは、特定の「高さ」ブロックに対する現在の要求のピア( "PeerID")へのマッピングを表すために使用される内部データ構造です。

```go
type BlockRequest {
  Height int64
  PeerID p2p.ID
}
```

### Receive routine of Blocksync Reactor

これは、p2p受信ルーチンのBlocksyncChannelでメッセージが受信されたときに実行されます。 ピアごとに個別のp2p受信ルーチン(したがって、Blocksync Reactorの受信ルーチン)があります。 送信バッファがいっぱいの場合、送信の試行はブロックされないことに注意してください(すぐに戻ります)。

```go
handleMsg(pool, m):
    upon receiving bcBlockRequestMessage m from peer p:
      block = load block for height m.Height from pool.store
      if block != nil then
        try to send BlockResponseMessage(block) to p
      else
        try to send bcNoBlockResponseMessage(m.Height) to p

    upon receiving bcBlockResponseMessage m from peer p:
      pool.mtx.Lock()
      requester = pool.requesters[m.Height]
      if requester == nil then
        error("peer sent us a block we didn't expect")
        continue

      if requester.block == nil and requester.peerID == p then
        requester.block = m
        pool.numPending -= 1 //atomic decrement
        peer = pool.peers[p]
        if peer != nil then
          peer.numPending--
          if peer.numPending == 0 then
            peer.timeout.Stop()
           //NOTE: we don't send Quit signal to the corresponding requester task!
        else
          trigger peer timeout to expire after peerTimeout
      pool.mtx.Unlock()


    upon receiving bcStatusRequestMessage m from peer p:
      try to send bcStatusResponseMessage(pool.store.Height)

    upon receiving bcStatusResponseMessage m from peer p:
      pool.mtx.Lock()
      peer = pool.peers[p]
      if peer != nil then
        peer.height = m.height
      else
        peer = create new Peer data structure with id = p and height = m.Height
        pool.peers[p] = peer

      if m.Height > pool.maxPeerHeight then
        pool.maxPeerHeight = m.Height
      pool.mtx.Unlock()

onTimeout(p):
  send error message to pool error channel
  peer = pool.peers[p]
  peer.didTimeout = true
```

### リクエスタータスク

リクエスタータスクは、「高さ」の位置で単一のブロックを取得する責任があります。

```go
fetchBlock(height, pool):
  while true do {
    peerID = nil
    block = nil
    peer = pickAvailablePeer(height)
    peerID = peer.id

    enqueue BlockRequest(height, peerID) to pool.requestsChannel
    redo = false
    while !redo do
      select {
        upon receiving Quit message do
          return
        upon receiving redo message with id on redoChannel do
          if peerID == id {
            mtx.Lock()
            pool.numPending++
            redo = true
            mtx.UnLock()
          }
      }
    }

pickAvailablePeer(height):
  selectedPeer = nil
  while selectedPeer = nil do
    pool.mtx.Lock()
    for each peer in pool.peers do
      if !peer.didTimeout and peer.numPending < maxPendingRequestsPerPeer and peer.height >= height then
        peer.numPending++
        selectedPeer = peer
        break
    pool.mtx.Unlock()

    if selectedPeer = nil then
      sleep requestIntervalMS

  return selectedPeer
```

requestIntervalMSのスリープ

### リクエスターのタスクを作成する

このタスクは、リクエスタータスクを常に作成して開始する責任があります。

```go
createRequesters(pool):
  while true do
    if !pool.isRunning then break
    if pool.numPending < maxPendingRequests or size(pool.requesters) < maxTotalRequesters then
      pool.mtx.Lock()
      nextHeight = pool.height + size(pool.requesters)
      requester = create new requester for height nextHeight
      pool.requesters[nextHeight] = requester
      pool.numPending += 1//atomic increment
      start requester task
      pool.mtx.Unlock()
    else
      sleep requestIntervalMS
      pool.mtx.Lock()
      for each peer in pool.peers do
        if !peer.didTimeout && peer.numPending > 0 && peer.curRate < minRecvRate then
          send error on pool error channel
          peer.didTimeout = true
        if peer.didTimeout then
          for each requester in pool.requesters do
            if requester.getPeerID() == peer then
              enqueue msg on requestor's redoChannel
          delete(pool.peers, peerID)
      pool.mtx.Unlock()
```

### マスターブロック同期リアクターコントローラータスク

```go
main(pool):
  create trySyncTicker with interval trySyncIntervalMS
  create statusUpdateTicker with interval statusUpdateIntervalSeconds
  create switchToConsensusTicker with interval switchToConsensusIntervalSeconds

  while true do
    select {
   upon receiving BlockRequest(Height, Peer) on pool.requestsChannel:
     try to send bcBlockRequestMessage(Height) to Peer

   upon receiving error(peer) on errorsChannel:
     stop peer for error

   upon receiving message on statusUpdateTickerChannel:
     broadcast bcStatusRequestMessage(bcR.store.Height)//message sent in a separate routine

   upon receiving message on switchToConsensusTickerChannel:
     pool.mtx.Lock()
     receivedBlockOrTimedOut = pool.height > 0 || (time.Now() - pool.startTime) > 5 Seconds
     ourChainIsLongestAmongPeers = pool.maxPeerHeight == 0 || pool.height >= pool.maxPeerHeight
     haveSomePeers = size of pool.peers > 0
     pool.mtx.Unlock()
     if haveSomePeers && receivedBlockOrTimedOut && ourChainIsLongestAmongPeers then
       switch to consensus mode

          upon receiving message on trySyncTickerChannel:
            for i = 0; i < 10; i++ do
              pool.mtx.Lock()
              firstBlock = pool.requesters[pool.height].block
              secondBlock = pool.requesters[pool.height].block
              if firstBlock == nil or secondBlock == nil then continue
              pool.mtx.Unlock()
              verify firstBlock using LastCommit from secondBlock
              if verification failed
                pool.mtx.Lock()
                peerID = pool.requesters[pool.height].peerID
                redoRequestsForPeer(peerId)
                delete(pool.peers, peerID)
                stop peer peerID for error
                pool.mtx.Unlock()
              else
                delete(pool.requesters, pool.height)
                save firstBlock to store
                pool.height++
                execute firstBlock
    }

redoRequestsForPeer(pool, peerId):
  for each requester in pool.requesters do
    if requester.getPeerID() == peerID
     enqueue msg on redoChannel for requester
```

## チャンネル

着信メッセージの最大サイズとして `maxMsgSize`を定義します。
`SendQueueCapacity`と` RecvBufferCapacity`は、最大の送信と
バッファを個別に受信します。 これらは拡大を防ぐためのものでなければなりません
送受信できるデータ量に上限を設定して攻撃する
なし。

誤ってコード化されたデータを送信すると、ピアが停止します。
