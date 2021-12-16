# ADR 024:privval 中的 SignBytes 和验证器类型

## 语境

目前，tendermint 和(可能是远程的)签名者/验证者之间交换的消息，
即投票、提案和心跳，被编码为 JSON 字符串
(例如，通过`Vote.SignBytes(...)`)然后
签 。 JSON 编码对于硬件钱包来说都是次优的
并用于以太坊智能合约。两者都在 [issue#1622] 中有详细规定。

此外，目前签名请求和回复之间没有区别。还有，没有可能
让远程签名者包含错误代码或消息，以防出现问题。
目前，tendermint 和远程签名者之间交换的消息位于
[privval/socket.go] 并将相应的类型封装在[types] 中。


[privval/socket.go]:https://github.com/tendermint/tendermint/blob/d419fffe18531317c28c29a292ad7d253f6cafdf/privval/socket.go#L496-L502
[问题#1622]:https://github.com/tendermint/tendermint/issues/1622
[类型]:https://github.com/tendermint/tendermint/tree/master/types


## 决定

- 重组投票、提案和心跳，以便它们的编码很容易被解析
使用二进制编码格式的硬件设备和智能合约(本例中为 [amino])
- 将tendermint和远程签名者之间交换的消息拆分为请求和
回复(详见下文)
- 在响应中包含错误类型

### 概述
```
+--------------+                      +----------------+
|              |     SignXRequest     |                |
|Remote signer |<---------------------+  tendermint    |
| (e.g. KMS)   |                      |                |
|              +--------------------->|                |
+--------------+    SignedXReply      +----------------+


SignXRequest {
    x: X
}

SignedXReply {
    x: X
  sig: Signature // []byte
  err: Error{
    code: int
    desc: string
  }
}
```

TODO:或者，类型“X”可能直接包含签名。 很多地方都期待投票
签名，不一定处理“回复”。
仍在探索什么在这里最有效。
这看起来像(以 X = Vote 为例):

```
Vote {
    // all fields besides signature
}

SignedVote {
 Vote Vote
 Signature []byte
}

SignVoteRequest {
   Vote Vote
}

SignedVoteReply {
    Vote SignedVote
    Err  Error
}
```

**注意:** 有一个相关的讨论，包括包含整个公钥的指纹或整个公钥
进入每个签名请求，告诉签名者对应的私钥
用于对消息进行签名。 这在 KMS 的背景下尤其重要
但目前不在本 ADR 中考虑。


[氨基]:https://github.com/tendermint/go-amino/

###投票

如 [issue#1622] 中所述，`Vote` 将更改为包含以下字段
(为了便于阅读，使用类似 protobuf 的语法表示法):

```proto
// vanilla protobuf / amino encoded
message Vote {
    Version       fixed32
    Height        sfixed64
    Round         sfixed32
    VoteType      fixed32
    Timestamp     Timestamp         // << using protobuf definition
    BlockID       BlockID           // << as already defined
    ChainID       string            // at the end because length could vary a lot
}

// this is an amino registered type; like currently privval.SignVoteMsg:
// registered with "tendermint/socketpv/SignVoteRequest"
message SignVoteRequest {
   Vote vote
}

//  amino registered type
// registered with "tendermint/socketpv/SignedVoteReply"
message SignedVoteReply {
   Vote      Vote
   Signature Signature
   Err       Error
}

// we will use this type everywhere below
message Error {
  Type        uint  // error code
  Description string  // optional description
}

```

`ChainID` 被直接移动到投票消息中。 以前是注射的
使用 [Signable] 接口方法 `SignBytes(chainID string) []byte`。 此外，该
签名不会被直接包含，只会包含在相应的“SignedVoteReply”消息中。

[可签名]:https://github.com/tendermint/tendermint/blob/d419fffe18531317c28c29a292ad7d253f6cafdf/types/signable.go#L9-L11

### 提议

```proto
// vanilla protobuf / amino encoded
message Proposal {
    Height            sfixed64
    Round             sfixed32
    Timestamp         Timestamp         // << using protobuf definition
    BlockPartsHeader  PartSetHeader     // as already defined
    POLRound          sfixed32
    POLBlockID        BlockID           // << as already defined
}

// amino registered with "tendermint/socketpv/SignProposalRequest"
message SignProposalRequest {
   Proposal proposal
}

// amino registered with "tendermint/socketpv/SignProposalReply"
message SignProposalReply {
   Prop   Proposal
   Sig    Signature
   Err    Error     // as defined above
}
```

### 心跳

**TODO**:澄清心跳是否也需要固定的偏移量并相应地更新字段:

```proto
message Heartbeat {
	ValidatorAddress Address
	ValidatorIndex   int
	Height           int64
	Round            int
	Sequence         int
}
// amino registered with "tendermint/socketpv/SignHeartbeatRequest"
message SignHeartbeatRequest {
   Hb Heartbeat
}

// amino registered with "tendermint/socketpv/SignHeartbeatReply"
message SignHeartbeatReply {
   Hb     Heartbeat
   Sig    Signature
   Err    Error     // as defined above
}

```

## 公钥

TBA - 这需要进一步思考:例如 在持有 KMS 的情况下，要做什么
几个键？ 它怎么知道用哪个键来回复？

## 符号字节
`SignBytes` 不需要 `ChainID` 参数:

```golang
type Signable interface {
	SignBytes() []byte
}

```
And the implementation for vote, heartbeat, proposal will look like:
```golang
// type T is one of vote, sign, proposal
func (tp *T) SignBytes() []byte {
	bz, err := cdc.MarshalBinary(tp)
	if err != nil {
		panic(err)
	}
	return bz
}
```

## 状态

部分接受

## 结果



### 积极的

最相关的积极影响是签名字节可以很容易地被解析
硬件模块和智能合约。 除此之外:

- 请求和响应之间更清晰的分离
- 添加的错误消息可以更好地处理错误


### 消极的

- 相对巨大的变化/重构涉及相当多的代码
- 很多地方都假设包含签名的“投票” -> 他们需要
- 需要修改一些接口

### 中性的

连瑞士人都不是中立的
