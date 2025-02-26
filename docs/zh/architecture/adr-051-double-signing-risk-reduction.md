# ADR 051:双重签名风险降低

## 变更日志

* 27-11-2019:初稿
* 13-01-2020:分成2个ADR，这个ADR只涵盖双重签名保护和ADR-052处理Tendermint模式
* 22-01-2020:将标题从“双签保护”改为“双签风险降低”

## 语境

为验证者错误执行的双重签名事件提供降低风险的方法
- 验证人经常错误地运行重复的验证人导致双重签名事件
- 此提议的功能是通过在投票开始前检查最近的 N 个区块来降低错误双重签名事件的风险
- 考虑到双重签名事件的严重影响，在node daemon中内置多重风险降低算法是非常合理的

## 决定

我们想建议一种双重签名风险降低方法.

- 方法论:查询最近的共识结果，以了解最近是否使用节点的共识密钥进行共识
- 何时检查
    - 当状态机在完全同步后启动`ConsensusReactor`
    - 当节点是验证器时(带有 privValidator )
    - 当`cs.config.DoubleSignCheckHeight > 0`
- 如何检查
    1. 当验证者从同步状态转变为完全同步状态时，状态机使用验证者的共识密钥检查最近的N个区块(`latest_height - double_sign_check_height`)以找出是否存在共识投票
    2. 如果验证者的共识密钥存在投票，退出状态机程序
- 配置
    - 我们想通过在 `config.toml` 和 cli 中引入 `double_sign_check_height` 参数来建议，有多少块状态机回顾检查投票
    - <span v-pre>`double_sign_check_height = {{ .Consensus.DoubleSignCheckHeight }}`</span> in `config.toml`
    - cli中的`tendermint节点--consensus.double_sign_check_height`
    - 当“double_sign_check_height == 0”时，状态机忽略检查过程

## 状态

实施的

## 结果

### 积极的

- 验证者可以避免错误的双重签名事件. (例如，如果另一个验证器节点正在对共识进行投票，则使用相同的共识密钥启动新的验证器节点将导致状态机恐慌停止，因为在最近的区块中发现了使用该共识密钥的共识投票)
- 我们希望这种方法可以防止大多数错误的双重签名事件.

### 消极的

- 当风险降低方法开启时，重新启动验证器节点会发生恐慌，因为节点本身使用相同的共识密钥投票达成共识.因此，验证者应该停止状态机，等待一些区块，然后重新启动状态机以避免恐慌停止.

### 中性的

## 参考

- 问题 [#4059](https://github.com/tendermint/tendermint/issues/4059):双重签名保护
