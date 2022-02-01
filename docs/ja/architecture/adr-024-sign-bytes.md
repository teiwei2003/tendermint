# ADR 024:privvalのSignBytesとバリデータータイプ

## 環境

現在、テンダーミントと(おそらくリモートの)署名者/検証者の間で交換されるメッセージは、
つまり、投票、提案、ハートビートはJSON文字列としてエンコードされます
(たとえば、 `Vote.SignBytes(...)`を介して)次に
サイン. JSONエンコーディングはハードウェアウォレットには最適ではありません
イーサリアムのスマートコントラクトで使用されます.どちらも[issue#1622]で指定されています.

さらに、現在、署名されたリクエストと返信の間に違いはありません.また、それは不可能です
問題が発生した場合に備えて、リモート署名者にエラーコードまたはメッセージを含めるように依頼します.
現在、テンダーミントとリモート署名者の間で交換されるメッセージは、
[privval /socket.go]そして対応するタイプを[types]にカプセル化します.


[privval/socket.go]:https://github.com/tendermint/tendermint/blob/d419fffe18531317c28c29a292ad7d253f6cafdf/privval/socket.go#L496-L502
[問題#1622]:https://github.com/tendermint/tendermint/issues/1622
[タイプ]:https://github.com/tendermint/tendermint/tree/master/types


## 決定

-投票、提案、ハートビートを再編成して、コードを簡単に解析できるようにしました
バイナリエンコーディング形式(この例では[amino])を使用したハードウェアデバイスとスマートコントラクト
-テンダーミントとリモート署名者の間で交換されるメッセージを要求に分割し、
返信(詳細は以下を参照)
-応答にエラータイプを含めます

### 概要
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

TODO:または、タイプ「X」に署名が直接含まれている場合もあります. 多くの場所が投票を楽しみにしています
署名します.必ずしも「返信」を処理する必要はありません.
ここで何が最も効果的かをまだ探っています.
これは次のようになります(例としてX =投票を取り上げます):

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

**注:**公開鍵全体または公開鍵全体のフィンガープリントを含む、関連する議論があります
各署名要求を入力し、署名者に対応する秘密鍵を伝えます
メッセージに署名するために使用されます. これは、KMSのコンテキストでは特に重要です
ただし、現在、このADRでは考慮されていません.


[アミノ]:https://github.com/tendermint/go-amino/

### 投票

[issue#1622]で述べたように、 `Vote`は次のフィールドを含むように変更されます
(読みやすくするために、protobufのような構文表記を使用してください):

```proto
// vanilla protobuf/amino encoded
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

`ChainID`は投票メッセージに直接移動されます. 以前は注射されていました
[Signable]インターフェースメソッド `SignBytes(chainID string)[] byte`を使用します. 加えて
署名は直接含まれず、対応する「SignedVoteReply」メッセージにのみ含まれます.

[署名可能]:https://github.com/tendermint/tendermint/blob/d419fffe18531317c28c29a292ad7d253f6cafdf/types/signable.go#L9-L11

### 提案

```proto
// vanilla protobuf/amino encoded
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

### ハートビート

** TODO **:ハートビートにも固定オフセットが必要かどうかを明確にし、それに応じてフィールドを更新します.

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

## 公開鍵

TBA-これはこれ必要必要なかます:か、KMSをするするか
キーはいかつます？ 回信に用するキーを付けます

## シンボル
`SignBytes`は` ChainID`パラメータを必要なパラメータ:

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

## ステータス

部分的に受け入れられた

## 結果



### ポジティブ

最も関連性のあるプラスの効果は、署名バイトを簡単に解析できることです.
ハードウェアモジュールとスマートコントラクト. そのほか:

-要求と応答の明確な分離
-エラーをより適切に処理するためのエラーメッセージを追加しました


### ネガティブ

-比較的大きな変更/リファクタリングには、非常に多くのコードが含まれます
-多くの場所は署名付きの「投票」を想定しています->必要な
-一部のインターフェースを変更する必要があります

### ニュートラル

スイス人でさえ中立ではありません
