# 什么是Tendermint

Tendermint 是一种用于安全且一致地复制
多台机器上的应用.通过安全，我们的意思是 Tendermint 有效
即使多达 1/3 的机器以任意方式出现故障.一直以来，
我们的意思是每台没有故障的机器都会看到相同的事务日志，并且
计算相同的状态.安全一致的复制是一个
分布式系统中的基本问题；它在
广泛的应用程序的容错能力，从货币，
选举、基础设施编排等.

容忍机器以任意方式失败的能力，包括
变得恶意，被称为拜占庭容错(BFT).这
BFT 理论已有数十年历史，但软件实现只有
最近流行起来，主要是因为“区块链”的成功
技术”，如比特币和以太坊.区块链技术只是一种
在更现代的环境中重新调整 BFT，重点是
对等网络和加密身份验证.名字
源自交易在块中批处理的方式，其中每个
块包含前一个的加密哈希，形成一个
链.在实践中，区块链数据结构实际上优化了 BFT
设计.

Tendermint 由两个主要技术组件组成:一个区块链
共识引擎和通用应用程序接口.共识
引擎，称为 Tendermint Core，可确保相同的交易
以相同的顺序记录在每台机器上.应用界面，
称为应用程序区块链接口 (ABCI)，使
以任何编程语言处理的事务.不同于其他
区块链和共识解决方案，预先打包内置
在状态机中(比如花哨的键值存储，或古怪的脚本
语言)，开发者可以使用 Tendermint 作为 BFT 状态机
复制以任何编程语言编写的应用程序和
开发环境适合他们.

Tendermint 旨在易于使用、易于理解、高度
性能好，适用于各种分布式应用程序.

## Tendermint 与 X

Tendermint 大致类似于两类软件.首先
类由分布式键值存储组成，如 Zookeeper、etcd、
和 consul，它们使用非 BFT 共识.第二类被称为
“区块链技术”，由两种加密货币组成，如
比特币和以太坊，以及替代的分布式账本设计，如
Hyperledger 的洞穴.

### Zookeeper、etcd、consul

Zookeeper、etcd 和 consul 都是键值存储的实现
在经典的非 BFT 共识算法之上. Zookeeper 使用一个版本
Paxos 的称为 Zookeeper Atomic Broadcast，而 etcd 和 consul 使用
Raft 共识算法，它更年轻、更简单.一个
典型集群包含 3-5 台机器，并且可以容忍崩溃故障
在多达 1/2 的机器中，但即使是单个拜占庭故障也可以
破坏系统.

每个产品都提供了一个稍微不同的实现
功能强大的键值存储，但通常都集中在
为分布式系统提供基本服务，例如动态
配置、服务发现、锁定、领导选举等.

Tendermint 本质上是类似的软件，但有两个主要区别:

- 它是拜占庭容错的，这意味着它最多只能容忍
  1/3 的失败，但这些失败可以包括任意行为 -
  包括黑客攻击和恶意攻击.
- 它没有指定特定的应用程序，如花哨的键值
  店铺.相反，它专注于任意状态机复制，
  因此开发人员可以构建适合他们的应用程序逻辑，
  从键值存储到加密货币再到电子投票平台等等.

### 比特币、以太坊等

Tendermint 出现在像比特币这样的加密货币的传统中，
以太坊等，旨在提供更高效、更安全的
共识算法优于比特币的工作量证明.在早些时候，
Tendermint 内置了一个简单的货币，可以参与
共识，用户必须将货币单位“绑定”为证券
如果他们行为不端，可以撤销押金 - 这就是
Tendermint 一种权益证明算法.

