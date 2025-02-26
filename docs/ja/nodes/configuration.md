# 構成

TendermintCoreはTOMLファイルを介して構成できます
`$ TMHOME/config/config.toml`. これらのパラメータの一部は上書きできます
コマンドラインフラグ. ほとんどのユーザーの場合、 `#####主な基本構成オプション#####` ##### `のオプションは、オプションを構成するときに変更することを目的としています.
以下は、上級ユーザー向けです.

## オプション

`tendermintinit`によって作成されたデフォルトの設定ファイルにはすべてが含まれています
パラメータはデフォルト値に設定されています. それは何かに見えます
以下のファイルと同じですが、検査を通じて
`config.toml`は、インストールした` tendermin`のバージョンで作成されます.

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

##空のブロックVS空のブロックなし

### create-empty-blocks = true

設定で `create-empty-blocks`が` true`に設定されている場合、ブロックは
〜毎秒作成(デフォルトのコンセンサスパラメータを使用).あなたは規制することができます
`timeout-commit`を変更することによるブロック間の遅延.例えば. `timeout-commit =" 10s "`は、約10秒のブロックになります.

### create-empty-blocks = false

この設定では、トランザクションを受信したときにブロックが作成されます.

ブロックHの後に、テンダーミントは「プルーフブロック」と呼ばれるものを作成することに注意してください
(アプリケーションハッシュが変更された場合のみ)H +1.これの理由はサポートすることです
証明.ブロックHに、状態をXに変更するトランザクションがある場合、
新しいアプリケーションハッシュは、ブロックH +1にのみ含まれます.あなたの後なら
トランザクションがコミットされました.新しい状態のライトクライアントプルーフを取得する必要があります
(X)、それを行うには、新しいブロックをコミットする必要があります.
ブロックには、状態Xの新しいアプリケーションハッシュがあります.そのため、新しいアプリケーションハッシュを作成します.
(空)アプリケーションハッシュが変更された場合はブロックします.そうでなければ、あなたはすることができません
新しい状態の証拠を作成します.

さらに、 `create-empty-blocks-interval`を
デフォルト( `0`)、Tendermintは、存在しない場合でも空のブロックを作成します
`create-empty-blocks-interval`ごとにトランザクションします.たとえば、
`create-empty-blocks = false`および` create-empty-blocks-interval = "30s" `、
Tendermintは、トランザクションがある場合、または待機した後にのみブロックを作成します
トランザクションを受信せずに30秒.

##コンセンサスタイムアウトの説明

[Running in]には、タイムアウトに関するさまざまな情報があります.
プロダクション](../tendermint-core/running-in-production.md)

