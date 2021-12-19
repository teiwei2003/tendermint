# ADR 044:具有弱主观性的轻客户端

## 变更日志
* 13-07-2019:初稿
* 14-08-2019:地址 cwgos 评论

## 语境

比特币白皮书中引入了轻客户端的概念。它
描述了一个分布式共识过程的观察者，它只验证
共识算法而不是其中的状态机交易。

Tendermint 轻客户端允许带宽和计算受限的设备，例如智能手机、低功耗嵌入式芯片或其他区块链
有效验证 Tendermint 区块链的共识。这形成了
新网络节点的安全高效状态同步的基础和
区块链间通信(其中一个 Tendermint 实例的轻客户端
在另一个链的状态机中运行)。

在一个预期会可靠地惩罚验证者的不当行为的网络中
通过削减保税权益和验证人集更改的地方
很少，客户可以利用这个假设来安全地
无需下载中间标头即可同步 lite 客户端。

在权益证明环境中运行的轻客户端(和全节点)需要一个
来自可信来源的可信块高度不超过 1 个解除绑定
窗口加上可配置的证据提交同步绑定。这就是所谓的“弱主观性”。

股权证明区块链需要弱主观性，因为它是
攻击者可以免费购买不再绑定的投票密钥
在其先前历史的某个时刻分叉网络。见 Vitalik 的帖子:
[股权证明:我如何学会爱弱者
主观性](https://blog.ethereum.org/2014/11/25/proof-stake-learned-love-weak-subjectivity/)。

目前，Tendermint 在
[light](https://github.com/tendermint/tendermint/tree/master/light) 包。这
lite 客户端实现了尝试使用二分搜索的二分算法
找到验证者设置投票的最小区块头数
功率变化小于 < 1/3。此接口不支持弱
此时的主观性。 Cosmos SDK 也不支持反事实
削减，lite 客户端也没有任何能力报告证据制作
这些系统*理论上不安全*。

注意:Tendermint 提供了一个稍微不同(更强)的轻客户端模型
比日食下的比特币，因为日食节点只能愚弄光
客户端，如果他们拥有来自最后一个信任根的三分之二的私钥。

## 决定

### 弱主观性接口

添加新轻客户端连接时的弱主观性接口
网络或当轻客户端离线时间超过
解绑期连接到网络。具体来说，节点需要
在从用户输入同步之前初始化以下结构:

```
type TrustOptions struct {
    // Required: only trust commits up to this old.
    // Should be equal to the unbonding period minus some delta for evidence reporting.
    TrustPeriod time.Duration `json:"trust-period"`

    // Option 1: TrustHeight and TrustHash can both be provided
    // to force the trusting of a particular height and hash.
    // If the latest trusted height/hash is more recent, then this option is
    // ignored.
    TrustHeight int64  `json:"trust-height"`
    TrustHash   []byte `json:"trust-hash"`

    // Option 2: Callback can be set to implement a confirmation
    // step if the trust store is uninitialized, or expired.
    Callback func(height int64, hash []byte) error
}
```

期望用户将从可信来源获得此信息
例如验证者、朋友或安全网站。更人性化
具有信任权衡的解决方案是我们建立一个基于 https 的协议
填充此信息的默认端点。也是一个链上注册表
信任根(例如在 Cosmos Hub 上)似乎在未来很可能出现。

### 线性验证

线性验证算法需要下载所有标头
在 `TrustHeight` 和 `LatestHeight` 之间。 lite客户端下载
提供的“TrustHeight”的完整标头，然后继续下载“N+1”
标头并应用 [Tendermint 验证
规则](https://docs.tendermint.com/master/spec/blockchain/blockchain.html#validation)
到每个块。

### 二等分验证

二分验证是一种带宽和计算密集型机制，
在最乐观的情况下需要一个轻客户端只下载两个块
头进入同步。

二分算法以下列方式进行。客户端下载
并验证“TrustHeight”的完整区块头，然后获取
`LatestHeight` 拦截器标题。客户端然后验证“最新高度”
标题。最后，客户端尝试使用以下方法验证“LatestHeight”标头
从 `TrustHeight` 标头中的 `NextValidatorSet` 获得投票权。这
如果来自 `TrustHeight` 的验证器仍然有 > 2/3，则验证将成功
在“最新高度”中获得 +1 的投票权。如果成功，则客户端完全
同步。如果失败，则应遵循二分算法
执行。

客户端尝试在中间的块中下载块
`LatestHeight` 和 `TrustHeight` 并尝试与上述相同的算法
使用 `MidPointHeight` 而不是 `LatestHeight` 和不同的阈值 -
*非相邻标题*的1/3 +1投票权。在失败的情况下，
递归执行`MidPoint`验证直到成功然后重新开始
具有更新的“NextValidatorSet”和“TrustHeight”。

如果客户端遇到伪造的标头，它应该一起提交标头
与其他一些中间标题作为对其他人不当行为的证据
全节点。之后，它可以使用另一个完整节点重试二分。一个
最佳客户端将缓存来自上次运行的可信标头以最小化
网络使用。

---

查看正式规范
[这里](https://github.com/tendermint/spec/tree/master/spec/light-client)。

## 状态

实施的

## 结果

### 积极的

* 安全使用的轻客户端(它可以离线，但不能太长时间)

### 消极的

* 对分的复杂性

### 中性的

* 社会共识可能容易出错(对于新的轻客户端的情况
  加入网络或离线时间过长)
