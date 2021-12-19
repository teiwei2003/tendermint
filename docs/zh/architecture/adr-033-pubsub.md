# ADR 033:发布订阅 2.0

作者:安东·卡利亚耶夫 (@melekes)

## 变更日志

02-10-2018:初稿

16-01-2019:基于我们与 Jae 对话的第二个版本

17-01-2019:第三版解释新设计如何解决当前问题

25-01-2019:第四个版本以区别对待缓冲和非缓冲通道

## 语境

自 pubsub 的初始版本以来，出现了许多问题
提出:[#951]、[#1879]、[#1880]。 其中一些是质疑
做出的核心设计选择。 其他是次要的，主要是关于
`订阅()`/`发布()` 函数。

### 同步与异步

现在，当向订阅者发布消息时，我们可以在 goroutine 中进行:

_using channels for data transmission_
```go
for each subscriber {
    out := subscriber.outc
    go func() {
        out <- msg
    }
}
```

_by invoking callback functions_
```go
for each subscriber {
    go subscriber.callbackFn()
}
```

这为我们提供了更高的性能并允许我们避免“慢客户端问题”
(当其他订阅者必须等待慢订阅者时)。一池
goroutines 可用于避免不受控制的内存增长。

在某些情况下，这就是您想要的。但在我们的例子中，因为我们需要
事件的严格排序(如果事件 A 在 B 之前发布，则保证
交付顺序将是 A -> B)，我们不能每次都在新的 goroutine 中发布 msg。

我们也可以为每个订阅者设置一个 goroutine，尽管我们需要小心
与订阅者的数量。也更难实施 +
不清楚我们是否会从中受益(因为我们将被迫创建 N 个额外的
将 msg 分发给这些 goroutine 的通道)。

### 非阻塞发送

每当我们应该进行非阻塞发送时，还有一个问题。
目前，发送是阻塞的，所以发布到一个客户端可以阻塞
发布到另一个。这意味着缓慢或无响应的客户端可以停止
系统。相反，我们可以使用非阻塞发送:

```go
for each subscriber {
    out := subscriber.outc
    select {
        case out <- msg:
        default:
            log("subscriber %v buffer is full, skipping...")
    }
}
```

这解决了“慢客户端问题”，但慢客户端无法
知道它是否错过了一条消息。我们可以返回第二个频道并关闭它
表示订阅终止。另一方面，如果我们要
坚持阻塞发送，**开发人员必须始终确保订阅者的处理代码
不堵**，这是他们肩上的艰巨任务。

临时选项是为单个消息运行 goroutines 池，等待所有
goroutines 来完成。这将解决“慢客户端问题”，但我们仍然
必须等待 `max(goroutine_X_time)` 才能发布下一条消息。

### 通道与回调

还有一个问题是我们是否应该使用通道进行消息传输或
调用订阅者定义的回调函数。回调函数给订阅者
更大的灵活性——你可以在那里使用互斥锁、通道、生成 goroutines，
你真正想要的任何东西。但它们也带有局部作用域，这可能导致
内存泄漏和/或内存使用量增加。

Go 通道是在 goroutine 之间传输数据的事实上的标准。

### 为什么`Subscribe()` 接受`out` 频道？

因为在我们的测试中，我们创建了缓冲通道(上限:1)。或者，我们
可以将容量作为参数并返回通道。

## 决定

### MsgAndTags

在订阅频道上使用 `MsgAndTags` 结构来指示哪些标记
msg 匹配。

```go
type MsgAndTags struct {
    Msg interface{}
    Tags TagMap
}
```

### 订阅结构


更改 `Subscribe()` 函数以返回 `Subscription` 结构:

```go
type Subscription struct {
  // private fields
}

func (s *Subscription) Out() <-chan MsgAndTags
func (s *Subscription) Canceled() <-chan struct{}
func (s *Subscription) Err() error
```

`Out()` 返回一个发布消息和标签的通道。
`Unsubscribe`/`UnsubscribeAll` 不会关闭频道以避免客户端
收到一条 nil 消息。

`Canceled()` 返回订阅终止时关闭的频道
并且应该在选择语句中使用。

如果 `Canceled()` 返回的通道尚未关闭，`Err()` 返回 nil。
如果通道关闭，`Err()` 返回一个非 nil 错误解释原因:
`ErrUnsubscribed` 如果订阅者选择取消订阅，
`ErrOutOfCapacity` 如果订阅者没有足够快地拉取消息并且 `Out()` 返回的通道已满。
在 `Err()` 返回非零错误后，对 `Err() 的连续调用返回相同的错误。

```go
subscription, err := pubsub.Subscribe(...)
if err != nil {
  // ...
}
for {
select {
  case msgAndTags <- subscription.Out():
    // ...
  case <-subscription.Canceled():
    return subscription.Err()
}
```

### 容量和订阅

默认情况下使 `Out()` 通道缓冲(容量为 1)。 大多数情况下，我们希望
终止慢速订阅者。 只有在极少数情况下，我们才想要阻止发布订阅
(例如，在调试共识时)。 这应该会降低发布订阅的机会
被冻结。

```go
// outCap can be used to set capacity of Out channel
// (1 by default, must be greater than 0).
Subscribe(ctx context.Context, clientID string, query Query, outCap... int) (Subscription, error) {
```

Use a different function for an unbuffered channel:

```go
// Subscription uses an unbuffered channel. Publishing will block.
SubscribeUnbuffered(ctx context.Context, clientID string, query Query) (Subscription, error) {
```

SubscribeUnbuffered 不应向用户公开。

### 阻塞/非阻塞

出版商应分别对待这些类型的渠道。
它应该阻塞无缓冲的通道(用于内部共识事件
在共识测试中)而不是阻止缓冲的。 如果客户太
跟上它的消息很慢，它的订阅被终止:

for each subscription {
    out := subscription.outChan
    if cap(out) == 0 {
        // block on unbuffered channel
        out <- msg
    } else {
        // don't block on buffered channels
        select {
            case out <- msg:
            default:
                // set the error, notify on the cancel chan
                subscription.err = fmt.Errorf("client is too slow for msg)
                close(subscription.cancelChan)

                // ... unsubscribe and close out
        }
    }
}

### 这种新设计如何解决当前的问题？

[#951]([#1880]):

由于非阻塞发送，我们不会死锁的情况
了。如果客户端停止读取消息，它将被删除。

[#1879]:

现在使用 MsgAndTags 代替普通消息。

### 未来的问题及其可能的解决方案

[#2826]

我还在思考的一个问题:如何防止pubsub变慢
下共识。我们可以增加发布订阅队列的大小(现在是 0)。还，
限制订阅者总数可能是个好主意。

这可以自动进行。假设我们将队列大小设置为 1000，并且当它 >=
80% 已满，拒绝新订阅。

## 状态

实施的

## 结果

### 积极的

- 更惯用的界面
- 订阅者知道 msg 是用什么标签发布的
- 订阅者知道他们的订阅被取消的原因

### 消极的

-(自 v1 起)在发布消息时没有并发

### 中性的


[#951]:https://github.com/tendermint/tendermint/issues/951
[#1879]:https://github.com/tendermint/tendermint/issues/1879
[#1880]:https://github.com/tendermint/tendermint/issues/1880
[#2826]:https://github.com/tendermint/tendermint/issues/2826