从那时起，Tendermint 已经发展成为一个通用的区块链
可以托管任意应用程序状态的共识引擎.这意味着
它可以用作共识引擎的即插即用替代品
其他区块链软件.所以可以使用当前的以太坊代码
基础，无论是在 Rust、Go 还是 Haskell 中，并将其作为 ABCI 运行
使用 Tendermint 共识的应用程序.事实上，[我们用
以太坊](https://github.com/cosmos/ethermint).我们计划做
比特币、ZCash 和其他各种确定性的
应用程序也是如此.

另一个基于 Tendermint 的加密货币应用程序示例是
[Cosmos 网络](http://cosmos.network).

### 其他区块链项目

[Fabric](https://github.com/hyperledger/fabric) 采取了类似的方法
对 Tendermint，但对如何管理状态更有意见，
并要求所有应用程序行为可能在许多
docker 容器，它称为“链码”的模块.它使用一个
[PBFT] 的实施(http://pmg.csail.mit.edu/papers/osdi99.pdf).
来自 IBM 的一个团队，该团队[增强以处理潜在的
不确定的
链码](https://www.zurich.ibm.com/~cca/papers/sieve.pdf) 是
可以将这种基于 docker 的行为实现为 ABCI 应用程序
Tendermint，虽然扩展了 Tendermint 以处理非确定性
留待以后的工作.

[Burrow](https://github.com/hyperledger/burrow) 是一个实现
以太坊虚拟机和以太坊交易机制，与
名称注册、权限和本机的附加功能
合约，以及替代的区块链 API.它使用 Tendermint 作为其
共识引擎，并提供特定的应用程序状态.

## ABCI 概述

[应用程序区块链接口
(ABCI)](https://github.com/tendermint/tendermint/tree/master/abci)
允许应用程序的拜占庭容错复制
用任何编程语言编写.

### 动机

到目前为止，所有区块链“堆叠”(例如
[比特币](https://github.com/bitcoin/bitcoin)) 有一个整体
设计.也就是说，每个区块链堆栈都是一个处理
去中心化账本的所有问题；这包括 P2P
连接性、交易的“内存池”广播、共识
最近的区块，账户余额，图灵完备合约，
用户级权限等

在计算机中使用单体架构通常是不好的做法
科学.这使得重用代码的组件变得困难，并且
尝试这样做会导致前叉的复杂维护程序
代码库.当代码库不是模块化时尤其如此
在设计中并遭受“意大利面条式代码”的困扰.

单体设计的另一个问题是它限制了你
区块链堆栈的语言(反之亦然).如果是
以太坊支持图灵完备的字节码虚拟机，它
将您限制为编译为该字节码的语言；今天，那些
是 Serpent 和 Solidity.

相比之下，我们的方法是将共识引擎和 P2P 解耦
从特定应用状态的细节层
区块链应用.我们通过抽象出细节来做到这一点
应用程序到一个接口，它被实现为一个套接字
协议.

因此我们有一个接口，应用程序区块链接口(ABCI)，
及其主要实现，Tendermint 套接字协议(TSP，或
茶匙).

### ABCI 介绍

[Tendermint 核心](https://github.com/tendermint/tendermint)(
“共识引擎”)通过套接字与应用程序通信
满足 ABCI 的协议.

打个比方，让我们谈谈一个著名的加密货币，
比特币.比特币是一个加密货币区块链，其中每个节点
维护一个经过全面审计的未花费交易输出 (UTXO) 数据库.如果
有人想在 ABCI、Tendermint 之上创建一个类似比特币的系统
核心将负责

- 在节点之间共享区块和交易
- 建立规范/不可变的交易顺序
  (区块链)

该应用程序将负责

- 维护UTXO数据库
- 验证交易的加密签名
- 防止交易花费不存在的交易
- 允许客户端查询 UTXO 数据库.

Tendermint 能够通过提供非常
应用程序和共识之间的简单 API(即 ABCI)
过程.

ABCI 由 3 种主要消息类型组成，这些消息类型从
应用的核心.应用程序回复相应的
响应消息.

消息在此处指定:[ABCI 消息
类型](https://github.com/tendermint/tendermint/blob/master/abci/README.md#message-types).

**DeliverTx** 消息是应用程序的工作马.每个
区块链中的交易与此消息一起传递.这
应用程序需要验证收到的每笔交易
**DeliverTx** 针对当前状态、应用协议、
以及交易的加密凭证.一个经过验证的
事务然后需要更新应用程序状态——通过绑定一个
value 到一个键值存储中，或者通过更新 UTXO 数据库，用于
实例.

**CheckTx** 消息类似于 **DeliverTx**，但它仅用于
验证交易. Tendermint Core 的内存池首先检查
**CheckTx** 交易的有效性，并且仅中继有效
与其同行进行交易.例如，一个应用程序可能会检查一个
在事务中递增序列号并返回错误
**CheckTx** 如果序列号是旧的.或者，他们可能会使用
基于能力的系统，需要更新能力
每笔交易.

**Commit** 消息用于计算对
当前应用程序状态，将被放入下一个区块头.
这有一些方便的特性.更新该状态的不一致
现在将显示为区块链分叉，它捕获了一整类
编程错误.这也简化了安全开发
轻量级客户端，因为 Merkle-hash 证明可以通过检查来验证
针对区块哈希，并且区块哈希由法定人数签名.

一个应用程序可以有多个 ABCI 套接字连接.
Tendermint Core 为应用程序创建了三个 ABCI 连接；一
用于在内存池中广播时验证交易，一个
用于运行区块提案的共识引擎，还有一个用于
查询应用程序状态.

很明显，应用程序设计人员需要非常小心地
设计他们的消息处理程序来创建一个可以做任何事情的区块链
有用，但这种架构提供了一个起点.图表
下面说明了通过 ABCI 的消息流.

![abci](../../imgs/abci.png)

## 关于决定论的说明

区块链交易处理的逻辑必须是确定性的.
如果应用程序逻辑不是确定性的，共识就不会
到达 Tendermint Core 副本节点之间.

以太坊上的 Solidity 是区块链的绝佳选择语言
应用程序，因为除其他原因外，它是一个完全
确定性编程语言.但是，也可以
使用现有的流行语言创建确定性应用程序，例如
Java、C++、Python 或 Go.游戏程序员和区块链开发者是
已经熟悉通过避免创建确定性程序
不确定性的来源，例如:

- 随机数生成器(没有确定性种子)
- 线程上的竞争条件(或完全避免线程)
- 系统时钟
- 未初始化的内存(在不安全的编程语言中，如 C
  或 C++)
- [浮点
  算术](http://gafferongames.com/networking-for-game-programmers/floating-point-determinism/)
- 随机的语言特征(例如 Go 中的地图迭代)

虽然程序员可以通过小心避免不确定性，但它也是
可以为每种语言创建一个特殊的 linter 或静态分析器
检查确定性.未来我们可能会与合作伙伴合作，
创建这样的工具.

## 共识概述

Tendermint 是一种易于理解的、主要是异步的、BFT 共识
协议.该协议遵循一个简单的状态机，看起来像
这:

![共识逻辑](../../imgs/consensus_logic.png)

协议的参与者被称为**验证者**；他们轮流
提出交易区块并对其进行投票.块是
在链中提交，每个 ** 高度** 一个块.一个块可能
未能提交，在这种情况下，协议移动到下一个
**round**，并且新的验证器可以为该高度提出一个块.
成功提交一个区块需要两个阶段的投票；我们
称它们为 **pre-vote** 和 **pre-commit**.当一个块被提交时
超过 2/3 的验证者在同一个区块中预先提交
圆形的.

有一张照片是一对夫妇在做波尔卡舞，因为验证器是
做一些像波尔卡舞这样的事情.当超过三分之二
验证者对同一个区块进行预投票，我们称之为 **polka**.每一个
预提交必须由同一轮中的波尔卡舞证明.

由于多种原因，验证者可能无法提交区块；这
当前提议者可能离线，或者网络可能很慢.嫩肤
允许他们确定应该跳过验证器.验证器
等待一小段时间从
投票前的提议者进入下一轮.这种依赖
超时是使 Tendermint 成为弱同步协议的原因，而不是
比异步的.但是，协议的其余部分是
异步，验证者只有在听取更多意见后才能取得进展
超过验证器集的三分之二.一个简化的元素
Tendermint 是它使用相同的机制来提交一个块
跳到下一轮.

假设不到三分之一的验证者是拜占庭、Tendermint
保证永远不会违反安全性——也就是说，验证者将
永远不要在同一高度提交冲突块.要做到这一点
引入了一些**锁定**规则，这些规则可以调节哪些路径可以
接着是流程图.一旦验证者预提交了一个区块，
锁定在那个街区.然后，

1.它必须对它被锁定的块进行投票
2.它只能解锁，并预提交一个新块，如果有
    在后面的回合中为那个街区打波尔卡

## 股权

在许多系统中，并非所有验证器都具有相同的“权重”
共识协议.因此，我们对三分之一或
三分之二的验证者，但在总数的这些比例中
投票权，可能不会在个人之间均匀分配
验证器.

由于 Tendermint 可以复制任意应用程序，因此可以
定义一种货币，并以该货币计价投票权.
当投票权以本国货币计价时，系统是
通常称为权益证明.可以通过逻辑强制验证器
在申请中，将他们持有的货币“绑定”在证券中
如果发现他们行为不端，可以销毁的存款
共识协议.这为网络的安全性增加了经济因素
协议，允许量化违反假设的成本
不到三分之一的投票权是拜占庭的.

[Cosmos Network](https://cosmos.network) 就是为了使用这个
跨一系列加密货币实施的权益证明机制
作为 ABCI 应用程序.

下图是 Tendermint 的(技术)简而言之.

![tx-flow](../../imgs/tm-transaction-flow.png)
