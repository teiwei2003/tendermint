# ADR 046:Lite 客户端实现

## 变更日志
* 13-02-2020:初稿
* 26-02-2020:交叉检查第一个标题
* 28-02-2020:二分算法细节
* 31-03-2020:验证签名已更改

## 语境

`Client` 结构代表一个轻客户端，连接到单个区块链。

用户可以选择使用 `VerifyHeader` 或
`VerifyHeaderAtHeight` 或 `Update` 方法。 后一种方法下载
来自主要的最新标头并将其与当前受信任的标头进行比较。

```go
type Client interface {
	// verify new headers
	VerifyHeaderAtHeight(height int64, now time.Time) (*types.SignedHeader, error)
	VerifyHeader(newHeader *types.SignedHeader, newVals *types.ValidatorSet, now time.Time) error
	Update(now time.Time) (*types.SignedHeader, error)

	// get trusted headers & validators
	TrustedHeader(height int64) (*types.SignedHeader, error)
	TrustedValidatorSet(height int64) (valSet *types.ValidatorSet, heightUsed int64, err error)
	LastTrustedHeight() (int64, error)
	FirstTrustedHeight() (int64, error)

	// query configuration options
	ChainID() string
	Primary() provider.Provider
	Witnesses() []provider.Provider

	Cleanup() error
}
```

一个新的轻客户端可以从头开始创建(通过 `NewClient`)或者
使用受信任的商店(通过`NewClientFromTrustedStore`)。 当有一些
可信存储中的数据并调用“NewClient”，轻客户端将 a)
检查存储的标头是否更新 b) 可选地在任何时候询问用户
应该回滚(默认不需要确认)。

```go
func NewClient(
	chainID string,
	trustOptions TrustOptions,
	primary provider.Provider,
	witnesses []provider.Provider,
	trustedStore store.Store,
	options ...Option) (*Client, error) {
```

`witnesses` 作为参数(与 `Option` 相反)是一个有意的选择，
默认情况下增加安全性。至少需要一名见证人，
尽管现在，轻客户端不会检查主 != 见证人。
与见证人交叉检查新标题时，见证人的最小数量
需要回应: 1. 注意第一个标头(`TrustOptions.Hash`)是
还与证人交叉核对以增加安全性。

由于二分算法的性质，可能会跳过一些标题。如果光
客户端没有高度“X”和“VerifyHeaderAtHeight(X)”的标头或
`VerifyHeader(H#X)` 方法被调用，这些方法将执行 a) 向后
从最新的标头返回到高度为“X”或 b) 的标头的验证
从第一个存储的标题到高度为“X”的标题的二分验证。

`TrustedHeader`、`TrustedValidatorSet` 仅与受信任的商店通信。
如果某个标头不存在，将返回一个错误，表明
需要验证。

```go
type Provider interface {
	ChainID() string

	SignedHeader(height int64) (*types.SignedHeader, error)
	ValidatorSet(height int64) (*types.ValidatorSet, error)
}
```

Provider 通常是一个完整的节点，但也可以是另一个轻客户端。 以上
接口很薄，可以容纳许多实现。

如果提供者(主要或见证人)长时间不可用
时间，它将被移除以确保顺利运行。

`Client` 和提供者都公开链 ID 以跟踪是否存在相同的链
链。 请注意，当链升级或有意分叉时，链 ID 会发生变化。

轻客户端在可信存储中存储标头和验证器:

```go
type Store interface {
	SaveSignedHeaderAndValidatorSet(sh *types.SignedHeader, valSet *types.ValidatorSet) error
	DeleteSignedHeaderAndValidatorSet(height int64) error

	SignedHeader(height int64) (*types.SignedHeader, error)
	ValidatorSet(height int64) (*types.ValidatorSet, error)

	LastSignedHeaderHeight() (int64, error)
	FirstSignedHeaderHeight() (int64, error)

	SignedHeaderAfter(height int64) (*types.SignedHeader, error)

	Prune(size uint16) error

	Size() uint16
}
```

目前，唯一的实现是 `db` 存储(围绕 KV
数据库，在 Tendermint 中使用)。 将来，远程适配器是可能的
(例如`Postgresql`)。

```go
func Verify(
	chainID string,
	trustedHeader *types.SignedHeader, // height=X
	trustedVals *types.ValidatorSet, // height=X or height=X+1
	untrustedHeader *types.SignedHeader, // height=Y
	untrustedVals *types.ValidatorSet, // height=Y
	trustingPeriod time.Duration,
	now time.Time,
	maxClockDrift time.Duration,
	trustLevel tmmath.Fraction) error {
```

`Verify` 纯函数被公开用于头验证。它同时处理
相邻和非相邻标头的情况。在前一种情况下，它比较
直接散列(2/3+ 签名转换)。否则，它验证 1/3+
(`trustLevel`) 可信验证器仍然存在于新验证器中。

虽然 `Verify` 函数肯定很方便，但 `VerifyAdjacent` 和
`VerifyNonAdjacent` 应该最常使用以避免逻辑错误。

###二分算法细节

尽管规范包含，但实现了非递归二分算法
递归版本。有两个主要原因:

1) 持续的内存消耗 => 没有出现 OOM(Out-Of-Memory)异常的风险；
2)更快的终结(见图1)。

_如图。 1:递归和非递归二分的区别_

！[如图。 1](./img/adr-046-fig1.png)

可以找到非递归二分的规范
[这里](https://github.com/tendermint/spec/blob/zm_non-recursive-verification/spec/consensus/light-client/non-recursive-verification.md)。

## 状态

实施的

## 结果

### 积极的

*单个`Client`结构，易于使用
* 头提供者和可信存储的灵活接口

### 消极的

* `Verify` 需要与当前规范保持一致

### 中性的

* `Verify` 函数可能被误用(在
  错误地实施顺序验证)
