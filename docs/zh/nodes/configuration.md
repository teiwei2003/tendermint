# 配置

Tendermint Core 可以通过 TOML 文件进行配置
`$TMHOME/config/config.toml`。 其中一些参数可以被覆盖
命令行标志。 对于大多数用户来说，`##### 主要基本配置选项 #####` 中的选项旨在在配置选项时进行修改
下文进一步供高级高级用户使用。

## 选项

`tendermint init` 创建的默认配置文件包含所有
参数设置为其默认值。 它会看起来有些东西
就像下面的文件，但是，通过检查
`config.toml` 使用你安装的 `tendermin` 版本创建:

```toml# This is a TOML config file.
# For more information, see https://github.com/toml-lang/toml

# NOTE: Any path below can be absolute (e.g. "/var/myawesomeapp/data") or
# relative to the home directory (e.g. "data"). The home directory is
# "$HOME/.tendermint" by default, but could be changed via $TMHOME env variable
# or --home cmd flag.

#######################################################################
###                   Main Base Config Options                      ###
#######################################################################

# TCP or UNIX socket address of the ABCI application,
# or the name of an ABCI application compiled in with the Tendermint binary
proxy-app = "tcp://127.0.0.1:26658"

# A custom human readable name for this node
moniker = "ape"


# Mode of Node: full | validator | seed (default: "validator")
# * validator node (default)
#   - all reactors
#   - with priv_validator_key.json, priv_validator_state.json
# * full node
#   - all reactors
#   - No priv_validator_key.json, priv_validator_state.json
# * seed node
#   - only P2P, PEX Reactor
#   - No priv_validator_key.json, priv_validator_state.json
mode = "validator"

# If this node is many blocks behind the tip of the chain, FastSync
# allows them to catchup quickly by downloading blocks in parallel
# and verifying their commits
fast-sync = true

# Database backend: goleveldb | cleveldb | boltdb | rocksdb | badgerdb
# * goleveldb (github.com/syndtr/goleveldb - most popular implementation)
#   - pure go
#   - stable
# * cleveldb (uses levigo wrapper)
#   - fast
#   - requires gcc
#   - use cleveldb build tag (go build -tags cleveldb)
# * boltdb (uses etcd's fork of bolt - github.com/etcd-io/bbolt)
#   - EXPERIMENTAL
#   - may be faster is some use-cases (random reads - indexer)
#   - use boltdb build tag (go build -tags boltdb)
# * rocksdb (uses github.com/tecbot/gorocksdb)
#   - EXPERIMENTAL
#   - requires gcc
#   - use rocksdb build tag (go build -tags rocksdb)
# * badgerdb (uses github.com/dgraph-io/badger)
#   - EXPERIMENTAL
#   - use badgerdb build tag (go build -tags badgerdb)
db-backend = "goleveldb"

# Database directory
db-dir = "data"

# Output level for logging, including package level options
log-level = "info"

# Output format: 'plain' (colored text) or 'json'
log-format = "plain"

##### additional base config options #####

# Path to the JSON file containing the initial validator set and other meta data
genesis-file = "config/genesis.json"

# Path to the JSON file containing the private key to use for node authentication in the p2p protocol
node-key-file = "config/node_key.json"

# Mechanism to connect to the ABCI application: socket | grpc
abci = "socket"

# If true, query the ABCI app on connecting to a new peer
# so the app can decide if we should keep the connection or not
filter-peers = false


#######################################################
###       Priv Validator Configuration              ###
#######################################################
[priv-validator]

# Path to the JSON file containing the private key to use as a validator in the consensus protocol
key-file = "config/priv_validator_key.json"

# Path to the JSON file containing the last sign state of a validator
state-file = "data/priv_validator_state.json"

# TCP or UNIX socket address for Tendermint to listen on for
# connections from an external PrivValidator process
# when the listenAddr is prefixed with grpc instead of tcp it will use the gRPC Client
laddr = ""

# Path to the client certificate generated while creating needed files for secure connection.
# If a remote validator address is provided but no certificate, the connection will be insecure
client-certificate-file = ""

# Client key generated while creating certificates for secure connection
validator-client-key-file = ""

# Path to the Root Certificate Authority used to sign both client and server certificates
certificate-authority = ""


#######################################################################
###                 Advanced Configuration Options                  ###
#######################################################################

#######################################################
###       RPC Server Configuration Options          ###
#######################################################
[rpc]

# TCP or UNIX socket address for the RPC server to listen on
laddr = "tcp://127.0.0.1:26657"

# A list of origins a cross-domain request can be executed from
# Default value '[]' disables cors support
# Use '["*"]' to allow any origin
cors-allowed-origins = []

# A list of methods the client is allowed to use with cross-domain requests
cors-allowed-methods = ["HEAD", "GET", "POST", ]

# A list of non simple headers the client is allowed to use with cross-domain requests
cors-allowed-headers = ["Origin", "Accept", "Content-Type", "X-Requested-With", "X-Server-Time", ]

# TCP or UNIX socket address for the gRPC server to listen on
# NOTE: This server only supports /broadcast_tx_commit
# Deprecated gRPC  in the RPC layer of Tendermint will be deprecated in 0.36.
grpc-laddr = ""

# Maximum number of simultaneous connections.
# Does not include RPC (HTTP&WebSocket) connections. See max-open-connections
# If you want to accept a larger number than the default, make sure
# you increase your OS limits.
# 0 - unlimited.
# Should be < {ulimit -Sn} - {MaxNumInboundPeers} - {MaxNumOutboundPeers} - {N of wal, db and other open files}
# 1024 - 40 - 10 - 50 = 924 = ~900
# Deprecated gRPC  in the RPC layer of Tendermint will be deprecated in 0.36.
grpc-max-open-connections = 900

# Activate unsafe RPC commands like /dial-seeds and /unsafe-flush-mempool
unsafe = false

# Maximum number of simultaneous connections (including WebSocket).
# Does not include gRPC connections. See grpc-max-open-connections
# If you want to accept a larger number than the default, make sure
# you increase your OS limits.
# 0 - unlimited.
# Should be < {ulimit -Sn} - {MaxNumInboundPeers} - {MaxNumOutboundPeers} - {N of wal, db and other open files}
# 1024 - 40 - 10 - 50 = 924 = ~900
max-open-connections = 900

# Maximum number of unique clientIDs that can /subscribe
# If you're using /broadcast_tx_commit, set to the estimated maximum number
# of broadcast_tx_commit calls per block.
max-subscription-clients = 100

# Maximum number of unique queries a given client can /subscribe to
# If you're using GRPC (or Local RPC client) and /broadcast_tx_commit, set to
# the estimated # maximum number of broadcast_tx_commit calls per block.
max-subscriptions-per-client = 5

# How long to wait for a tx to be committed during /broadcast_tx_commit.
# WARNING: Using a value larger than 10s will result in increasing the
# global HTTP write timeout, which applies to all connections and endpoints.
# See https://github.com/tendermint/tendermint/issues/3435
timeout-broadcast-tx-commit = "10s"

# Maximum size of request body, in bytes
max-body-bytes = 1000000

# Maximum size of request header, in bytes
max-header-bytes = 1048576

# The path to a file containing certificate that is used to create the HTTPS server.
# Might be either absolute path or path related to Tendermint's config directory.
# If the certificate is signed by a certificate authority,
# the certFile should be the concatenation of the server's certificate, any intermediates,
# and the CA's certificate.
# NOTE: both tls-cert-file and tls-key-file must be present for Tendermint to create HTTPS server.
# Otherwise, HTTP server is run.
tls-cert-file = ""

# The path to a file containing matching private key that is used to create the HTTPS server.
# Might be either absolute path or path related to Tendermint's config directory.
# NOTE: both tls-cert-file and tls-key-file must be present for Tendermint to create HTTPS server.
# Otherwise, HTTP server is run.
tls-key-file = ""

# pprof listen address (https://golang.org/pkg/net/http/pprof)
pprof-laddr = ""

#######################################################
###           P2P Configuration Options             ###
#######################################################
[p2p]

# Select the p2p internal queue
queue-type = "priority"

# Address to listen for incoming connections
laddr = "tcp://0.0.0.0:26656"

# Address to advertise to peers for them to dial
# If empty, will use the same port as the laddr,
# and will introspect on the listener or use UPnP
# to figure out the address. ip and port are required
# example: 159.89.10.97:26656
external-address = ""

# Comma separated list of seed nodes to connect to
# We only use these if we can’t connect to peers in the addrbook
# NOTE: not used by the new PEX reactor. Please use BootstrapPeers instead.
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
seeds = ""

# Comma separated list of peers to be added to the peer store
# on startup. Either BootstrapPeers or PersistentPeers are
# needed for peer discovery
bootstrap-peers = ""

# Comma separated list of nodes to keep persistent connections to
persistent-peers = ""

# UPNP port forwarding
upnp = false

# Path to address book
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
addr-book-file = "config/addrbook.json"

# Set true for strict address routability rules
# Set false for private or local networks
addr-book-strict = true

# Maximum number of inbound peers
#
# TODO: Remove once p2p refactor is complete in favor of MaxConnections.
# ref: https://github.com/tendermint/tendermint/issues/5670
max-num-inbound-peers = 40

# Maximum number of outbound peers to connect to, excluding persistent peers
#
# TODO: Remove once p2p refactor is complete in favor of MaxConnections.
# ref: https://github.com/tendermint/tendermint/issues/5670
max-num-outbound-peers = 10

# Maximum number of connections (inbound and outbound).
max-connections = 64

# Rate limits the number of incoming connection attempts per IP address.
max-incoming-connection-attempts = 100

# List of node IDs, to which a connection will be (re)established ignoring any existing limits
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
unconditional-peer-ids = ""

# Maximum pause when redialing a persistent peer (if zero, exponential backoff is used)
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
persistent-peers-max-dial-period = "0s"

# Time to wait before flushing messages out on the connection
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
flush-throttle-timeout = "100ms"

# Maximum size of a message packet payload, in bytes
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
max-packet-msg-payload-size = 1400

# Rate at which packets can be sent, in bytes/second
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
send-rate = 5120000

# Rate at which packets can be received, in bytes/second
# TODO: Remove once p2p refactor is complete
# ref: https:#github.com/tendermint/tendermint/issues/5670
recv-rate = 5120000

# Set true to enable the peer-exchange reactor
pex = true

# Comma separated list of peer IDs to keep private (will not be gossiped to other peers)
# Warning: IPs will be exposed at /net_info, for more information https://github.com/tendermint/tendermint/issues/3055
private-peer-ids = ""

# Toggle to disable guard against peers connecting from the same ip.
allow-duplicate-ip = false

# Peer connection configuration.
handshake-timeout = "20s"
dial-timeout = "3s"

#######################################################
###          Mempool Configuration Option          ###
#######################################################
[mempool]

# Mempool version to use:
#   1) "v0" - The legacy non-prioritized mempool reactor.
#   2) "v1" (default) - The prioritized mempool reactor.
version = "v1"

recheck = true
broadcast = true

# Maximum number of transactions in the mempool
size = 5000

# Limit the total size of all txs in the mempool.
# This only accounts for raw transactions (e.g. given 1MB transactions and
# max-txs-bytes=5MB, mempool will only accept 5 transactions).
max-txs-bytes = 1073741824

# Size of the cache (used to filter transactions we saw earlier) in transactions
cache-size = 10000

# Do not remove invalid transactions from the cache (default: false)
# Set to true if it's not possible for any invalid transaction to become valid
# again in the future.
keep-invalid-txs-in-cache = false

# Maximum size of a single transaction.
# NOTE: the max size of a tx transmitted over the network is {max-tx-bytes}.
max-tx-bytes = 1048576

# Maximum size of a batch of transactions to send to a peer
# Including space needed by encoding (one varint per transaction).
# XXX: Unused due to https://github.com/tendermint/tendermint/issues/5796
max-batch-bytes = 0

# ttl-duration, if non-zero, defines the maximum amount of time a transaction
# can exist for in the mempool.
#
# Note, if ttl-num-blocks is also defined, a transaction will be removed if it
# has existed in the mempool at least ttl-num-blocks number of blocks or if it's
# insertion time into the mempool is beyond ttl-duration.
ttl-duration = "0s"

# ttl-num-blocks, if non-zero, defines the maximum number of blocks a transaction
# can exist for in the mempool.
#
# Note, if ttl-duration is also defined, a transaction will be removed if it
# has existed in the mempool at least ttl-num-blocks number of blocks or if
# it's insertion time into the mempool is beyond ttl-duration.
ttl-num-blocks = 0

#######################################################
###         State Sync Configuration Options        ###
#######################################################
[statesync]
# State sync rapidly bootstraps a new node by discovering, fetching, and restoring a state machine
# snapshot from peers instead of fetching and replaying historical blocks. Requires some peers in
# the network to take and serve state machine snapshots. State sync is not attempted if the node
# has any local state (LastBlockHeight > 0). The node will have a truncated block history,
# starting from the height of the snapshot.
enable = false

# RPC servers (comma-separated) for light client verification of the synced state machine and
# retrieval of state data for node bootstrapping. Also needs a trusted height and corresponding
# header hash obtained from a trusted source, and a period during which validators can be trusted.
#
# For Cosmos SDK-based chains, trust-period should usually be about 2/3 of the unbonding time (~2
# weeks) during which they can be financially punished (slashed) for misbehavior.
rpc-servers = ""
trust-height = 0
trust-hash = ""
trust-period = "168h0m0s"

# Time to spend discovering snapshots before initiating a restore.
discovery-time = "15s"

# Temporary directory for state sync snapshot chunks, defaults to the OS tempdir (typically /tmp).
# Will create a new, randomly named directory within, and remove it when done.
temp-dir = ""

# The timeout duration before re-requesting a chunk, possibly from a different
# peer (default: 15 seconds).
chunk-request-timeout = "15s"

# The number of concurrent chunk and block fetchers to run (default: 4).
fetchers = "4"

#######################################################
###       Block Sync Configuration Connections       ###
#######################################################
[blocksync]

# If this node is many blocks behind the tip of the chain, BlockSync
# allows them to catchup quickly by downloading blocks in parallel
# and verifying their commits
enable = true

# Block Sync version to use:
#   1) "v0" (default) - the standard block sync implementation
#   2) "v2" - DEPRECATED, please use v0
version = "v0"

#######################################################
###         Consensus Configuration Options         ###
#######################################################
[consensus]

wal-file = "data/cs.wal/wal"

# How long we wait for a proposal block before prevoting nil
timeout-propose = "3s"
# How much timeout-propose increases with each round
timeout-propose-delta = "500ms"
# How long we wait after receiving +2/3 prevotes for “anything” (ie. not a single block or nil)
timeout-prevote = "1s"
# How much the timeout-prevote increases with each round
timeout-prevote-delta = "500ms"
# How long we wait after receiving +2/3 precommits for “anything” (ie. not a single block or nil)
timeout-precommit = "1s"
# How much the timeout-precommit increases with each round
timeout-precommit-delta = "500ms"
# How long we wait after committing a block, before starting on the new
# height (this gives us a chance to receive some more precommits, even
# though we already have +2/3).
timeout-commit = "1s"

# How many blocks to look back to check existence of the node's consensus votes before joining consensus
# When non-zero, the node will panic upon restart
# if the same consensus key was used to sign {double-sign-check-height} last blocks.
# So, validators should stop the state machine, wait for some blocks, and then restart the state machine to avoid panic.
double-sign-check-height = 0

# Make progress as soon as we have all the precommits (as if TimeoutCommit = 0)
skip-timeout-commit = false

# EmptyBlocks mode and possible interval between empty blocks
create-empty-blocks = true
create-empty-blocks-interval = "0s"

# Reactor sleep duration parameters
peer-gossip-sleep-duration = "100ms"
peer-query-maj23-sleep-duration = "2s"

#######################################################
###   Transaction Indexer Configuration Options     ###
#######################################################
[tx-index]

# The backend database list to back the indexer.
# If list contains "null" or "", meaning no indexer service will be used.
#
# The application will set which txs to index. In some cases a node operator will be able
# to decide which txs to index based on configuration set in the application.
#
# Options:
#   1) "null"
#   2) "kv" (default) - the simplest possible indexer, backed by key-value storage (defaults to levelDB; see DBBackend).
#   3) "psql" - the indexer services backed by PostgreSQL.
# When "kv" or "psql" is chosen "tx.height" and "tx.hash" will always be indexed.
indexer = ["kv"]

# The PostgreSQL connection configuration, the connection format:
#   postgresql://<user>:<password>@<host>:<port>/<db>?<opts>
psql-conn = ""

#######################################################
###       Instrumentation Configuration Options     ###
#######################################################
[instrumentation]

# When true, Prometheus metrics are served under /metrics on
# PrometheusListenAddr.
# Check out the documentation for the list of available metrics.
prometheus = false

# Address to listen for Prometheus collector(s) connections
prometheus-listen-addr = ":26660"

# Maximum number of simultaneous connections.
# If you want to accept a larger number than the default, make sure
# you increase your OS limits.
# 0 - unlimited.
max-open-connections = 3

# Instrumentation namespace
namespace = "tendermint"
```

