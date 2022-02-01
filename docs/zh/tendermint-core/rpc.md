# RPC

RPC 文档托管在此处:

- [https://docs.tendermint.com/master/rpc/](https://docs.tendermint.com/master/rpc/)

要更新文档，请编辑 [rpc/core 目录](https://github.com/tendermint/tendermint/tree/master/rpc/core) 中的相关 `godoc` 注释.

如果您在进程内使用 Tendermint，则需要设置要在 RPC 中显示的版本.

如果你在你的 go 项目中使用 makefile，这可以通过使用 sed 和 `ldflags` 来完成.

例子:

```
VERSION := $(shell go list -m github.com/tendermint/tendermint | sed 's:.* ::')
LD_FLAGS = -X github.com/tendermint/tendermint/version.TMCoreSemVer=$(VERSION)

install:
  @echo "Installing the brr machine"
  @go install -mod=readonly -ldflags "$(LD_FLAGS)" ./cmd/<app>
.PHONY: install
```
