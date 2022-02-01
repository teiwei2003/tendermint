# ADR 063:Privval gRPC

## 変更ログ

-2020年11月23日:初期バージョン(@ marbar3778)

## 環境

検証者は、リモート署名者を使用してキーを保護します.このシステムは、バリデーターを保護するためにTendermintが推奨する方法ですが、Tendermintのプライベートバリデータークライアントとの統合パスは、カスタムプロトコルによって問題が発生します.

Tendermintは、独自のカスタムセキュア接続プロトコル( `SecretConnection`)と元のtcp/unixソケット接続プロトコルを使用します.最近まで、安全な接続プロトコルは中間者攻撃の対象となってきました.Golangを使用しない場合、統合に時間がかかる可能性があります.元のTCP接続プロトコルはあまりカスタマイズできませんが、ユーザーにいくつかの小さな問題を引き起こしました.

Tendermintのプライベートバリデータークライアントを広く採用されているプロトコルgRPCに移行すると、現在のプロトコルで発生する現在のメンテナンスと統合の負担が軽減されます.

## 決定

複数の利害関係者と話し合った後、現在のプライベートバリデータープロトコルを[gRPC](https://grpc.io/)に置き換えることが決定されました. gRPCは、マイクロサービスとクラウドインフラストラクチャの分野で広く採用されているプロトコルです. gRPCは、[protocol-buffers](https://developers.google.com/protocol-buffers)を使用して、サービスを記述し、言語に依存しない実装を提供します. Tendermintは、ディスクおよびオンラインエンコーディングにprotobufを使用しているため、gRPCとの統合が容易になっています.

## 代替方法

-JSON-RPC:Tendermintはprotobufを広範囲に使用し、gRPCを自然に選択するため、JSON-RPCは考慮しませんでした.

## 詳細設計

[Protobuf](https://developers.google.com/protocol-buffers)がTendermintに最近統合されたため、現在のプライベートバリデータープロトコルからgRPCに移行するために必要な変更は重要ではありません.

gRPCの[サービス定義](https://grpc.io/docs/what-is-grpc/core-concepts/#service-definition)は次のように定義されます.

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

>注:[grpcステータスエラーコード](https://grpc.io/docs/guides/error/)をサポートするために、リモートシンガーエラーが削除されました.

以前のバージョンのリモート署名者では、Tendermintがサーバーであり、リモート署名者がクライアントでした.このプロセスでは、クライアントは長期接続を確立します.これにより、サーバーがクライアントに要求を送信する方法が提供されます.新しいバージョンでは、簡略化されています. Tendermintはクライアントであり、リモート署名者はサーバーです.これはクライアントとサーバーのアーキテクチャに従い、以前のプロトコルを簡素化します.

#### 生きている

プライベートバリデーターシステムを使用した場合は、「PingRequest」メッセージと「PingResponse」メッセージが削除されていることがわかります.これらのメッセージは、接続を維持するための関数を作成するために使用されます. gRPCを使用した[キープアライブ機能](https://github.com/grpc/grpc/blob/master/doc/keepalive.md)があります.これは、同じ機能を提供するために統合とともに追加されます.

#### インジケーター

リモート署名者は、安全で継続的に実行されるバリデーターを操作するために不可欠です.以前は、ノードが署名されていないことを除いて、問題があったかどうかをオペレーターに伝えることができるインジケーターはありませんでした.クライアントと提供されたサーバーへのインジケーターの統合は、[prometheus](https://github.com/grpc-ecosystem/go-grpc-prometheus)を使用して行われます.これは、ノードオペレーターのノードPrometheusエクスポートに統合されます.

#### 安全性

[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)gRPCは広く採用されています. TLSには多くの形式があります(一方向および双方向). 1つの方法は、クライアントがサーバーを識別する方法であり、2つの方法は、両方の当事者が他方を識別する方法です. Tendermintのユースケースでは、両方の当事者がお互いを識別できるようにすることで、セキュリティの追加レイヤーが提供されます.これには、ユーザーがTLS接続用のクライアント証明書とサーバー証明書を生成する必要があります.

接続を保護したくないユーザーには、安全でないオプションが提供されます.

#### アップグレードパス

バリデーターオペレーターにとって、これは大きな画期的な変更です.最適なアップグレードパスは、マイナーバージョンでgRPCをリリースし、キー管理システムを新しいプロトコルに移行できるようにすることです.次のメジャーバージョンでは、現在のシステム(元のtcp/unix)が削除されます.これにより、ユーザーは、ネットワークのアップグレードと同時にキー管理システムのアップグレードを調整しなくても、新しいシステムに移行できます.

[tmkms](https://github.com/iqlusioninc/tmkms)アップグレードはIqlusionと調整されます.ユーザーが現在のプロトコルからgRPCに移行できるように、必要なアップグレードを行うことができます.

## ステータス


実装

### ポジティブ

-採用された安全通信規格を使用します. (TLS)
-採用された通信プロトコルを使用します. (gRPC)
-要求はtcp接続に多重化されます. (http/2)
-言語に依存しないサービス定義.

### ネガティブ

-ユーザーはTLSを使用するために証明書を生成する必要があります. (ステップを追加)
-ユーザーは、gRPCでサポートされているサポートされているキー管理システムを見つける必要があります

### ニュートラル
