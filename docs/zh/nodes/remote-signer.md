# 远程签名者

Tendermint 为验证者提供了远程签名者选项。远程签名者使操作员能够将验证器密钥存储在不同的机器上，从而在服务器受到威胁时最大限度地减少攻击面。

远程签名者协议实现了[客户端和服务器架构](https://en.wikipedia.org/wiki/Client%E2%80%93server_model)。当 Tendermint 需要提案或投票的公钥或签名时，它会从远程签名者那里请求。

要运行安全验证器和远程签名器系统，建议使用 VPC(虚拟私有云)或私有连接。

可以使用两种不同的配置:Raw 或 gRPC。

## 生的

虽然这两个选项都使用 tcp 或 unix 套接字，但原始选项使用没有 http 的 tcp 或 unix 套接字。原始协议将 Tendermint 设置为服务器，将远程签名者设置为客户端。这有助于不将远程签名者暴露给公共网络。

> 警告:Raw 将在未来的主要版本中弃用，我们建议针对 gRPC 配置实施您的密钥管理服务器。

## gRPC

[gRPC](https://grpc.io/) 是一个使用 [HTTP/2](https://en.wikipedia.org/wiki/HTTP/2) 构建的 RPC 框架，使用 [Protocol Buffers](https: //developers.google.com/protocol-buffers) 来定义服务并已在云基础设施社区内标准化。 gRPC 提供了一种与语言无关的方式来实现服务。这有助于开发人员以各种不同的语言编写密钥管理服务器。

GRPC 利用另一个广泛标准化的协议 [TLS](https://en.wikipedia.org/wiki/Transport_Layer_Security) 来保护连接。有两种形式的 TLS 来保护连接，单向和双向。一种方法是当客户端识别服务器但服务器允许任何人连接到它时。两种方式是当客户端识别服务器和服务器识别客户端时，禁止来自未知方的连接。

使用 gRPC 时，Tendermint 设置为客户端。 Tendermint 将调用远程签名者。我们建议不要使用虚拟私有云将远程签名者暴露给公共网络。

强烈建议保护您的远程签名者连接，但我们提供了使用不安全连接运行它的选项。

### 生成证书

要与 gRPC 运行安全连接，我们需要生成证书和密钥。 我们将演练如何为双向 TLS 自签名证书。

生成证书有两种方式，[openssl](https://www.openssl.org/)和[certstarp](https://github.com/square/certstrap)。 这两个选项都可以使用，但我们将介绍 `certstrap`，因为它提供了一个比 openssl 更简单的过程。

- 安装`Certstrap`:

```sh
  go get github.com/square/certstrap@v1.2.0
```

- 创建用于自签名的证书颁发机构。

```sh
 # generate self signing ceritificate authority
 certstrap init --common-name "<name_CA>" --expires "20 years"
```

- 为服务器申请证书。
   - 出于通用目的，我们将 IP 设置为“127.0.0.1”，但对于您的节点，请使用服务器 IP。
- 使用您的证书颁发机构签署服务器证书

```sh
 # generate server cerificate
 certstrap request-cert -cn server -ip 127.0.0.1
 # self-sign server cerificate with rootCA
 certstrap sign server --CA "<name_CA>" 127.0.0.1
  ```

- 为服务器申请证书。
   - 出于通用目的，我们将 IP 设置为“127.0.0.1”，但对于您的节点，请使用服务器 IP。
- 使用您的证书颁发机构签署服务器证书

```sh
# generate client cerificate
 certstrap request-cert -cn client -ip 127.0.0.1
# self-sign client cerificate with rootCA
 certstrap sign client --CA "<name_CA>" 127.0.0.1
```