## 空块 VS 没有空块

### create-empty-blocks = true

如果在您的配置中将 `create-empty-blocks` 设置为 `true`，则块将是
每秒创建~(使用默认共识参数)。你可以调节
通过更改`timeout-commit`来延迟块之间的延迟。例如。 `timeout-commit = "10s"` 应该导致大约 10 秒的块。

### create-empty-blocks = false

在此设置中，在收到交易时创建块。

注意在块 H 之后，Tendermint 创建了一个我们称之为“证明块”的东西
(仅当应用程序哈希更改时) H+1。这样做的原因是为了支持
证明。如果您在区块 H 中有一笔交易将状态更改为 X，则
新的应用程序哈希将只包含在块 H+1 中。如果在你之后
交易已提交，您想获得新状态的轻客户端证明
(X)，您需要提交新块才能做到这一点，因为新块
块具有状态 X 的新应用程序哈希值。这就是我们创建一个新应用程序的原因
(空)如果应用程序哈希更改则阻止。否则，你将无法
为新状态做一个证明。

另外，如果您将 `create-empty-blocks-interval` 设置为除
默认(`0`)，Tendermint 将创建空块，即使没有
交易每个`create-empty-blocks-interval`。例如，与
`create-empty-blocks = false` 和 `create-empty-blocks-interval = "30s"`，
Tendermint 只会在有交易或等待后创建区块
30 秒没有收到任何交易。

