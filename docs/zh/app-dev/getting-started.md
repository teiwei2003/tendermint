# 入门

## 第一个 Tendermint 应用程序

作为一个通用的区块链引擎，Tendermint 是不可知的
要运行的应用程序。 所以，要运行一个完整的区块链，
一些有用的东西，你必须启动两个程序:一个是 Tendermint Core，
另一个是你的应用程序，它可以用任何程序编写
语。 回忆起[介绍到
ABCI](../introduction/what-is-tendermint.md#abci-overview) Tendermint Core 处理所有 p2p 和共识的东西，只是将交易转发到
应用程序需要验证时，或者当他们准备好时
提交到一个块。

在本指南中，我们向您展示了一些如何运行应用程序的示例
使用 Tendermint。

### 安装

我们将使用的第一个应用程序是用 Go 编写的。 要安装它们，您
需要[安装Go](https://golang.org/doc/install)，把
在你的 `$PATH` 中添加 `$GOPATH/bin` 并使用以下说明启用 go 模块:

```bash
echo export GOPATH=\"\$HOME/go\" >> ~/.bash_profile
echo export PATH=\"\$PATH:\$GOPATH/bin\" >> ~/.bash_profile
```

Then run

```sh
go get github.com/tendermint/tendermint
cd $GOPATH/src/github.com/tendermint/tendermint
make install_abci
```

现在你应该已经安装了 `abci-cli`； 你会注意到`kvstore`
命令，编写的示例应用程序
在去。 有关用 JavaScript 编写的应用程序，请参见下文。

现在，让我们运行一些应用程序！

## KVStore - 第一个例子

kvstore 应用程序是一个 [Merkle
tree](https://en.wikipedia.org/wiki/Merkle_tree) 只存储所有
交易。 如果交易包含`=`，例如 `key=value`，然后
`value` 存储在 Merkle 树的 `key` 下。 否则，
完整的交易字节存储为键和值。

让我们启动一个 kvstore 应用程序。

```sh
abci-cli kvstore
```

在另一个终端中，我们可以启动 Tendermint。 你应该已经有了
已安装 Tendermint 二进制文件。 如果没有，请按照以下步骤操作
[这里](../introduction/install.md)。 如果您从未运行过 Tendermint
使用前:

```sh
tendermint init validator
tendermint start
```

如果您使用过 Tendermint，您可能需要为新的数据重置数据
通过运行“tendermint unsafe_reset_all”来实现区块链。 然后你可以运行
`tendermint start` 启动 Tendermint，并连接到应用程序。 更多
详细信息，请参阅[使用 Tendermint 的指南](../tendermint-core/using-tendermint.md)。

你应该看到 Tendermint 制作积木块！ 我们可以得到我们的状态
Tendermint节点如下:

```sh
curl -s localhost:26657/status
```

`-s` 只是让 `curl` 静音。 为了更好的输出，将结果通过管道传输到一个
像 [jq](https://stedolan.github.io/jq/) 或 `json_pp` 这样的工具。

现在让我们向 kvstore 发送一些交易。

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="abcd"'
```

注意 url 周围的单引号 (`'`)，它确保
双引号 (`"`) 不会被 bash 转义。这个命令发送了一个
带有字节 `abcd` 的事务，因此 `abcd` 将被存储为两个键
以及 Merkle 树中的值。 响应应该看起来有些东西
喜欢:

```json
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "check_tx": {},
    "deliver_tx": {
      "tags": [
        {
          "key": "YXBwLmNyZWF0b3I=",
          "value": "amFl"
        },
        {
          "key": "YXBwLmtleQ==",
          "value": "YWJjZA=="
        }
      ]
    },
    "hash": "9DF66553F98DE3C26E3C3317A3E4CED54F714E39",
    "height": 14
  }
}
```

我们可以确认我们的交易有效并且价值被存储
查询应用程序:

```sh
curl -s 'localhost:26657/abci_query?data="abcd"'
```

The result should look like:

```json
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "response": {
      "log": "exists",
      "index": "-1",
      "key": "YWJjZA==",
      "value": "YWJjZA=="
    }
  }
}
```

注意结果中的`value`(`YWJjZA==`)； 这是 base64 编码
`abcd` 的 ASCII 码。 您可以通过以下方式在 python 2 shell 中验证这一点
运行 `"YWJjZA==".decode('base64')` 或在 python 3 shell 中运行
`导入编解码器； codecs.decode(b"YWJjZA==", 'base64').decode('ascii')`。
请继续关注[使此输出更多
人类可读](https://github.com/tendermint/tendermint/issues/1794)。

现在让我们尝试设置不同的键和值:

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="name=satoshi"'
```

现在如果我们查询`name`，我们应该得到`satoshi`，或者`c2F0b3NoaQ==`
在 base64 中:

```sh
curl -s 'localhost:26657/abci_query?data="name"'
```

尝试一些其他事务和查询以确保一切正常
在职的！


## CounterJS - 另一种语言的示例

我们还想以另一种语言运行应用程序 - 在这种情况下，
我们将运行 `counter` 的 Javascript 版本。 要运行它，你需要
[安装节点](https://nodejs.org/en/download/)。

您还需要从
[这里](https://github.com/tendermint/js-abci)，然后安装:

```sh
git clone https://github.com/tendermint/js-abci.git
cd js-abci
npm install abci
```

杀死之前的 `counter` 和 `tendermint` 进程。 现在运行应用程序:

```sh
node example/counter.js
```

在另一个窗口中，重置并启动“tendermint”:

```sh
tendermint unsafe_reset_all
tendermint start
```

再一次，你应该看到块流过——但现在，我们的
应用程序是用 Javascript 编写的！ 尝试发送一些交易，然后
像以前一样 - 结果应该是一样的:

```sh
# ok
curl localhost:26657/broadcast_tx_commit?tx=0x00
# invalid nonce
curl localhost:26657/broadcast_tx_commit?tx=0x05
# ok
curl localhost:26657/broadcast_tx_commit?tx=0x01
```

Neat, eh?
