# 共识

Tendermint 共识是一种分布式协议，由验证器进程执行以达成一致
要添加到 Tendermint 区块链的下一个区块.该协议按轮次进行，其中
每一轮都是试图就下一个区块达成协议.一轮开始时有一个专门的
进程(称为提议者)向其他进程建议下一个块应该是什么
`ProposalMessage`.
进程通过投票给一个带有 `VoteMessage` 的块来响应(有两种投票方式
消息、预先投票和预先提交投票).请注意，提案消息只是一个建议
下一个块应该是；验证者可能会使用“VoteMessage”为不同的区块投票.如果在某些
轮，足够数量的进程投票给同一个块，然后提交这个块，然后
添加到区块链中. `ProposalMessage` 和 `VoteMessage` 由用户的私钥签名
验证器.协议的内部结构以及它如何确保安全性和活性属性是
在即将发布的文件中进行了解释.

出于效率原因，Tendermint 共识协议中的验证者不直接就
块因为块大小很大，即，他们不将块嵌入到 `Proposal` 中，并且
`投票消息`.相反，他们就“BlockID”达成了一致(参见中的“BlockID”定义)
[区块链](https://github.com/tendermint/spec/blob/master/spec/core/data_structures.md#blockid)部分)
唯一标识每个块.块本身是
使用点对点八卦协议传播给验证器进程.它首先有一个
提议者首先将一个块拆分为多个块部分，然后在它们之间进行八卦
使用`BlockPartMessage` 处理.

Tendermint 中的验证器通过点对点八卦协议进行通信.每个验证器都是连接的
仅适用于称为对等点的进程子集.通过八卦协议，验证器向其对等方发送
所有需要的信息(`ProposalMessage`、`VoteMessage` 和 `BlockPartMessage`)，这样他们就可以
就某个区块达成一致，同时也获得了被选中区块(区块部分)的内容.作为
作为八卦协议的一部分，进程还发送辅助消息，通知对等方
执行核心共识算法的步骤(`NewRoundStepMessage` 和 `NewValidBlockMessage`)，以及
还有消息通知对等进程已经看到了什么投票(`HasVoteMessage`，
`VoteSetMaj23Message` 和 `VoteSetBitsMessage`).这些消息然后用于八卦
协议来确定进程应该向其对等方发送什么消息.

我们现在描述在 Tendermint 共识协议期间交换的每条消息的内容.
