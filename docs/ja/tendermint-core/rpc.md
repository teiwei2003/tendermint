# RPC

RPCドキュメントはここでホストされています:

-[https://docs.tendermint.com/master/rpc/](https://docs.tendermint.com/master/rpc/)

ドキュメントを更新するには、[rpc/coreディレクトリ](https://github.com/tendermint/tendermint/tree/master/rpc/core)の関連する `godoc`コメントを編集してください。

Tendermintをインプロセスで使用する場合は、RPCに表示されるバージョンを設定する必要があります。

goプロジェクトでmakefileを使用する場合、これはsedと `ldflags`を使用して実行できます。

例:

```
VERSION := $(shell go list -m github.com/tendermint/tendermint | sed 's:.* ::')
LD_FLAGS = -X github.com/tendermint/tendermint/version.TMCoreSemVer=$(VERSION)

install:
  @echo "Installing the brr machine"
  @go install -mod=readonly -ldflags "$(LD_FLAGS)" ./cmd/<app>
.PHONY: install
```
