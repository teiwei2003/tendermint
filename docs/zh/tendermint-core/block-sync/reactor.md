# 反应堆

Blocksync Reactor 的高层职责是让那些
远远落后于当前的共识状态，通过下载快速赶上
许多块并行，验证它们的提交，并针对它们执行它们
ABCI 申请。

Tendermint 全节点运行 Blocksync Reactor 作为提供区块的服务
到新节点。新节点以“fast_sync”模式运行 Blocksync Reactor，
他们主动请求更多块，直到它们同步。
一旦赶上，“fast_sync”模式被禁用，节点切换到
使用(并打开)Consensus Reactor。

## 架构和算法

Blocksync 反应器被组织为一组并发任务:

- Blocksync Reactor的接收例程
- 创建请求者的任务
- 一组请求者任务和 - 控制器任务。

![Blocksync Reactor 架构图](img/bc-reactor.png)

### 数据结构

这些是提供 Blocksync Reactor 逻辑所必需的核心数据结构。

请求者数据结构用于跟踪对位置“height”处的“block”的请求分配给 id 等于“peerID”的对等方。

```go
type Requester {
  mtx          Mutex
  block        Block
  height       int64
  peerID       p2p.ID
  redoChannel  chan p2p.ID //redo may send multi-time; peerId is used to identify repeat
}
```

Pool 是一个核心数据结构，它存储最后执行的块(`height`)、对peer的请求分配(`requesters`)、每个peer的当前高度和每个peer的待处理请求数(`peers`)、最大peer高度 ， 等等。

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

Peer 数据结构存储每个 Peer 当前的“高度”和发送到对等点的待处理请求数(“numPending”)等。

```go
type Peer struct {
  id           p2p.ID
  height       int64
  numPending   int32
  timeout      *time.Timer
  didTimeout   bool
}
```

BlockRequest 是内部数据结构，用于表示当前对某个“高度”的块的请求到对等方(“PeerID”)的映射。

```go
type BlockRequest {
  Height int64
  PeerID p2p.ID
}
```

### Receive routine of Blocksync Reactor

它在 p2p 接收例程内的 BlocksyncChannel 上接收消息时执行。 有一个单独的 p2p 接收例程(因此是 Blocksync Reactor 的接收例程)为每个对等点执行。 请注意，如果传出缓冲区已满，则尝试发送不会阻塞(立即返回)。

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
        pool.numPending -= 1  // atomic decrement
        peer = pool.peers[p]
        if peer != nil then
          peer.numPending--
          if peer.numPending == 0 then
            peer.timeout.Stop()
            // NOTE: we don't send Quit signal to the corresponding requester task!
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

### 请求者任务

请求者任务负责在“height”位置获取单个块。

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

为 requestIntervalMS 睡眠

### 创建请求者的任务

此任务负责不断创建和启动请求者任务。

```go
createRequesters(pool):
  while true do
    if !pool.isRunning then break
    if pool.numPending < maxPendingRequests or size(pool.requesters) < maxTotalRequesters then
      pool.mtx.Lock()
      nextHeight = pool.height + size(pool.requesters)
      requester = create new requester for height nextHeight
      pool.requesters[nextHeight] = requester
      pool.numPending += 1 // atomic increment
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

### 主块同步反应器控制器任务

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
     broadcast bcStatusRequestMessage(bcR.store.Height) // message sent in a separate routine

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

## 频道

为传入消息的最大大小定义 `maxMsgSize`，
`SendQueueCapacity` 和 `RecvBufferCapacity` 用于最大发送和
分别接收缓冲区。 这些应该是为了防止放大
通过设置我们可以接收和发送多少数据的上限来进行攻击
一个梨。

发送错误编码的数据将导致停止对等点。
