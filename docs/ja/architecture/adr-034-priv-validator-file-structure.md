# ADR 034:PrivValidatorファイル構造

## 変更ログ

2018年3月11日:最初のドラフト

## 環境

現在、PrivValidatorファイル「priv_validator.json」には可変部分と不変部分が含まれています。
ディスク上の秘密鍵が暗号化されていない安全でないモードでも、分離は合理的です
可変部分と不変部分。

参照する:
[#1181](https://github.com/tendermint/tendermint/issues/1181)
[#2657](https://github.com/tendermint/tendermint/issues/2657)
[#2313](https://github.com/tendermint/tendermint/issues/2313)

## 推奨される解決策

可変部分と不変部分を2つの構造で分割できます。
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

次に、 `FilePVKey`と` FilePVLastSignState`を組み合わせて、元の `FilePV`を取得できます。

```go
type FilePV struct {
	Key           FilePVKey
	LastSignState FilePVLastSignState
}
```

前述のように、 `FilePV`は` config`に配置し、 `FilePVLastSignState`は` data`に保存する必要があります。 この
各ファイルのストレージパスは `config.yml`で指定する必要があります。

次に行う必要があるのは、 `FilePV`のメソッドを変更することです。

## ステータス

実装

## 結果

### ポジティブ

-可変と不変のPrivValidatorを分離します

### ネガティブ

-ファイルパスに構成を追加する必要があります

### ニュートラル
