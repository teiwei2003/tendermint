# ADR 064:批量验证

## 变更日志

- 2021 年 1 月 28 日:创建 (@marbar3778)

## 语境

Tendermint 使用公钥私钥加密进行验证器签名。当一个区块被提议并投票给验证者签署一条表示接受一个区块的消息时，通过零投票表示拒绝。如果节点正在同步，这些签名还用于验证先前的块是否正确。目前，Tendermint 要求单独验证每个签名，这会导致出块时间变慢。

批量验证是将许多消息、密钥和签名添加到一起并同时验证它们的过程。公钥可以相同，在这种情况下，这意味着单个用户正在签署多条消息。在我们的例子中，每个公钥都是唯一的，每个验证者都有自己的并提供唯一的消息。该算法可能因曲线而异，但性能优势与单一验证消息、公钥和签名是共享的。

## 替代方法

- 签名聚合
  - 签名聚合是批量验证的替代方法。签名聚合导致快速验证和更小的块大小。在撰写此 ADR 时，正在开展在 Tendermint 中启用签名聚合的工作。我们选择在此时不引入它的原因是因为每个验证者都签署了一个独特的消息。
  对唯一消息进行签名可防止在验证之前进行聚合。例如，如果我们要使用 BLS 实施签名聚合，则验证速度可能会降低 10 到 100 倍。

## 决定

采用批量验证。

## 详细设计

将引入一个新界面。这个接口将有三个方法`NewBatchVerifier`、`Add`和`VerifyBatch`。

```go
type BatchVerifier interface {
  Add(key crypto.Pubkey, signature, message []byte) error // Add appends an entry into the BatchVerifier.
  Verify() bool // Verify verifies all the entries in the BatchVerifier. If the verification fails it is unknown which entry failed and each entry will need to be verified individually.
}
```

- `NewBatchVerifier` 创建一个新的验证器。 此验证器将填充要验证的条目。
- `Add` 向验证器添加一个条目。 Add 接受一个公钥和两个字节片(签名和消息)。
- `Verify` 验证所有内容。 在验证结束时，如果底层 API 没有将验证器重置为其初始状态(空)，则应在此处完成。 这可以防止意外地将验证器与先前验证的条目重用。

上面提到了一个条目。 根据底层曲线的需要，可以通过多种方式构建条目。 一个简单的方法是:

```go
type entry struct {
  pubKey crypto.Pubkey
  signature []byte
  message []byte
}
```

采取这种方法的主要原因是为了防止简单的错误。 一些 API 允许用户创建三个切片并将它们传递给“VerifyBatch”函数，但这依赖于用户安全地生成所有切片(参见下面的示例)。 我们希望尽量减少出错的可能性。

```go
func Verify(keys []crypto.Pubkey, signatures, messages[][]byte) bool
```

除了更快的验证时间之外，此更改不会以任何方式影响任何用户。

这个新的 api 将用于共识和块同步的验证。 在当前的验证函数中，将检查密钥类型是否支持 BatchVerification API。 如果是，则执行批量验证，否则将使用单签名验证。

#### 共识

  共识中的过程将等待收到 2/3+ 的选票，一旦收到，将调用“Verify()”来批量验证所有消息。 2/3+ 之后收到的消息将被单独验证。

#### 块同步和轻客户端

  块同步和轻客户端验证的过程将以批处理方式仅验证 2/3+。由于这些进程不参与共识，因此无需等待更多消息。

如果批量验证因任何原因失败，将不知道是哪个条目导致了失败。验证将需要恢复到单一签名验证。

刚开始，只有 ed25519 将支持批量验证。

## 状态

实施的

### 积极的

- 更快的验证时间，如果曲线支持它

### 消极的

- 无法查看哪个密钥验证失败
  - 失败意味着恢复到单一签名验证。

### 中性的

## 参考

[Ed25519 库](https://github.com/hdevalence/ed25519consensus)
[Ed25519 规格](https://ed25519.cr.yp.to/)
【签名聚合投票】(https://github.com/tendermint/tendermint/issues/1319)
[基于 Proposer 的时间戳](https://github.com/tendermint/tendermint/issues/2840)
