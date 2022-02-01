# 索引交易

Tendermint 允许您对交易和区块进行索引，然后进行查询或
订阅他们的结果. 交易由“TxResult.Events”和
块由“Response(Begin|End)Block.Events”索引. 然而，交易
也由主键索引，其中包括交易哈希和映射
到并存储相应的“TxResult”. 块由主键索引
其中包括块高度并映射到并存储块高度，即
块本身永远不会被存储.

每个事件都包含一个类型和一个属性列表，它们是键值对
表示方法执行期间发生的事情. 更多
有关“事件”的详细信息，请参阅
[ABCI](https://github.com/tendermint/spec/blob/master/spec/abci/abci.md#events)
文档.

`Event` 有一个与之关联的复合键. “复合密钥”是
由其类型和由点分隔的键构成.

例如:

```json
"jack": [
  "account.number": 100
]
```

将等于 `jack.account.number` 的组合键.

默认情况下，Tendermint 将通过各自的哈希索引所有交易
和高度和块的高度.

## 配置

操作员可以通过 `[tx_index]` 部分配置索引. `索引器`
字段采用一系列受支持的索引器. 如果包含 `null`，索引将
无论提供的其他值如何，都将被关闭.

```toml
[tx-index]

# The backend database list to back the indexer.
# If list contains null, meaning no indexer service will be used.
#
# The application will set which txs to index. In some cases a node operator will be able
# to decide which txs to index based on configuration set in the application.
#
# Options:
#   1) "null"
#   2) "kv" (default) - the simplest possible indexer, backed by key-value storage (defaults to levelDB; see DBBackend).
#     - When "kv" is chosen "tx.height" and "tx.hash" will always be indexed.
#   3) "psql" - the indexer services backed by PostgreSQL.
# indexer = []
```

### 支持的索引器

#### KV

`kv` 索引器类型是主要支持的嵌入式键值存储
底层 Tendermint 数据库.使用 `kv` 索引器类型允许您查询
用于直接针对 Tendermint 的 RPC 的块和交易事件.但是，那
查询语法有限，因此可能会弃用或删除此索引器类型
完全在未来.

#### PostgreSQL

`psql` 索引器类型允许操作员启用块和事务事件
通过将其代理到允许事件的外部 PostgreSQL 实例进行索引
存储在关系模型中.由于事件存储在 RDBMS 中，操作员
可以利用 SQL 来执行一系列丰富而复杂的查询
`kv` 索引器类型支持.由于运算符可以直接利用 SQL，
未通过 Tendermint 的 RPC 为 `psql` 索引器类型启用搜索 - 任何
这样的查询将失败.

注意，SQL 模式存储在 `state/indexer/sink/psql/schema.sql` 和操作符中
必须在启动 Tendermint 和启用之前明确创建关系
`psql` 索引器类型.

例子:

```shell
$ psql ... -f state/indexer/sink/psql/schema.sql
```

## 默认索引

Tendermint 交易和区块事件索引器索引一些选择的保留事件
默认情况下.

### 交易

默认情况下对以下索引进行索引:

- `tx.height`
- `tx.hash`

### 块

默认情况下对以下索引进行索引:

- `block.height`

## 添加事件

应用程序可以自由定义要索引的事件. Tendermint 没有
公开功能以定义要索引哪些事件以及要忽略哪些事件. 在
您的应用程序的 `DeliverTx` 方法，添加带有成对的 `Events` 字段
UTF-8 编码的字符串(例如“transfer.sender”:“Bob”、“transfer.recipient”:
"Alice", "transfer.balance": "100").

例子:

```go
func (app *KVStoreApplication) DeliverTx(req types.RequestDeliverTx) types.Result {
    //...
    events := []abci.Event{
        {
            Type: "transfer",
            Attributes: []abci.EventAttribute{
                {Key: []byte("sender"), Value: []byte("Bob"), Index: true},
                {Key: []byte("recipient"), Value: []byte("Alice"), Index: true},
                {Key: []byte("balance"), Value: []byte("100"), Index: true},
                {Key: []byte("note"), Value: []byte("nothing"), Index: true},
            },
        },
    }
    return types.ResponseDeliverTx{Code: code.CodeTypeOK, Events: events}
}
```

如果索引器不是 `null`，事务将被索引. 每个事件都是
使用“{eventType}.{eventAttribute}={eventValue}”形式的复合键索引，
例如 `transfer.sender=bob`.

## 查询交易事件

您可以通过调用它们的事件来查询分页的一组事务
`/tx_search` RPC 端点:

```bash
curl "localhost:26657/tx_search?query=\"message.sender='cosmos1...'\"&prove=true"
```

查看 [API 文档](https://docs.tendermint.com/master/rpc/#/Info/tx_search)
有关查询语法和其他选项的更多信息.

## 订阅交易

客户端可以通过 WebSocket 订阅具有给定标签的交易
对`/subscribe` RPC 端点的查询.

```json
{
  "jsonrpc": "2.0",
  "method": "subscribe",
  "id": "0",
  "params": {
    "query": "message.sender='cosmos1...'"
  }
}
```

查看 [API 文档](https://docs.tendermint.com/master/rpc/#subscribe) 了解更多信息
关于查询语法和其他选项.

## 查询块事件

您可以通过调用它们的事件来查询一组分页的块
`/block_search` RPC 端点:

```bash
curl "localhost:26657/block_search?query=\"block.height > 10 AND val_set.num_changed > 0\""
```

查看 [API 文档](https://docs.tendermint.com/master/rpc/#/Info/block_search)
有关查询语法和其他选项的更多信息.
