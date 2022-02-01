# ADR 052:Tendermint 模式

## 变更日志

* 27-11-2019:来自 ADR-051 的初稿
* 13-01-2020:将 ADR Tendermint 模式与 ADR-051 分开
* 29-03-2021:更新有关默认值的信息

## 语境

- 完整模式:完整模式没有成为验证者的能力.
- 验证器模式:此模式与现有状态机行为完全相同.同步不投票共识，完全同步时参与共识
- 种子模式:轻量级种子节点维护地址簿，p2p 类似于 [TenderSeed](https://gitlab.com/polychainlabs/tenderseed)

## 决定

我们想建议一个简单的 Tendermint 模式抽象.这些模式将存在于一个二进制文件中，并且在初始化节点时，用户将能够指定他们想要创建的节点.

- 每个节点包含哪个反应器、组件
    - 满的
        - 开关，运输
        - 反应堆
          - 内存池
          - 共识
          - 证据
          - 区块链
          - p2p/pex
          - 状态同步
        - rpc(仅限安全连接)
        - *~~没有privValidator(priv_validator_key.json, priv_validator_state.json)~~*
    - 验证器
        - 开关，运输
        - 反应堆
          - 内存池
          - 共识
          - 证据
          - 区块链
          - p2p/pex
          - 状态同步
        - rpc(仅限安全连接)
        - 使用 privValidator(priv_validator_key.json, priv_validator_state.json)
    - 种子
        - 开关，运输
        - 反应堆
           - p2p/pex
- 配置，cli 命令
    - 我们建议通过在 `config.toml` 和 cli 中引入 `mode` 参数
    - <span v-pre>`mode = "{{ .BaseConfig.Mode }}"`</span> in `config.toml`
    - cli中的`tendermint start --mode验证器`
    - 全|验证器 |种子节点
    - 不会有默认值.用户需要指定何时运行 `tendermint init`
- RPC修改
    -`主机:26657/状态`
        - 在完整模式下返回空的`validator_info`
    - 种子节点中没有 rpc 服务器
- 在代码库中修改的位置
    - 在`node/node.go:DefaultNewNode`上为`config.Mode`添加开关
    - 如果`config.Mode==validator`，调用默认的`NewNode`(当前逻辑)
    - 如果`config.Mode==full`，调用`NewNode` 和`nil` `privValidator`(不加载或生成)
        - 需要将`nil``privValidator`的异常例程添加到相关函数中
    - 如果`config.Mode==seed`，调用`NewSeedNode`(`node/node.go:NewNode` 的种子节点版本)
        - 需要为`nil` `reactor`, `component` 添加异常例程到相关函数中

## 状态

实施的

## 结果

### 积极的

- 节点操作者可以根据节点的用途选择运行状态机时的模式.
- 模式可以防止错误，因为用户必须通过标志指定他们想要运行的模式. (例如，如果用户想要运行验证器节点，她/他应该明确地将验证器写为模式)
- 不同的模式需要不同的反应器，从而实现高效的资源利用.

### 消极的

- 用户需要研究每种模式如何运作以及它具有哪些能力.

### 中性的

## 参考

- 问题 [#2237](https://github.com/tendermint/tendermint/issues/2237):Tendermint“模式”
- [TenderSeed](https://gitlab.com/polychainlabs/tenderseed):一个轻量级的 Tendermint 种子节点.
