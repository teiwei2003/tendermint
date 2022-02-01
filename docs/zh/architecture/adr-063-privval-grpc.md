# ADR 063:Privval gRPC

## 变更日志

- 23/11/2020:初始版本 (@marbar3778)

## 语境

验证者使用远程签名者来帮助保护他们的密钥.该系统是 Tendermint 推荐的保护验证器的方法，但与 Tendermint 的私有验证器客户端集成的路径受到自定义协议的困扰.

Tendermint 使用其自己的自定义安全连接协议(`SecretConnection`)和原始 tcp/unix 套接字连接协议.直到最近，安全连接协议都受到中间人攻击，如果不使用 Golang，则可能需要更长的时间来集成.原始 tcp 连接协议不太自定义，但已经给用户带来了一些小问题.

将 Tendermint 的私有验证器客户端迁移到广泛采用的协议 gRPC，将减轻当前协议所经历的当前维护和集成负担.

## 决定

在与多个利益相关者讨论后，决定 [gRPC](https://grpc.io/) 替换当前的私有验证器协议. gRPC 是微服务和云基础设施领域广泛采用的协议. gRPC 使用 [protocol-buffers](https://developers.google.com/protocol-buffers) 来描述其服务，提供与语言无关的实现. Tendermint 使用 protobuf 进行磁盘和在线编码，已经使与 gRPC 的集成更加简单.

## 替代方法

- JSON-RPC:我们没有考虑 JSON-RPC，因为 Tendermint 广泛使用 protobuf，使 gRPC 成为自然的选择.

## 详细设计

随着最近将 [Protobuf](https://developers.google.com/protocol-buffers) 集成到 Tendermint，从当前私有验证器协议迁移到 gRPC 所需的更改并不大.

gRPC 的 [服务定义](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition) 将定义为:

```proto
  service PrivValidatorAPI {
    rpc GetPubKey(tendermint.proto.privval.PubKeyRequest) returns (tendermint.proto.privval.PubKeyResponse);
    rpc SignVote(tendermint.proto.privval.SignVoteRequest) returns (tendermint.proto.privval.SignedVoteResponse);
    rpc SignProposal(tendermint.proto.privval.SignProposalRequest) returns (tendermint.proto.privval.SignedProposalResponse);

    message PubKeyRequest {
    string chain_id = 1;
  }

  // PubKeyResponse is a response message containing the public key.
  message PubKeyResponse {
    tendermint.crypto.PublicKey pub_key = 1 [(gogoproto.nullable) = false];
  }

  // SignVoteRequest is a request to sign a vote
  message SignVoteRequest {
    tendermint.types.Vote vote     = 1;
    string                chain_id = 2;
  }

  // SignedVoteResponse is a response containing a signed vote or an error
  message SignedVoteResponse {
    tendermint.types.Vote vote  = 1 [(gogoproto.nullable) = false];
  }

  // SignProposalRequest is a request to sign a proposal
  message SignProposalRequest {
    tendermint.types.Proposal proposal = 1;
    string                    chain_id = 2;
  }

  // SignedProposalResponse is response containing a signed proposal or an error
  message SignedProposalResponse {
    tendermint.types.Proposal proposal = 1 [(gogoproto.nullable) = false];
  }
}
```

> 注意:Remote Singer 错误已被移除，以支持 [grpc 状态错误代码](https://grpc.io/docs/guides/error/).

在远程签名者的早期版本中，Tendermint 作为服务器，远程签名者作为客户端.在这个过程中，客户端建立了一个长期连接，为服务器向客户端发出请求提供了一种方式.在新版本中，它已被简化. Tendermint 是客户端，远程签名者是服务器.这遵循客户端和服务器架构并简化了之前的协议.

#### 活着

如果您使用过私有验证器系统，您将看到我们正在删除“PingRequest”和“PingResponse”消息.这些消息用于创建保持连接活动的功能.使用 gRPC 有一个 [保持活动功能](https://github.com/grpc/grpc/blob/master/doc/keepalive.md)，它将与集成一起添加以提供相同的功能.

#### 指标

远程签名者对于操作安全且持续运行的验证器至关重要.过去，除了节点未签名之外，没有任何指标可以告诉操作员是否有问题.将指标集成到客户端和提供的服务器将使用 [prometheus](https://github.com/grpc-ecosystem/go-grpc-prometheus) 完成.这将集成到节点操作员的节点普罗米修斯导出中.

#### 安全

[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) 广泛采用 gRPC.有多种形式的 TLS(单向和双向).一种方式是客户端识别服务器是谁，而两种方式是双方识别另一方.对于 Tendermint 的用例，让双方相互识别提供了额外的安全层.这要求用户为 TLS 连接生成客户端和服务器证书.

将为不希望保护连接的用户提供不安全选项.

####升级路径

对于验证器操作员来说，这是一个重大的突破性变化.最佳升级路径是在次要版本中发布 gRPC，允许密钥管理系统迁移到新协议.在下一个主要版本中，当前系统(原始 tcp/unix)被删除.这允许用户迁移到新系统，而不必在网络升级的同时协调升级密钥管理系统.

[tmkms](https://github.com/iqlusioninc/tmkms)的升级将与Iqlusion协调​​.他们将能够进行必要的升级，以允许用户从当前协议迁移到 gRPC.

## 状态


实施的

### 积极的

- 使用已采用的安全通信标准. (TLS)
- 使用采用的通信协议. (gRPC)
- 请求被多路复用到 tcp 连接上. (http/2)
- 语言不可知的服务定义.

### 消极的

- 用户需要生成证书才能使用 TLS. (添加步骤)
- 用户需要找到受支持的 gRPC 支持的密钥管理系统

### 中性的
