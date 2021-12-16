# 使用 Tendermint

这是从命令行使用“tendermint”程序的指南。
它仅假设您安装了 `tendermint` 二进制文件并且
Tendermint 和 ABCI 是什么的一些基本概念。

您可以使用 `tendermint --help` 查看帮助菜单，以及版本
带有“tendermint 版本”的编号。

## 目录根

区块链数据的默认目录是`~/.tendermint`。 覆盖
这是通过设置`TMHOME`环境变量来实现的。

## 初始化

通过运行初始化根目录:

```sh
tendermint init validator
```

这将创建一个新的私钥(`priv_validator_key.json`)，以及一个
包含相关公钥的创世文件(`genesis.json`)，在
`$TMHOME/config`。 这是运行本地测试网所需的全部内容
有一个验证器。

更详细的初始化，参见 testnet 命令:

```sh
tendermint testnet --help
```

### 创世纪

`$TMHOME/config/` 中的 `genesis.json` 文件定义了初始
TendermintCore 在区块链起源时的状态([参见
定义](https://github.com/tendermint/tendermint/blob/master/types/genesis.go))。

#### 字段

- `genesis_time`:区块链开始的官方时间。
- `chain_id`:区块链的 ID。 **这必须是唯一的
  每个区块链。** 如果您的测试网区块链没有唯一的
  链 ID，你会过得很糟糕。 ChainID 必须少于 50 个符号。
- `initial_height`:Tendermint 应该开始的高度。如果区块链正在进行网络升级，
    从停止的高度开始为以前的高度带来独特性。
- `consensus_params` [规范](https://github.com/tendermint/spec/blob/master/spec/core/state.md#consensusparams)
    -`块`
        - `max_bytes`:最大块大小，以字节为单位。
        - `max_gas`:每个区块的最大gas。
        - `time_iota_ms`:未使用。这已被弃用，并将在未来版本中删除。
    -`证据`
        - `max_age_num_blocks`:证据的最大年龄，以块为单位。基本公式
      计算这个是:MaxAgeDuration / {平均区块时间}。
        - `max_age_duration`:证据的最大年龄，及时。应该对应
      使用应用程序的“解除绑定期”或其他类似的处理机制
      [无利害关系
      攻击](https://github.com/ethereum/wiki/wiki/Proof-of-Stake-FAQ#what-is-the-nothing-at-stake-problem-and-how-can-it-be-fixed )。
        - `max_num`:设置可以提交的最大证据数量
      在单个块中。并且应该舒适地落在最大块下
      当我们考虑每个证据的大小时。
    -`验证器`
        - `pub_key_types`:验证器可以使用的公钥类型。
    -`版本`
        - `app_version`:ABCI 应用程序版本。
- `validators`:初始验证器列表。请注意，这可能会被完全覆盖
  应用程序，并且可以留空以明确表示
  应用程序将使用 ResponseInitChain 初始化验证器集。
    - `pub_key`:第一个元素指定了 `pub_key` 类型。 1
    == Ed25519。第二个元素是公钥字节。
    - `power`:验证者的投票权。
    - `name`:验证器的名称(可选)。
- `app_hash`:预期的应用程序哈希值(由
  `ResponseInfo` ABCI 消息)在创世时。如果应用程序的哈希值
  不匹配，Tendermint 会恐慌。
- `app_state`:应用程序状态(例如初始分发
  代币)。

> :warning: **ChainID 对于每个区块链必须是唯一的。重复使用旧的 chainID 可能会导致问题**

#### 示例 genesis.json

```json
{
  "genesis_time": "2020-04-21T11:17:42.341227868Z",
  "chain_id": "test-chain-ROp9KF",
  "initial_height": "0",
  "consensus_params": {
    "block": {
      "max_bytes": "22020096",
      "max_gas": "-1",
      "time_iota_ms": "1000"
    },
    "evidence": {
      "max_age_num_blocks": "100000",
      "max_age_duration": "172800000000000",
      "max_num": 50,
    },
    "validator": {
      "pub_key_types": [
        "ed25519"
      ]
    }
  },
  "validators": [
    {
      "address": "B547AB87E79F75A4A3198C57A8C2FDAF8628CB47",
      "pub_key": {
        "type": "tendermint/PubKeyEd25519",
        "value": "P/V6GHuZrb8rs/k1oBorxc6vyXMlnzhJmv7LmjELDys="
      },
      "power": "10",
      "name": ""
    }
  ],
  "app_hash": ""
}
```

## Run

要运行 Tendermint 节点，请使用:

```bash
tendermint start
```

默认情况下，Tendermint 将尝试连接到 ABCI 应用程序
`127.0.0.1:26658`。 如果您安装了 `kvstore` ABCI 应用程序，请在
另一个窗口。 如果不这样做，请杀死 Tendermint 并运行一个进程内版本的
`kvstore` 应用程序:

```bash
tendermint start --proxy-app=kvstore
```

几秒钟后，您应该看到块开始流入。注意块
即使没有交易，也会定期生产。 见_No Empty
下面的 Blocks_ 来修改这个设置。

Tendermint 支持 `counter`、`kvstore` 和 `noop` 的进程内版本
以 `abci-cli` 作为示例发布的应用程序。 编译您的应用程序很容易
如果 Tendermint 是用 Go 编写的，则在进程中。 如果您的应用程序未写入
去，在另一个进程中运行它，并使用`--proxy-app`标志来指定
它正在侦听的套接字的地址，例如:

```bash
tendermint start --proxy-app=/var/run/abci.sock
```

您可以通过运行 `tendermint start --help` 来了解支持哪些标志。

## 交易

要发送交易，请使用 `curl` 向 Tendermint RPC 发出请求
服务器，例如:

```sh
curl http://localhost:26657/broadcast_tx_commit?tx=\"abcd\"
```

我们可以在 `/status` 端点看到链的状态:

```sh
curl http://localhost:26657/status | json_pp
```

and the `latest_app_hash` in particular:

```sh
curl http://localhost:26657/status | json_pp | grep latest_app_hash
```

在浏览器中访问 `http://localhost:26657` 以查看其他列表
端点。 有些不带参数(如`/status`)，而另一些则指定
参数名称并使用 `_` 作为占位符。


> 提示:找到 RPC 文档 [此处](https://docs.tendermint.com/master/rpc/)

### 格式化

向RPC接口发送交易时，以下格式规则
必须遵循:

使用 `GET`(在 URL 中带有参数):

要将 UTF8 字符串作为交易数据发送，请将 `tx` 的值括起来
双引号中的参数:

```sh
curl 'http://localhost:26657/broadcast_tx_commit?tx="hello"'
```

它发送一个 5 字节的交易:“h e l l o”\[68 65 6c 6c 6f\]。

请注意，此示例中的 URL 用单引号括起来以防止
shell 从解释双引号。 或者，您可以逃避
带反斜杠的双引号:

```sh
curl http://localhost:26657/broadcast_tx_commit?tx=\"hello\"
```

双引号格式适用于多字节字符，只要它们
是有效的 UTF8，例如:

```sh
curl 'http://localhost:26657/broadcast_tx_commit?tx="€5"'
```

发送一个 4 字节的交易:“€5”(UTF8)\[e2 82 ac 35\]。

任意(非 UTF8)交易数据也可以编码为一串
十六进制数字(每字节 2 位)。 为此，请省略引号
并在十六进制字符串前加上`0x`:

```sh
curl http://localhost:26657/broadcast_tx_commit?tx=0x68656C6C6F
```

发送 5 字节事务:\[68 65 6c 6c 6f\]。

使用“POST”(带有 JSON 中的参数)，交易数据作为 JSON 发送
base64 编码的字符串:

```sh
curl http://localhost:26657 -H 'Content-Type: application/json' --data-binary '{
  "jsonrpc": "2.0",
  "id": "anything",
  "method": "broadcast_tx_commit",
  "params": {
    "tx": "aGVsbG8="
  }
}'
```

它发送相同的 5 字节事务:\[68 65 6c 6c 6f\]。

请注意，交易数据的十六进制编码_不_支持
JSON (`POST`) 请求。

## 重启

> :warning: **UNSAFE** 仅在开发中执行此操作，并且仅当您可以
承担丢失所有区块链数据的代价！


要重置区块链，请停止节点并运行:
```sh
tendermint unsafe_reset_all
```

此命令将删除数据目录并重置私有验证器和
地址簿文件。

## 配置

Tendermint 使用 `config.toml` 进行配置。 有关详细信息，请参阅 [
配置规范](./configuration.md)。

值得注意的选项包括应用程序的套接字地址
(`proxy-app`)，Tendermint peer 的监听地址
(`p2p.laddr`)，以及RPC服务器的监听地址
(`rpc.laddr`)。

配置文件中的某些字段可以用标志覆盖。

## 没有空块

虽然 `tendermint` 的默认行为仍然是创建块
大约每秒一次，可以禁用空块或
设置块创建间隔。 在前一种情况下，块将是
当有新交易或 AppHash 改变时创建。

将 Tendermint 配置为不产生空块，除非有
交易或应用程序哈希更改，使用此运行 Tendermint
附加标志:

```sh
tendermint start --consensus.create_empty_blocks=false
```

or set the configuration via the `config.toml` file:

```toml
[consensus]
create_empty_blocks = false
```

记住:因为默认是_创建空块_，避免
空块需要将配置选项设置为 `false`。

块间隔设置允许延迟(以 time.Duration 格式 [ParseDuration](https://golang.org/pkg/time/#ParseDuration))
创建每个新的空块。 可以使用此附加标志设置它:

```sh
--consensus.create_empty_blocks_interval="5s"
```

或通过 `config.toml` 文件设置配置:

```toml
[consensus]
create_empty_blocks_interval = "5s"
```

使用此设置，如果没有块，将每 5 秒产生一次空块
以其他方式生产，无论其价值如何
`create_empty_blocks`。

## 广播 API

早些时候，我们使用 `broadcast_tx_commit` 端点发送一个
交易。 当交易被发送到 Tendermint 节点时，它会
通过“CheckTx”针对应用程序运行。 如果它通过`CheckTx`，它
将被包含在内存池中，广播给其他节点，以及
最终包含在一个块中。

由于处理交易有多个阶段，我们提供
广播交易的多个端点:

```md
/broadcast_tx_async
/broadcast_tx_sync
/broadcast_tx_commit
```

这些对应于无处理，通过内存池处理，以及
分别通过一个块进行处理。也就是说，`broadcast_tx_async`，
将立即返回而无需等待听到交易是否成功
甚至有效，而 `broadcast_tx_sync` 将返回结果
通过“CheckTx”运行交易。使用`broadcast_tx_commit`
将等到事务在块中提交或直到某些
达到超时，但如果交易完成，将立即返回
不通过`CheckTx`。 `broadcast_tx_commit` 的返回值包括
两个字段，`check_tx` 和 `deliver_tx`，与结果相关
通过这些 ABCI 消息运行事务。

使用 `broadcast_tx_commit` 的好处是请求返回
在事务提交后(即包含在一个块中)，但是
可以采取一秒钟的顺序。为了快速获得结果，请使用
`broadcast_tx_sync`，但直到事务才会提交
之后，到那时它对状态的影响可能会改变。

请注意，内存池不提供强有力的保证 - 仅仅因为通过了 tx
CheckTx(即被接受到内存池中)，并不意味着它会被提交，
因为在内存池中有 tx 的节点可能会在他们提出建议之前崩溃。
有关更多信息，请参阅 [mempool
预写日志](../tendermint-core/running-in-production.md#mempool-wal)

## Tendermint 网络

当 `tendermint init` 运​​行时，一个 `genesis.json` 和
`priv_validator_key.json` 在 `~/.tendermint/config` 中创建。这
`genesis.json` 可能看起来像:

```json
{
  "validators" : [
    {
      "pub_key" : {
        "value" : "h3hk+QE8c6QLTySp8TcfzclJw/BG79ziGB/pIA+DfPE=",
        "type" : "tendermint/PubKeyEd25519"
      },
      "power" : 10,
      "name" : ""
    }
  ],
  "app_hash" : "",
  "chain_id" : "test-chain-rDlYSN",
  "genesis_time" : "0001-01-01T00:00:00Z"
}
```

And the `priv_validator_key.json`:

```json
{
  "last_step" : 0,
  "last_round" : "0",
  "address" : "B788DEDE4F50AD8BC9462DE76741CCAFF87D51E2",
  "pub_key" : {
    "value" : "h3hk+QE8c6QLTySp8TcfzclJw/BG79ziGB/pIA+DfPE=",
    "type" : "tendermint/PubKeyEd25519"
  },
  "last_height" : "0",
  "priv_key" : {
    "value" : "JPivl82x+LfVkp8i3ztoTjY6c6GJ4pBxQexErOCyhwqHeGT5ATxzpAtPJKnxNx/NyUnD8Ebv3OIYH+kgD4N88Q==",
    "type" : "tendermint/PrivKeyEd25519"
  }
}
```

`priv_validator_key.json` 实际上包含一个私钥，并且应该
因此绝对保密；现在我们使用纯文本。
请注意 `last_` 字段，用于防止我们签名
相互矛盾的消息。

还要注意，`pub_key`(公钥)在
`priv_validator_key.json` 也存在于 `genesis.json` 中。

创世文件包含可能参与的公钥列表
在共识中，以及他们相应的投票权。大于 2/3
的投票权必须处于活动状态(即相应的私钥
必须产生签名)以达成共识才能取得进展。在我们的
在这种情况下，创世文件包含我们的公钥
`priv_validator_key.json`，所以一个 Tendermint 节点以默认值开始
根目录就能进。投票权使用 int64
但必须是正数，因此范围是:0 到 9223372036854775807。
由于当前提议者选择算法的工作方式，我们不
建议投票权大于 10\^12(即 1 万亿)。

如果我们想在网络中添加更多节点，我们有两种选择:我们可以
添加一个新的验证器节点，该节点也将通过以下方式参与共识
提议区块并对它们进行投票，或者我们可以添加一个新的非验证器
节点，不会直接参与，但会验证和跟上
与共识协议。

### 同行

#### 种子

种子节点是中继他们知道的其他对等点的地址的节点
的。这些节点不断地爬行网络以试图获得更多的对等点。这
种子节点中继的地址保存到本地地址簿中。一次
这些在地址簿中，您将直接连接到这些地址。
基本上种子节点的工作只是中继每个人的地址。你不会
收到足够的地址后连接到种子节点，因此通常您
只在第一次启动时需要它们。种子节点会立即断开
在向您发送一些地址后从您那里。

#### 持久对等体

持久的同龄人是您想要不断联系的人。如果你
断开连接，您将尝试直接连接回它们，而不是使用
地址簿中的另一个地址。在重新启动时，您将始终尝试
无论您的地址簿大小如何，都可以连接到这些对等点。

默认情况下，所有对等点都会中继他们知道的对等点。这称为对等交换
协议 (PeX)。使用 PeX，同行将八卦已知的同行并形成
一个网络，在 addrbook 中存储对等地址。正因为如此，你不
如果您有一个实时的持久对等点，则必须使用种子节点。

#### 连接到对等点

要在启动时连接到对等点，请在
`$TMHOME/config/config.toml` 或在命令行上。使用“种子”来
指定种子节点，和
`persistent-peers` 指定你的节点将维护的对等点
与的持久连接。

例如，

```sh
tendermint start --p2p.seeds "f9baeaa15fedf5e1ef7448dd60f46c01f1a9e9c4@1.2.3.4:26656,0491d373a8e0fcf1023aaf18c51d6a1d0d4f31bd@5.6.7.8:26656"
```

或者，您可以使用 RPC 的 `/dial_seeds` 端点来
为正在运行的节点指定要连接到的种子:

```sh
curl 'localhost:26657/dial_seeds?seeds=\["f9baeaa15fedf5e1ef7448dd60f46c01f1a9e9c4@1.2.3.4:26656","0491d373a8e0fcf1023aaf18c51d6a1d0d4f31bd@5.6.7.8:26656"\]'
```

请注意，启用 PeX 后，您
第一次启动后应该不需要种子。

如果您希望 Tendermint 连接到一组特定的地址和
与每个保持持久连接，您可以使用
`--p2p.persistent-peers` 标志或相应的设置
`config.toml` 或 `/dial_peers` RPC 端点可以在没有
停止 Tendermint 核心实例。

```sh
tendermint start --p2p.persistent-peers "429fcf25974313b95673f58d77eacdd434402665@10.11.12.13:26656,96663a3dd0d7b9d17d4c8211b191af259621c693@10.11.12.14:26656"

curl 'localhost:26657/dial_peers?persistent=true&peers=\["429fcf25974313b95673f58d77eacdd434402665@10.11.12.13:26656","96663a3dd0d7b9d17d4c8211b191af259621c693@10.11.12.14:26656"\]'
```

### 添加一个非验证器

添加非验证器很简单。 只需复制原始的`genesis.json`
到新机器上的`~/.tendermint/config` 并启动节点，
根据需要指定种子或持久对等点。 如果没有种子或
指定了持久对等点，节点不会产生任何块，因为
它不是验证者，也不会听到任何区块的消息，因为它是
未连接到其他对等方。

### 添加验证器

添加新验证器的最简单方法是在 `genesis.json` 中进行，
在启动网络之前。 例如，我们可以创建一个新的
`priv_validator_key.json`，并将它的 `pub_key` 复制到上面的 genesis 中。

我们可以使用以下命令生成一个新的 `priv_validator_key.json`:

```sh
tendermint gen_validator
```

现在我们可以更新我们的创世文件。 例如，如果新
`priv_validator_key.json` 看起来像:

```json
{
  "address" : "5AF49D2A2D4F5AD4C7C8C4CC2FB020131E9C4902",
  "pub_key" : {
    "value" : "l9X9+fjkeBzDfPGbUM7AMIRE6uJN78zN5+lk5OYotek=",
    "type" : "tendermint/PubKeyEd25519"
  },
  "priv_key" : {
    "value" : "EDJY9W6zlAw+su6ITgTKg2nTZcHAH1NMTW5iwlgmNDuX1f35+OR4HMN88ZtQzsAwhETq4k3vzM3n6WTk5ii16Q==",
    "type" : "tendermint/PrivKeyEd25519"
  },
  "last_step" : 0,
  "last_round" : "0",
  "last_height" : "0"
}
```

那么新的 `genesis.json` 将是:

```json
{
  "validators" : [
    {
      "pub_key" : {
        "value" : "h3hk+QE8c6QLTySp8TcfzclJw/BG79ziGB/pIA+DfPE=",
        "type" : "tendermint/PubKeyEd25519"
      },
      "power" : 10,
      "name" : ""
    },
    {
      "pub_key" : {
        "value" : "l9X9+fjkeBzDfPGbUM7AMIRE6uJN78zN5+lk5OYotek=",
        "type" : "tendermint/PubKeyEd25519"
      },
      "power" : 10,
      "name" : ""
    }
  ],
  "app_hash" : "",
  "chain_id" : "test-chain-rDlYSN",
  "genesis_time" : "0001-01-01T00:00:00Z"
}
```

更新 `~/.tendermint/config` 中的 `genesis.json`。复制起源
文件和新的 `priv_validator_key.json` 到 `~/.tendermint/config` 上
一台新机器。

现在在两台机器上运行 `tendermint start`，并使用任一
`--p2p.persistent-peers` 或 `/dial_peers` 来让他们对等。
他们应该开始制作积木，并且只会继续这样做
因为他们都在线。

制作一个可以容忍其中一个验证者的 Tendermint 网络
失败，您至少需要四个验证器节点(例如，2/3)。

支持在实时网络中更新验证器，但必须
由应用程序开发人员明确编程。

### 本地网络

要在本地运行网络，比如在一台机器上，你必须改变 `_laddr`
`config.toml` 中的字段(或使用标志)以便监听
各种套接字的地址不冲突。此外，您必须设置
`config.toml` 中的 `addr_book_strict=false`，否则为 Tendermint 的 p2p
库将拒绝与具有相同 IP 地址的对等点建立连接。

### 升级

见
[升级.md](https://github.com/tendermint/tendermint/blob/master/UPGRADING.md)
指导。您可能需要在重大突破性版本之间重置您的链。
虽然，我们预计 Tendermint 在未来会有更少的突破性版本
(尤其是在 1.0 版本之后)。
