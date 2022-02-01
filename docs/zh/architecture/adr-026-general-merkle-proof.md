# ADR 026:一般默克尔证明

## 语境

我们在“abci.ResponseQuery”中使用原始的“[]byte”作为默克尔证明. 这使得处理多层默克尔证明和一般情况变得困难. 在这里，定义了新的接口“ProofOperator”. 用户可以定义自己的 Merkle 证明格式并轻松分层.

目标:
- 无需解码/重新编码的层 Merkle 证明
- 提供链证明的一般方法
- 使证明格式可扩展，允许第三方证明类型

## 决定

### ProofOperator

`type ProofOperator` 是 Merkle 证明的接口. 定义是:

```go
type ProofOperator interface {
    Run([][]byte) ([][]byte, error)
    GetKey() []byte
    ProofOp() ProofOp
}
```

由于证明可以处理各种数据类型，“Run()”将“[][]byte”作为参数，而不是“[]byte”.例如，范围证明的“Run()”可以将多个键值作为参数.然后它将返回树的根以进行进一步的处理，并使用输入值进行计算.

`ProofOperator` 不必是 Merkle 证明——它可以是一个函数，可以转换中间过程的参数，例如将长度添加到`[]byte`.

### ProofOp

`type ProofOp` 是一个 protobuf 消息，它是 `Type string`、`Key []byte` 和 `Data []byte` 的三元组. `ProofOperator` 和 `ProofOp` 是可以相互转换的，使用 `ProofOperator.ProofOp()` 和 `OpDecoder()`，其中 `OpDecoder` 是一个函数，每个证明类型都可以为其自己的编码方案注册.例如，我们可以在序列化证明之前添加一个用于编码方案的字节，支持JSON解码.

## 状态

实施的

## 结果

### 积极的

- 分层变得更容易(每一步都没有编码/解码)
- 第三方证明格式可用

### 消极的

- abci.ResponseQuery 的更大尺寸
- 不直观的证明链接(不清楚 `Run()` 在做什么)
- 用于注册 `OpDecoder`s 的附加代码
