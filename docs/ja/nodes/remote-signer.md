# リモート署名者

Tendermintは、検証者にリモート署名者オプションを提供します.リモート署名者を使用すると、オペレーターはバリデーターキーを別のマシンに保存できるため、サーバーが侵害された場合の攻撃対象領域を最小限に抑えることができます.

リモート署名者プロトコルは、[クライアントおよびサーバーアーキテクチャ](https://en.wikipedia.org/wiki/Client%E2%80%93server_model)を実装します. Tendermintは、提案または投票のために公開鍵または署名を必要とする場合、リモート署名者にそれを要求します.

安全なバリデーターとリモート署名者システムを実行するには、VPC(仮想プライベートクラウド)またはプライベート接続を使用することをお勧めします.

RawまたはgRPCの2つの異なる構成を使用できます.

## 生

どちらのオプションもtcpまたはunixソケットを使用しますが、元のオプションはhttpなしのtcpまたはunixソケットを使用します.元のプロトコルでは、Tendermintをサーバーとして設定し、リモート署名者をクライアントとして設定していました.これは、リモート署名者をパブリックネットワークに公開しないようにするのに役立ちます.

>警告:Rawは将来のメジャーリリースで非推奨になります.gRPC構成用にキー管理サーバーを実装することをお勧めします.

## gRPC

[gRPC](https://grpc.io/)は、[HTTP/2](https://en.wikipedia.org/wiki/HTTP/2)と[Protocol Buffers](https: //developers.google.com/protocol-buffers)サービスを定義し、クラウドインフラストラクチャコミュニティで標準化されています. gRPCは、サービスを実装するための言語に依存しない方法を提供します.これは、開発者がさまざまな異なる言語でキー管理サーバーを作成するのに役立ちます.

GRPCは、別の広く標準化されたプロトコル[TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security)を使用して接続を保護します.接続を保護するためのTLSには、一方向と双方向の2つの形式があります. 1つの方法は、クライアントがサーバーを認識しているが、サーバーが誰でもサーバーに接続できるようにする場合です. 2つの方法は、クライアントがサーバーを認識し、サーバーがクライアントを認識して、不明な関係者からの接続を禁止する場合です.

gRPCを使用する場合、Tendermintがクライアントとして設定されます. Tendermintはリモート署名者を呼び出します.リモート署名者をパブリックネットワークに公開するために仮想プライベートクラウドを使用しないことをお勧めします.

リモート署名者接続を保護することを強くお勧めしますが、安全でない接続を使用して実行するオプションを提供しています.

### 証明書を生成する

gRPCとの安全な接続を実行するには、証明書とキーを生成する必要があります.相互TLSの証明書に自己署名する方法について説明します.

証明書を生成するには、[openssl](https://www.openssl.org/)と[certstarp](https://github.com/square/certstrap)の2つの方法があります.どちらのオプションも使用できますが、opensslよりも単純なプロセスを提供するため、 `certstrap`を紹介します.

-`Certstrap`をインストールします.

```sh
  go get github.com/square/certstrap@v1.2.0
```

-自己署名用の認証局を作成します.

```sh
 # generate self signing ceritificate authority
 certstrap init --common-name "<name_CA>" --expires "20 years"
```

-サーバーの証明書を要求します.
    -汎用的にはIPを「127.0.0.1」に設定していますが、ノードにはサーバーIPを使用してください.
-認証局を使用してサーバー証明書に署名します

```sh
 # generate server cerificate
 certstrap request-cert -cn server -ip 127.0.0.1
 # self-sign server cerificate with rootCA
 certstrap sign server --CA "<name_CA>" 127.0.0.1
  ```

-サーバーの証明書を要求します.
    -汎用的にはIPを「127.0.0.1」に設定していますが、ノードにはサーバーIPを使用してください.
-認証局を使用してサーバー証明書に署名します

```sh
# generate client cerificate
 certstrap request-cert -cn client -ip 127.0.0.1
# self-sign client cerificate with rootCA
 certstrap sign client --CA "<name_CA>" 127.0.0.1
```
