# 安装 Tendermint

## 来自二进制

要下载预构建的二进制文件，请参阅 [发布页面](https://github.com/tendermint/tendermint/releases)。

## 使用自制软件

您还可以通过简单地使用自制软件来安装 Tendermint 二进制文件，

```
brew install tendermint
```

## 来自源码

你需要 `go` [已安装](https://golang.org/doc/install) 和所需的
设置环境变量，可以使用以下命令完成:

```sh
echo export GOPATH=\"\$HOME/go\" >> ~/.bash_profile
echo export PATH=\"\$PATH:\$GOPATH/bin\" >> ~/.bash_profile
```

获取源代码:

```sh
git clone https://github.com/tendermint/tendermint.git
cd tendermint
```

然后运行:

```sh
make install
```

将二进制文件放入 `$GOPATH/bin` 或使用:

```sh
make build
```

将二进制文件放入`./build`。

_免责声明_ Tendermint 的二进制文件是在没有 DWARF 的情况下构建/安装的
符号表。 如果您想使用 DWARF 构建/安装 Tendermint
符号和调试信息，从make中的`BUILD_FLAGS`中删除`-s -w`
文件。

现在安装了最新的 Tendermint。 您可以通过以下方式验证安装
跑步:

```sh
tendermint version
```

## 运行

要使用简单的进程内应用程序启动单节点区块链:

```sh
tendermint init validator
tendermint start --proxy-app=kvstore
```

## 重新安装

如果您已经安装了 Tendermint，并且您进行了更新，只需

```sh
make install
```

要升级，请运行

```sh
git pull origin master
make install
```

## 使用 CLevelDB 支持编译

安装 [LevelDB](https://github.com/google/leveldb)(最低版本为 1.7)。

使用 snappy(可选)安装 LevelDB。 以下是 Ubuntu 的命令:

```sh
sudo apt-get update
sudo apt install build-essential

sudo apt-get install libsnappy-dev

wget https://github.com/google/leveldb/archive/v1.20.tar.gz && \
  tar -zxvf v1.20.tar.gz && \
  cd leveldb-1.20/ && \
  make && \
  sudo cp -r out-static/lib* out-shared/lib* /usr/local/lib/ && \
  cd include/ && \
  sudo cp -r leveldb /usr/local/include/ && \
  sudo ldconfig && \
  rm -f v1.20.tar.gz
```

将数据库后端设置为 `cleveldb`:

```toml
# config/config.toml
db_backend = "cleveldb"
```

要安装 Tendermint，请运行:

```sh
CGO_LDFLAGS="-lsnappy" make install TENDERMINT_BUILD_OPTIONS=cleveldb
```

或运行:

```sh
CGO_LDFLAGS="-lsnappy" make build TENDERMINT_BUILD_OPTIONS=cleveldb
```

which puts the binary in `./build`.
