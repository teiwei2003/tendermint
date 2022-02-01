# ADR 061:P2P 重构范围

## 变更日志

- 2020 年 10 月 30 日:初始版本 (@erikgrinaker)

## 语境

负责点对点网络的“p2p”包相当陈旧，有许多弱点，包括紧耦合、抽象泄漏、缺乏测试、DoS 漏洞、性能不佳、自定义协议和不正确的行为.重构已经讨论了好几年([#2067](https://github.com/tendermint/tendermint/issues/2067)).

Informal Systems 也在构建 Tendermint 的 Rust 实现，[Tendermint-rs](https://github.com/informalsystems/tendermint-rs)，并计划在明年实现 P2P 网络支持.作为这项工作的一部分，他们要求采用例如[QUIC](https://datatracker.ietf.org/doc/draft-ietf-quic-transport/) 作为传输协议，而不是实现 Tendermint 当前使用的自定义应用程序级`MConnection` 流复用协议.

本 ADR 总结了最近与利益相关者关于 P2P 重构范围的讨论.具体设计和实施将作为单独的 ADR 提交.

## 替代方法

已经反复出现采用 [LibP2P](https://libp2p.io) 而不是维护我们自己的 P2P 网络堆栈的建议(参见 [#3696](https://github.com/tendermint/tendermint/issues/3696) ).虽然这在原则上似乎是一个好主意，但这将是一个高度破坏性的协议更改，有迹象表明我们可能不得不分叉和修改 LibP2P，并且对使用的抽象存在担忧.

在与 Informal Systems 的讨论中，我们决定从对当前 P2P 堆栈的增量改进开始，添加对可插拔传输的支持，然后逐渐开始尝试将 LibP2P 作为传输层.如果这证明是成功的，我们可以考虑稍后将其用于更高级别的组件.

## 决定

P2P 堆栈将分几个阶段迭代重构和改进:

* **第一阶段:** 代码和API重构，尽可能保持协议兼容性.

* **阶段 2:** 额外的传输和增量协议改进.

* **第 3 阶段:** 破坏性协议更改.

第 2 和第 3 阶段的范围仍然不确定，一旦前面的阶段完成，我们将重新审视，因为我们将对需求和挑战有更好的认识.

## 详细设计

在研究和原型设计之后，将针对每个阶段的特定设计和更改提交单独的 ADR.以下是按优先顺序排列的目标.

### 阶段 1:代码和 API 重构

此阶段将专注于改进 p2p 包中的内部抽象和实现.尽可能不以向后不兼容的方式改变 P2P 协议.

* 更清晰、解耦的抽象，例如`Reactor`、`Switch` 和 `Peer`. [#2067](https://github.com/tendermint/tendermint/issues/2067) [#5287](https://github.com/tendermint/tendermint/issues/5287) [#3833](https:// /github.com/tendermint/tendermint/issues/3833)
    * Reactor 应该在单独的 goroutine 中或通过缓冲通道接收消息. [#2888](https://github.com/tendermint/tendermint/issues/2888)
* 改进了对等生命周期管理. [#3679](https://github.com/tendermint/tendermint/issues/3679) [#3719](https://github.com/tendermint/tendermint/issues/3719) [#3653](https:// /github.com/tendermint/tendermint/issues/3653) [#3540](https://github.com/tendermint/tendermint/issues/3540) [#3183](https://github.com/tendermint/tendermint /issues/3183) [#3081](https://github.com/tendermint/tendermint/issues/3081) [#1356](https://github.com/tendermint/tendermint/issues/1356)
    * 同行优先. [#2860](https://github.com/tendermint/tendermint/issues/2860) [#2041](https://github.com/tendermint/tendermint/issues/2041)
* 可插拔传输，以`MConnection` 作为一种实现. [#5587](https://github.com/tendermint/tendermint/issues/5587) [#2430](https://github.com/tendermint/tendermint/issues/2430) [#805](https:// /github.com/tendermint/tendermint/issues/805)
* 改进的对等地址处理.
    * 地址簿重构. [#4848](https://github.com/tendermint/tendermint/issues/4848) [#2661](https://github.com/tendermint/tendermint/issues/2661)
    * 与传输无关的对等寻址. [#5587](https://github.com/tendermint/tendermint/issues/5587) [#3782](https://github.com/tendermint/tendermint/issues/3782) [#3692](https:// /github.com/tendermint/tendermint/issues/3692)
    * 改进了对自己地址的检测和广告. [#5588](https://github.com/tendermint/tendermint/issues/5588) [#4260](https://github.com/tendermint/tendermint/issues/4260) [#3716](https:// /github.com/tendermint/tendermint/issues/3716) [#1727](https://github.com/tendermint/tendermint/issues/1727)
    * 每个对等点支持多个 IP. [#1521](https://github.com/tendermint/tendermint/issues/1521) [#2317](https://github.com/tendermint/tendermint/issues/2317)

重构应该尝试解决以下次要目标:可测试性、可观察性、性能、安全性、服务质量、背压和 DoS 弹性.其中大部分将作为第 2 阶段的明确目标重新审视.

理想情况下，重构应该逐步进行，每隔几周定期合并到“master”.总体而言，这将花费更多时间，并导致内部 Go API 频繁发生重大更改，但它减少了分支漂移并使代码更快、更广泛地测试.

### 阶段 2:额外的传输和协议改进

此阶段将侧重于协议改进和其他重大更改.以下是重构完成后需要单独评估的建议.在第 1 阶段可能会添加其他提案.

* QUIC 传输. [#198](https://github.com/tendermint/spec/issues/198)
* 秘密连接握手的噪声协议. [#5589](https://github.com/tendermint/tendermint/issues/5589) [#3340](https://github.com/tendermint/tendermint/issues/3340)
* 连接握手中的对等 ID. [#5590](https://github.com/tendermint/tendermint/issues/5590)
* 对等点和服务发现(例如 RPC 节点、状态同步快照). [#5481](https://github.com/tendermint/tendermint/issues/5481) [#4583](https://github.com/tendermint/tendermint/issues/4583)
* 速率限制、背压和 QoS 调度. [#4753](https://github.com/tendermint/tendermint/issues/4753) [#2338](https://github.com/tendermint/tendermint/issues/2338)
* 压缩. [#2375](https://github.com/tendermint/tendermint/issues/2375)
* 改进的指标和跟踪. [#3849](https://github.com/tendermint/tendermint/issues/3849) [#2600](https://github.com/tendermint/tendermint/issues/2600)
* 简化的 P2P 配置选项.

### 第 3 阶段:破坏性协议更改

此阶段涵盖了定义不明确且高度不确定的投机性、影响广泛的提案.一旦前几个阶段完成，它们将被评估.

* 采用 LibP2P. [#3696](https://github.com/tendermint/tendermint/issues/3696)
* 允许跨反应器通信，可能没有通道.
* 动态频道广告，因为反应器已启用/禁用. [#4394](https://github.com/tendermint/tendermint/issues/4394) [#1148](https://github.com/tendermint/tendermint/issues/1148)
* 发布订阅式网络拓扑和模式.
* 支持同一网络中的多个链ID.

## 状态

公认

## 结果

### 积极的

* 更简洁、更简单的架构，更容易推理和测试，因此希望减少错误.

* 改进的性能和稳健性.

* 通过可能采用 QUIC 和 Noise 等标准化协议，减少维护负担并提高互操作性.

* 提高可用性，具有更好的可观察性、更简单的配置和更多的自动化(例如对等/服务/地址发现、速率限制和背压).

### 消极的

* 维护我们自己的 P2P 网络堆栈是资源密集型的.

* 抽象掉底层传输可能会阻止使用高级传输功能.

* 对 API 和协议的重大更改会对用户造成破坏.

## 参考

请参阅上面的问题链接.

- [#2067: P2P 重构](https://github.com/tendermint/tendermint/issues/2067)

- [P2P 重构头脑风暴文档](https://docs.google.com/document/d/1FUTADZyLnwA9z7ndayuhAdAFRKujhh_y73D0ZFdKiOQ/edit?pli=1#)
