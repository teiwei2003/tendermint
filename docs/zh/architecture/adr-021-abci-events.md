# ADR 012:ABCI 事件

## 变更日志

- *2018-09-02* 删除 ABCI 错误组件.更新事件描述
- *2018-07-12* 初始版本

## 语境

ABCI 标签首先在 [ADR 002](https://github.com/tendermint/tendermint/blob/master/docs/architecture/adr-002-event-subscription.md) 中描述.
它们是可用于索引交易的键值对.

目前，ABCI 消息返回一个标签列表来描述一个
在 Check/DeliverTx/Begin/EndBlock 期间发生的“事件”，
其中每个标签指的是事件的不同属性，例如发送和接收帐户地址.

由于只有一个标签列表，记录多个此类事件的数据
必须使用密钥中的前缀完成单个 Check/DeliverTx/Begin/EndBlock
空间.

或者，构成事件的标签组可以用
表示事件之间中断的特殊标记.这将允许
将多个事件直接编码到单个标签列表中，无需
前缀，以这些“特殊”标签为代价来分隔不同的事件.

TODO:索引工作原理的简要说明

## 决定

不是返回标签列表，而是返回事件列表，其中
每个事件都是一个标签列表.这样我们自然而然地捕捉到了
在单个 ABCI 消息期间发生的多个事件.

TODO:描述对索引和查询的影响

## 状态

实施的

## 结果

### 积极的

- 能够跟踪与 ABCI 调用分开的不同事件(DeliverTx/BeginBlock/EndBlock)
- 更强大的查询能力

### 消极的

- 更复杂的查询语法
- 更复杂的搜索实现

### 中性的
