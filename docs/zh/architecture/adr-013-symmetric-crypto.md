# ADR 013:对称密码学的需要

## 语境

我们需要对称密码来处理我们如何加密 sdk 中的密钥，
并可能在tendermint中加密`priv_validator.json`.

目前我们使用AEAD来支持对称加密，
这很好，因为除了隐私和真实性之外，我们还需要数据完整性.
我们目前没有想要在没有数据完整性的情况下加密的场景，
因此可以优化我们的代码以仅使用 AEAD.
目前没有办法轻松切换 AEAD，此 ADR 概述了一种方法
轻松地换掉这些.

### 我们如何使用 AEAD 加密

除了密钥之外，AEAD 通常还需要一个随机数.
出于我们需要对称加密的目的，
我们需要加密是无状态的.
因此，我们使用随机数.
(因此 AEAD 必须支持随机数)

我们目前构造一个随机数，并用它加密数据.
返回值为 `nonce ||加密数据`.
这样做的限制是不提供识别的方法
加密中使用了哪种算法.
因此，使用多种算法进行解密是次优的.
(你必须全部尝试)

## 决定

我们应该在一个新的 `crypto/encoding/symmetric` 包中创建以下两个方法:

```golang
func Encrypt(aead cipher.AEAD, plaintext []byte) (ciphertext []byte, err error)
func Decrypt(key []byte, ciphertext []byte) (plaintext []byte, err error)
func Register(aead cipher.AEAD, algo_name string, NewAead func(key []byte) (cipher.Aead, error)) error
```

这允许您指定加密算法，但不必指定
它在解密.
这是为了便于在下游应用程序中使用，除了人员
直接看文件.
一个缺点是，对于加密功能，您必须已经初始化了一个 AEAD，
但我真的不认为这是一个问题.

如果加密没有错误，Encrypt 会返回 `algo_name || 随机数 || aead_密文`.
`algo_name` 应该是长度前缀，使用标准的 varuint 编码.
这将是二进制数据，但考虑到随机数和密文也是二进制，这不是问题.

此解决方案需要从 aead 类型到名称的映射.
我们可以通过反射来实现这一点.

```golang
func getType(myvar interface{}) string {
    if t := reflect.TypeOf(myvar); t.Kind() == reflect.Ptr {
        return "*" + t.Elem().Name()
    } else {
        return t.Name()
    }
}
```

然后我们维护一个从`getType(aead)`返回的名称到`algo_name`的映射.

在解密中，我们读取`algo_name`，然后用密钥实例化一个新的AEAD.
然后我们在提供的随机数/密文上调用 AEAD 的解密方法.

`Register` 允许下游用户将他们自己想要的 AEAD 添加到对称包中.
如果已经注册了 AEAD 名称，则会出错.
这可以防止恶意导入在运行时修改/取消 AEAD.

## 实施策略

所提议内容的 golang 实现相当简单.
令人担忧的是，如果我们只是切换到这个，我们将破坏现有的私钥.
如果这是相关的，我们可以制作一个不需要解码私钥的简单脚本，
用于从旧格式转换为新格式.

## 状态

建议的.

## 结果

### 积极的

- 允许我们以一种使解密更容易的方式支持新的 AEAD
- 允许下游用户添加自己的AEAD

### 消极的

- 我们将不得不破解存储在磁盘上的所有私钥.
   它们可以使用种子词恢复，升级脚本也很简单.

### 中性的

- 调用者必须使用私钥实例化 AEAD.
   然而，它迫使他们知道他们正在使用什么签名算法，这是积极的.
