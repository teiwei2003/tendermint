# ADR 011:监控

## 变更日志

08-06-2018:初稿
11-06-2018:@xla 评论后的重组
13-06-2018:澄清标签的使用

## 语境

为了提高 Tendermint 的知名度，我们希望它报告
指标，也许以后还有交易和 RPC 查询的痕迹.看
https://github.com/tendermint/tendermint/issues/986.

考虑了几种解决方案:

1.[普罗米修斯](https://prometheus.io)
   a) 普罗米修斯 API
   b) [go-kit metrics 包](https://github.com/go-kit/kit/tree/master/metrics) 作为接口加上 Prometheus
   c) [电报](https://github.com/influxdata/telegraf)
   d) 新服务，它将监听 pubsub 发出的事件并报告指标
2. [OpenCensus](https://opencensus.io/introduction/)

### 1. 普罗米修斯

Prometheus 似乎是最流行的监控产品.它有
一个 Go 客户端库，强大的查询，警报.

**a) 普罗米修斯 API**

我们可以承诺在 Tendermint 中使用 Prometheus，但我认为 Tendermint 用户
应该可以自由选择他们认为更适合的任何监控工具
他们的需求(如果他们还没有现有的需求).所以我们应该尝试
足够抽象的接口，以便人们可以在 Prometheus 和其他
类似的工具.

**b) 作为接口的 go-kit 指标包**

metrics 包为服务提供了一组统一的接口
检测并为流行的指标包提供适配器:

https://godoc.org/github.com/go-kit/kit/metrics#pkg-subdirectories

与 Prometheus API 相比，我们失去了可定制性和控制力，但获得了
鉴于我们将提取，从上述列表中选择任何乐器的自由
指标创建到一个单独的函数中(参见 node/node.go 中的“提供者”).

**c) 电报**

与已经讨论过的选项不同，telegraf 不需要修改 Tendermint
源代码.你创建了一个叫做输入插件的东西，它轮询
Tendermint RPC 每秒执行一次并计算指标本身.

虽然听起来不错，但我们想要报告的一些指标并未通过
RPC 或 pubsub，因此无法从外部访问.

**d) 服务，收听 pubsub**

与上述相同的问题.

### 2. 开放人口普查

opencensus 提供度量和跟踪，这在
未来.它的 API 看起来与 go-kit 和 Prometheus 不同，但看起来很像
涵盖了我们需要的一切.

不幸的是，OpenCensus go 客户端没有定义任何
接口，所以如果我们想抽象出指标，我们
需要自己编写接口.

### 指标列表

|     | Name                                 | Type   | Description                                                                   |
| --- | ------------------------------------ | ------ | ----------------------------------------------------------------------------- |
| A   | consensus_height                     | Gauge  |                                                                               |
| A   | consensus_validators                 | Gauge  | Number of validators who signed                                               |
| A   | consensus_validators_power           | Gauge  | Total voting power of all validators                                          |
| A   | consensus_missing_validators         | Gauge  | Number of validators who did not sign                                         |
| A   | consensus_missing_validators_power   | Gauge  | Total voting power of the missing validators                                  |
| A   | consensus_byzantine_validators       | Gauge  | Number of validators who tried to double sign                                 |
| A   | consensus_byzantine_validators_power | Gauge  | Total voting power of the byzantine validators                                |
| A   | consensus_block_interval             | Timing | Time between this and last block (Block.Header.Time)                          |
|     | consensus_block_time                 | Timing | Time to create a block (from creating a proposal to commit)                   |
|     | consensus_time_between_blocks        | Timing | Time between committing last block and (receiving proposal creating proposal) |
| A   | consensus_rounds                     | Gauge  | Number of rounds                                                              |
|     | consensus_prevotes                   | Gauge  |                                                                               |
|     | consensus_precommits                 | Gauge  |                                                                               |
|     | consensus_prevotes_total_power       | Gauge  |                                                                               |
|     | consensus_precommits_total_power     | Gauge  |                                                                               |
| A   | consensus_num_txs                    | Gauge  |                                                                               |
| A   | mempool_size                         | Gauge  |                                                                               |
| A   | consensus_total_txs                  | Gauge  |                                                                               |
| A   | consensus_block_size                 | Gauge  | In bytes                                                                      |
| A   | p2p_peers                            | Gauge  | Number of peers node's connected to                                           |

`A` - 将首先实施.

**提议的解决方案**

## 状态

实施的

## 结果

### 积极的

更好的可见性，支持各种监控后端

### 消极的

另一个用于审计的库，将指标报告代码与业务领域混淆.

### 中性的

——
