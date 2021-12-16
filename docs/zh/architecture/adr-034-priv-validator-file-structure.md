# ADR 034:PrivValidator 文件结构

## 变更日志

03-11-2018:初稿

## 语境

目前，PrivValidator 文件“priv_validator.json”包含可变和不可变部分。
即使在不加密磁盘上的私钥的不安全模式下，分离也是合理的
可变部分和不可变部分。

参考:
[#1181](https://github.com/tendermint/tendermint/issues/1181)
[#2657](https://github.com/tendermint/tendermint/issues/2657)
[#2313](https://github.com/tendermint/tendermint/issues/2313)

## 建议的解决方案

我们可以用两个结构体拆分可变部分和不可变部分:
```go
// FilePVKey stores the immutable part of PrivValidator
type FilePVKey struct {
	Address types.Address  `json:"address"`
	PubKey  crypto.PubKey  `json:"pub_key"`
	PrivKey crypto.PrivKey `json:"priv_key"`

	filePath string
}

// FilePVState stores the mutable part of PrivValidator
type FilePVLastSignState struct {
	Height    int64        `json:"height"`
	Round     int          `json:"round"`
	Step      int8         `json:"step"`
	Signature []byte       `json:"signature,omitempty"`
	SignBytes cmn.HexBytes `json:"signbytes,omitempty"`

	filePath string
	mtx      sync.Mutex
}
```

然后我们可以将`FilePVKey` 和`FilePVLastSignState` 结合起来，就可以得到原来的`FilePV`。

```go
type FilePV struct {
	Key           FilePVKey
	LastSignState FilePVLastSignState
}
```

如前所述，`FilePV` 应该位于`config` 中，而`FilePVLastSignState` 应该存储在`data` 中。 这
每个文件的存储路径应该在`config.yml`中指定。

接下来我们需要做的是改变`FilePV`的方法。

## 状态

实施的

## 结果

### 积极的

- 分离 PrivValidator 的可变和不可变

### 消极的

- 需要为文件路径添加更多配置

### 中性的