また、仕様でより詳細な技術的説明を見つけることができます:[最新
BFTコンセンサスに関するゴシップ](https://arxiv.org/abs/1807.04938).
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

成功したラウンドでは、私たちが絶対に待つ唯一のタイムアウトはそうではないことに注意してください
「提出のタイムアウト」が何であっても.

タイムアウトの概要は次のとおりです.

-`timeout-propose` =事前投票の前に提案ブロックを待つ時間
  零
-`timeout-propose-delta` =各タイムアウトプロポーザルをどれだけ増やすか
-`timeout-prevote` = + 2/3のprevoteを受け取ってから待機した時間
  何でも(つまり、単一のブロックまたはゼロではない)
-`timeout-prevote-delta` =タイムアウトごとにprevoteはどのくらい増加しますか
  円形
-`timeout-precommit` = + 2/3precommitを受信して​​から待機する時間
  何でも(つまり、単一のブロックまたはゼロではない)
-`timeout-precommit-delta` =タイムアウトのprecommitの増加量
  毎ラウンド
-`timeout-commit` =ブロックをコミットしてから開始するまでの時間
  新しい高さで(これにより、より多くの事前コミットを受け取る機会が得られます.
  すでに+2/3を持っていても)

## P2P設定

このセクションでは、 `config.toml`のp2pセクションの設定を紹介します.

-`external-address` =は他のノードにアナウンスされるアドレスです.このフィールドを設定するには、パブリックIPとp2pポートを使用することをお勧めします.
  ->外部アドレスを設定することをお勧めします.プライベートネットワークで使用される場合、TendermintCoreは現在ノードのパブリックアドレスをアナウンスしません. P2Pシステムを改善するための積極的かつ継続的な作業がいくつかありますが、これは現在有用なソリューションです.
-`persistent-peers` =はコンマで区切られたピアのリストであり、常にこれらのピアに接続する必要があります.最大数のピアに接続している場合、永続的なピアは追加されません.
-`pex` =ピアツーピア交換リアクターをオンまたはオフにします.バリデーターノードは、ネットワーク上の不明なピアとのチャットを開始しないように、「pex」を閉じたいと思うでしょう.固定ネットワーク接続で静的に構成されたネットワークの場合、PeXをオフにすることもできます.オープンで動的なネットワーク上のフルノードの場合は、オンにする必要があります.
-`private-peer-ids` =は、ノードIDのコンマ区切りのリストであり、他のピアには公開されません(つまり、このリストのIDを他のピアに通知しません).これには、バリデーターのノードIDを入力できます.

最近、Tendermintチームはp2pレイヤーをリファクタリングしました.これにより、複数の構成パラメーターが非推奨または置き換えられました.

以下に、新しいパラメータと廃止されたパラメータについて説明します.
###新しいパラメータ

use-legacyがfalseに設定されている場合に有効になる、3つの新しいパラメーターがあります.

-`queue-type` = p2pレイヤーで使用されるキュータイプを設定します. 「fifo」、「priority」、「wdrr」の3つのオプションを使用できます.デフォルトは優先度です
-`bootstrap-peers` =は、アドレスブックをブートストラップするために使用されるノードのコンマ区切りのリストです.
-`max-connections` =は、許可されるインバウンド接続とアウトバウンド接続の最大数です.
###非推奨のパラメータ

>注:Tendermint 0.35の場合、2つのp2p実装があります.古いバージョンは、非推奨のフィールドを持つdeafultによって使用されていました.新しい実装では、上記のように、さまざまな構成パラメーターを使用します.

-`max-num-inbound-peers` =一度にインバウンド接続を受け入れることができるピアの最大数です(アドレスをダイヤルして接続を開始します). *これは `max-connections`に置き換えられます*
-`max-num-outbound-peers` =一度にアウトバウンド接続を開始するピアの最大数です(アドレスをダイヤルして接続を開始します). *これは `max-connections`に置き換えられます*
-`unconditional-peer-ids` = `persistent-peers`に似ています違いは、最大数のピアに接続している場合でも、これらのピアは接続されるということです.これは、センチネルノードのバリデーターノードIDにすることができます. *非推奨*
-`seeds` =は、シードノードのコンマ区切りのリストです.最初に接続してピアノードを要求します.シードノードは、コンセンサスに参加しないノードですが、ノードをネットワーク内のノードに拡散するのに役立つだけです*推奨されません.ガイドノードに置き換えられます*

##インデックス設定

オペレーターは、 `[tx_index]`セクションを介してインデックスを構成できます. `インデクサ`
このフィールドは、サポートされている一連のインデクサーを使用します. `null`が含まれている場合、インデックスは
提供された他の値に関係なく、それは閉じられます.

###サポートされているインデクサー

#### KV

`kv`インデクサータイプは、サポートされている主要な組み込みKey-Valueストアです
基盤となるTendermintデータベース. `kv`インデクサータイプを使用すると、クエリを実行できます
TendermintのRPCブロックおよびトランザクションイベントを直接ターゲットにするために使用されます.でもそれは
クエリ構文が制限されているため、このインデクサータイプは非推奨または削除される可能性があります
完全に将来.

#### PostgreSQL

`psql`インデクサータイプにより、オペレーターはブロックイベントとトランザクションイベントを有効にできます
イベントを許可する外部PostgreSQLインスタンスにプロキシしてインデックスを作成します
リレーショナルモデルに格納されます.イベントはRDBMSに保存されるため、オペレーターは
SQLを使用して、一連のリッチで複雑なクエリを実行できます
`kv`インデクサータイプのサポート.演算子はSQLを直接使用できるため、
TendermintのRPCが `psql`インデクサータイプの検索を有効にできませんでした-任意
このようなクエリは失敗します.

SQLスキーマは `state/indexer/sink/psql/schema.sql`と演算子に格納されていることに注意してください
Tendermintを起動して有効にする前に、関係を明示的に作成する必要があります
`psql`インデクサータイプ.

例:

```shell
$ psql ... -f state/indexer/sink/psql/schema.sql
```
