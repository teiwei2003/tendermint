# 快速开始

## 概述

这是一个快速入门指南。 如果您对 Tendermint
有效并想立即开始，请继续。 确保你已经安装了二进制文件。
如果还没有，请查看 [install](./install.md)。

## 初始化

跑步:

```sh
tendermint init validator
```

将为单个本地节点创建所需的文件。

这些文件位于 `$HOME/.tendermin` 中:

```sh
$ ls $HOME/.tendermint

config  data

$ ls $HOME/.tendermint/config/

config.toml  genesis.json  node_key.json  priv_validator.json
```

For a single, local node, no further configuration is required.
Configuring a cluster is covered further below.

## Local Node

Start Tendermint with a simple in-process application:

```sh
tendermint start --proxy-app=kvstore
```

> Note: `kvstore` is a non persistent app, if you would like to run an application with persistence run `--proxy-app=persistent_kvstore`

and blocks will start to stream in:

```sh
I[01-06|01:45:15.592] Executed block                               module=state height=1 validTxs=0 invalidTxs=0
I[01-06|01:45:15.624] Committed state                              module=state height=1 txs=0 appHash=
```

Check the status with:

```sh
curl -s localhost:26657/status
```

### Sending Transactions

With the KVstore app running, we can send transactions:

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="abcd"'
```

and check that it worked with:

```sh
curl -s 'localhost:26657/abci_query?data="abcd"'
```

We can send transactions with a key and value too:

```sh
curl -s 'localhost:26657/broadcast_tx_commit?tx="name=satoshi"'
```

and query the key:

```sh
curl -s 'localhost:26657/abci_query?data="name"'
```

where the value is returned in hex.

## Cluster of Nodes

首先创建四台Ubuntu云机器。 以下是在数字上测试的
Ocean Ubuntu 16.04 x64(3GB/1CPU，20GB SSD)。 我们会参考他们各自的IP
以下地址为 IP1、IP2、IP3、IP4。

然后，`ssh` 进入每台机器，并执行[这个脚本](https://git.io/fFfOR):

```sh
curl -L https://git.io/fFfOR | bash
source ~/.profile
```

这将安装 `go` 和其他依赖项，获取 Tendermint 源代码，然后编译 `tendermint` 二进制文件。

接下来，使用`tendermint testnet`命令创建配置文件的四个目录(在`./mytestnet`中找到)，并将每个目录复制到云端的相关机器上，这样每台机器都有`$HOME/mytestnet/node[ 0-3]` 目录。

在您可以启动网络之前，您需要对等标识符(IP 是不够的，可以更改)。 我们将它们称为 ID1、ID2、ID3、ID4。

```sh
tendermint show_node_id --home ./mytestnet/node0
tendermint show_node_id --home ./mytestnet/node1
tendermint show_node_id --home ./mytestnet/node2
tendermint show_node_id --home ./mytestnet/node3
```

Finally, from each machine, run:

```sh
tendermint start --home ./mytestnet/node0 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
tendermint start --home ./mytestnet/node1 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
tendermint start --home ./mytestnet/node2 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
tendermint start --home ./mytestnet/node3 --proxy-app=kvstore --p2p.persistent-peers="ID1@IP1:26656,ID2@IP2:26656,ID3@IP3:26656,ID4@IP4:26656"
```

注意，在第三个节点启动后，块将开始流入
因为 >2/3 的验证器(在 `genesis.json` 中定义)已经上线。
持久对等点也可以在`config.toml` 中指定。 有关配置选项的更多信息，请参见 [此处](../tendermint-core/configuration.md)。

然后可以发送交易，如上面单个本地节点示例中所述。
