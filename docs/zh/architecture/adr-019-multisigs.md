# ADR 019:多重签名的编码标准

## 变更日志

06-08-2018:小更新

27-07-2018:更新草案以使用氨基编码

11-07-2018:初稿

2021 年 5 月 26 日:Multisig 被移入 Cosmos-sdk

## 语境

多重签名，或技术上 _Accountable Subgroup Multisignatures_ (ASM)，
是签名方案，使一组签名者的任何子组能够签署任何消息，
并向验证者透露签名者的确切身份。
这允许何时验证签名的复杂条件。

假设签名者集的大小为 _n_。
如果我们验证签名，如果任何大小为 _k_ 的子组签署消息，
这成为比特币中通常称为 n 多重签名的 _k 的内容。

本 ADR 规定了一般负责子组多重签名的编码标准，
n 个负责子组多重签名中的 k 个，及其加权变体。

将来，我们还可以允许对可问责子组使用更复杂的条件。

## 建议的解决方案

### 新结构

然后每个 ASM 都有自己的结构，实现了 crypto.Pubkey 接口。

此 ADR 假定已接受 [用 [] 字节替换 crypto.Signature](https://github.com/tendermint/tendermint/issues/1957)。

#### K of N 个阈值签名

公钥是以下结构:

```golang
type ThresholdMultiSignaturePubKey struct { // K of N threshold multisig
	K       uint               `json:"threshold"`
	Pubkeys []crypto.Pubkey    `json:"pubkeys"`
}
```

我们将从公钥的长度推导出 N。 (为了编码的空间效率)

`Verify` 需要一个 `[]byte` 编码版本的多重签名。
(多重签名在下一节中描述)
如果位图的索引少于 k 个，则多重签名将被拒绝，
或者如果 k 个索引中的任何一个的签名不是来自
消息中的第 k 个公钥。
(如果包含超过k个签名，则所有签名都必须有效)

`Bytes` 将是公钥的氨基编码版本。

地址将是`Hash(amino_encoded_pubkey)`

这不为每个签名者使用 `log_8(n)` 字节的原因是因为它针对需要非常少量签名者的情况进行了大量优化。
例如 对于大小为“24”的“n”，对于“k < 3”，这只会更节省空间。
这似乎不太可能，并且不应该针对这种情况进行优化。

#### 加权阈值签名

公钥是以下结构:

```golang
type WeightedThresholdMultiSignaturePubKey struct {
	Weights []uint             `json:"weights"`
	Threshold uint             `json:"threshold"`
	Pubkeys []crypto.Pubkey    `json:"pubkeys"`
}
```

Weights 和 Pubkeys 的长度必须相同。
其他一切都与 N 多重签名的 K 相同，
如果权重之和小于阈值，则多重签名失败。

#### 多重签名

签名的中间阶段(因为它会产生更多签名)将是以下结构:

```golang
type Multisignature struct {
	BitArray    CryptoBitArray // Documented later
	Sigs        [][]byte
```

重要的是要记住，每个私钥都会在提供的消息本身上输出一个签名。
所以没有签名算法会输出多重签名。
UI 将接受签名，转换为多重签名，然后继续添加
将新签名加入其中，完成后编组为 `[]byte`。
这将需要以下辅助方法:

```golang
func SigToMultisig(sig []byte, n int)
func GetIndex(pk crypto.Pubkey, []crypto.Pubkey)
func AddSignature(sig Signature, index int, multiSig *Multisignature)
```

多重签名将使用amino.MarshalBinaryBare 转换为`[]byte`。 \*

####位阵列

我们将使用位数组的新实现。 它将被编码/解码的结构是

```golang
type CryptoBitArray struct {
	ExtraBitsStored  byte      `json:"extra_bits"` // The number of extra bits in elems.
	Elems            []byte    `json:"elems"`
}
```

不使用当前在`libs/common/bit_array.go`中实现的BitArray的原因
是由于空间/时间权衡，它的空间效率较低。
[本期](https://github.com/tendermint/tendermint/issues/2077) 中概述了这方面的证据。

在多重签名中，我们不会执行算术运算，
所以当前的实现没有性能提升，
而只是空间效率的损失。
用 `[]byte` 实现这个新的位数组_应该_很简单，因为没有
需要位数组之间的算术运算，并节省几个字节。
(在同一问题中解释过)

当这个位数组编码时，元素的数量是由氨基编码的。
然而，我们可能正在为我们实际上只需要 1-7 位的内容编码一个完整的字节。
我们将该差异存储在 ExtraBitsStored 中。
这允许我们拥有无限数量的签名者，并且比目前在 `libs/common` 中使用的更节省空间。
再次，此节省空间的功能的实现是直截了当的。

### 编码结构

我们将使用直接的氨基编码。选择此选项是为了便于与其他语言兼容。

### 未来的讨论点

如果需要，我们可以对所有 ed25519 密钥使用 ed25519 批量验证。
这是一个未来的讨论点，但会向后兼容，因为不需要编组这些信息。
(如果没有 ristretto，甚至可能存在辅因子问题)
Schnorr sigs/BLS sigs 中公钥/sigs 的聚合不向后兼容，需要是新的 ASM 类型。

## 状态

已实现(移至 cosmos-sdk)

## 结果

### 积极的

- 支持多重签名，在我们的下游验证码中不需要任何特殊情况。
- 易于序列化/反序列化
- 无限数量的签名者

### 消极的

- 更大的代码库，但是这应该位于tendermint/crypto 的子文件夹中，因为它没有提供新的接口。 (参考 #https://github.com/tendermint/go-crypto/issues/136)
- 由于使用氨基编码，空间效率低下
- 建议的实现要求每个 ASM 都有一个新结构。

### 中性的