## 共识超时解释

在 [Running in
生产](../tendermint-core/running-in-production.md)

您还可以在规范中找到更详细的技术说明:[最新的
关于 BFT 共识的八卦](https://arxiv.org/abs/1807.04938)。

```toml
[consensus]
...

timeout-propose = "3s"
timeout-propose-delta = "500ms"
timeout-prevote = "1s"
timeout-prevote-delta = "500ms"
timeout-precommit = "1s"
timeout-precommit-delta = "500ms"
timeout-commit = "1s"
```

请注意，在成功的回合中，我们绝对等待的唯一超时没有
不管什么是“超时提交”。

以下是超时的简要摘要:

- `timeout-propose` = 在预投票之前我们等待提案块的时间
  零
- `timeout-propose-delta` = 每轮超时提议增加多少
- `timeout-prevote` = 我们在收到 +2/3 prevotes 后等待的时间
  任何东西(即不是单个块或零)
- `timeout-prevote-delta` = 每次超时预投票增加多少
  圆形的
- `timeout-precommit` = 在收到 +2/3 预提交后我们等待的时间
  任何东西(即不是单个块或零)
- `timeout-precommit-delta` = 超时预提交增加多少
  每轮
- `timeout-commit` = 提交块后等待多长时间，然后开始
  在新的高度上(这让我们有机会收到更多的预提交，
  即使我们已经有 +2/3)

## P2P 设置

本节将介绍 `config.toml` 的 p2p 部分中的设置。

- `external-address` = 是将公布给其他节点使用的地址。我们建议使用您的公共 IP 和 p2p 端口设置此字段。
  - > 我们建议设置一个外部地址。当在专用网络中使用时，Tendermint Core 目前不公布节点的公共地址。有一些积极和持续的工作来改进 P2P 系统，但目前这是一个有用的解决方法。
- `persistent-peers` = 是一个逗号分隔的对等点列表，您将始终希望连接到这些对等点。如果您已连接到最大数量的对等点，则不会添加持久对等点。
- `pex` = 打开或关闭对等交换反应器。验证器节点将希望关闭“pex”，这样它就不会开始向网络上的未知对等点闲聊。对于具有固定网络连接的静态配置网络，也可以关闭 PeX。对于开放、动态网络上的完整节点，应该打开它。
- `private-peer-ids` = 是一个逗号分隔的节点 id 列表，_不会_暴露给其他对等点(即，您不会将这个列表中的 id 告诉其他对等点)。这可以用验证器的节点 ID 填充。

最近，Tendermint 团队对 p2p 层进行了重构。这导致多个配置参数被弃用和/或替换。

我们将在下面介绍新的和已弃用的参数。
### 新参数

有三个新参数，如果 use-legacy 设置为 false，则启用这些参数。

- `queue-type` = 设置在 p2p 层使用的队列类型。有三个选项可用`fifo`、`priority`和`wdrr`。默认是优先级
- `bootstrap-peers` = 是一个逗号分隔的节点列表，用于引导地址簿。
- `max-connections` = 是允许的入站和出站连接的最大数量。
### 弃用的参数

> 注意:对于 Tendermint 0.35，有两个 p2p 实现。旧版本由 deafult 与已弃用的字段一起使用。新的实现使用不同的配置参数，如上所述。

- `max-num-inbound-peers` = 是您一次接受入站连接的最大对等点数(他们拨打您的地址并启动连接)。 *这被替换为`max-connections`*
- `max-num-outbound-peers` = 是你一次将发起出站连接的最大对等点数(你拨打他们的地址并发起连接)。*这被替换为 `max-connections`*
- `unconditional-peer-ids` = 类似于 `persistent-peers` 不同之处在于，即使您已经连接到最大数量的对等点，这些对等点也会被连接。这可以是哨兵节点上的验证器节点 ID。 *已弃用*
- `seeds` = 是一个逗号分隔的种子节点列表，您将在开始时连接并请求对等节点。种子节点是不参与共识的节点，但仅有助于将节点传播到网络中的节点 * 不推荐使用，由引导节点替换 *

## 索引设置

操作员可以通过 `[tx_index]` 部分配置索引。 `索引器`
字段采用一系列受支持的索引器。如果包含 `null`，索引将
无论提供的其他值如何，都将被关闭。

### 支持的索引器

#### KV

`kv` 索引器类型是主要支持的嵌入式键值存储
底层 Tendermint 数据库。使用 `kv` 索引器类型允许您查询
用于直接针对 Tendermint 的 RPC 的块和交易事件。但是，那
查询语法有限，因此可能会弃用或删除此索引器类型
完全在未来。

#### PostgreSQL

`psql` 索引器类型允许操作员启用块和事务事件
通过将其代理到允许事件的外部 PostgreSQL 实例进行索引
存储在关系模型中。由于事件存储在 RDBMS 中，操作员
可以利用 SQL 来执行一系列丰富而复杂的查询
`kv` 索引器类型支持。由于运算符可以直接利用 SQL，
未通过 Tendermint 的 RPC 为 `psql` 索引器类型启用搜索 - 任何
这样的查询将失败。

注意，SQL 模式存储在 `state/indexer/sink/psql/schema.sql` 和操作符中
必须在启动 Tendermint 和启用之前明确创建关系
`psql` 索引器类型。

例子:

```shell
$ psql ... -f state/indexer/sink/psql/schema.sql
```
