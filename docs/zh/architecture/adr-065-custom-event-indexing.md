# ADR 065:自定义事件索引

- [ADR 065:自定义事件索引](#adr-065-custom-event-indexing)
  - [更新日志](#changelog)
  - [状态](#状态)
  - [上下文](#context)
  - [替代方法](#alternative-approaches)
  - [决定](#decision)
  - [详细设计](#detailed-design)
    - [EventSink](#eventsink)
    - [支持的接收器](#supported-sinks)
      - [`KVEventSink`](#kveventsink)
      - [`PSQLEventSink`](#psqleventsink)
    - [配置](#configuration)
  - [未来改进](#future-improvements)
  - [后果](#consequences)
    - [正](#positive)
    - [负](#负)
    - [中性](#中性)
  - [参考文献](#references)

## 变更日志

- 2021 年 4 月 1 日:初稿 (@alexanderbez)
- 2021 年 4 月 28 日:仅通过 KV 索引器支持指定搜索功能 (@marbar3778)
- 2021 年 5 月 19 日:更新 SQL 架构和 eventsink 接口 (@jayt106)
- 2021 年 8 月 30 日:更新 SQL 架构和 psql 实现 (@creachadair)
- 2021 年 10 月 5 日:阐明目标和实施变更 (@creachadair)

## 状态

公认

## 语境

目前，Tendermint Core 支持通过区块和交易事件索引
`tx_index.indexer` 配置.事件在事务中被捕获，并且
通过“TxIndexer”类型进行索引.事件是在块中捕获的，特别是
来自“BeginBlock”和“EndBlock”应用程序响应，并通过
`BlockIndexer` 类型.这两种类型都由单个“IndexerService”管理
它负责消费事件并将这些事件发送出去
按相应类型索引.

除了索引之外，Tendermint Core 还支持查询
通过 Tendermint 的 RPC 层索引交易和区块事件.能力，技能
查询这些索引事件有助于大量上游客户端
和应用能力，例如区块浏览器、IBC 中继器和辅助
数据可用性和索引服务.

目前，Tendermint 仅支持通过 `kv` 索引器进行索引，该索引器受支持
通过底层嵌入式键/值存储数据库. `kv` 索引器实现
它自己的索引和查询机制.虽然前者有些微不足道，
提供丰富而灵活的查询层并非易事，并且已经引起了许多
上游客户端和应用程序的问题和用户体验问题.

专有的“kv”查询引擎的脆弱性和潜力
当大量消费者使用时出现的性能和扩展问题
引入，激发对更健壮和灵活的索引和查询的需求
解决方案.
## 替代方法

关于更强大的解决方案的替代方法，唯一严重的
被考虑的竞争者是过渡到使用 [SQLite](https://www.sqlite.org/index.html).

虽然该方法可行，但它会将我们锁定在特定的查询语言中
存储层，所以在某些方面它只比我们目前的方法好一点.
此外，实施将需要将 CGO 引入
Tendermint 核心堆栈，而现在 CGO 仅根据
使用的数据库.

## 决定

我们将采用与 Cosmos SDK 的 `KVStore` 状态类似的方法
[ADR-038](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-038-state-listening.md) 中描述的聆听.

我们将实施以下更改:

- 引入一个新接口`EventSink`，所有数据接收器都必须实现该接口.
- 增加现有的`tx_index.indexer` 配置，现在接受一个系列
  一种或多种索引器类型，即接收器.
- 将当前的`TxIndexer` 和`BlockIndexer` 组合成一个`KVEventSink`
  实现了`EventSink`接口.
- 引入一个额外的`EventSink` 实现，该实现由
  [PostgreSQL](https://www.postgresql.org/).
  - 实施必要的模式以支持块和交易事件索引.
- 更新 `IndexerService` 以使用一系列 `EventSinks`.

此外:

- Postgres 索引器实现将_不_实现专有的`kv`
  查询语言.希望针对 Postgres 索引器编写查询的用户
  将直接连接到底层 DBMS 并使用基于
  索引模式.

  未来的自定义索引器实现将不需要支持
  专有查询语言.

- 目前，现有的 `kv` 索引器将保留在其当前位置
  查询支持，但将在后续版本中标记为已弃用，并且
  文档将更新以鼓励需要查询的用户
  要迁移到 Postgres 索引器的事件索引.

- 将来我们可能会完全移除 `kv` 索引器，或者将其替换为
  不同的实现；该决定被推迟为今后的工作.

- 将来，我们可能会从 RPC 服务中删除索引查询端点
  完全;该决定被推迟为未来的工作，但建议.


## 详细设计

### 事件接收器

我们介绍了所有支持的接收器必须实现的 `EventSink` 接口类型.
接口定义如下:

```go
type EventSink interface {
  IndexBlockEvents(types.EventDataNewBlockHeader) error
  IndexTxEvents([]*abci.TxResult) error

  SearchBlockEvents(context.Context, *query.Query) ([]int64, error)
  SearchTxEvents(context.Context, *query.Query) ([]*abci.TxResult, error)

  GetTxByHash([]byte) (*abci.TxResult, error)
  HasBlock(int64) (bool, error)

  Type() EventSinkType
  Stop() error
}
```

`IndexerService` 将接受一个或多个 `EventSink` 类型的列表.中
`OnStart` 方法它将在每个 `EventSink` 上调用适当的 API 以
索引块和交易事件.

### 支持的接收器

我们最初将支持两种开箱即用的“EventSink”类型.

####`KVEventSink`

这种类型的`EventSink`是`TxIndexer`和`BlockIndexer`的组合
索引器，两者都由单个嵌入式键/值数据库支持.

大部分现有业务逻辑将保持不变，但现有 API
映射到新的`EventSink` API.两种类型都将被删除以支持单一
`KVEventSink` 类型.

`KVEventSink` 将是唯一默认启用的 `EventSink`，因此从 UX
从角度来看，操作员不应注意到配置之外的差异
改变.

我们省略了 EventSink 实现细节，因为它应该相当简单
将现有业务逻辑映射到新 API.

####`PSQLEventSink`

这种类型的 `EventSink` 将区块和交易事件索引到 [PostgreSQL](https://www.postgresql.org/) 中.
数据库.我们定义并自动迁移以下架构时
`IndexerService` 启动.

postgres 事件接收器将不支持 `tx_search`、`block_search`、`GetTxByHash` 和 `HasBlock`.

```sql
-- Table Definition ----------------------------------------------

-- The blocks table records metadata about each block.
-- The block record does not include its events or transactions (see tx_results).
CREATE TABLE blocks (
  rowid      BIGSERIAL PRIMARY KEY,

  height     BIGINT NOT NULL,
  chain_id   VARCHAR NOT NULL,

  -- When this block header was logged into the sink, in UTC.
  created_at TIMESTAMPTZ NOT NULL,

  UNIQUE (height, chain_id)
);

-- Index blocks by height and chain, since we need to resolve block IDs when
-- indexing transaction records and transaction events.
CREATE INDEX idx_blocks_height_chain ON blocks(height, chain_id);

-- The tx_results table records metadata about transaction results.  Note that
-- the events from a transaction are stored separately.
CREATE TABLE tx_results (
  rowid BIGSERIAL PRIMARY KEY,

  -- The block to which this transaction belongs.
  block_id BIGINT NOT NULL REFERENCES blocks(rowid),
  -- The sequential index of the transaction within the block.
  index INTEGER NOT NULL,
  -- When this result record was logged into the sink, in UTC.
  created_at TIMESTAMPTZ NOT NULL,
  -- The hex-encoded hash of the transaction.
  tx_hash VARCHAR NOT NULL,
  -- The protobuf wire encoding of the TxResult message.
  tx_result BYTEA NOT NULL,

  UNIQUE (block_id, index)
);

-- The events table records events. All events (both block and transaction) are
-- associated with a block ID; transaction events also have a transaction ID.
CREATE TABLE events (
  rowid BIGSERIAL PRIMARY KEY,

  -- The block and transaction this event belongs to.
  -- If tx_id is NULL, this is a block event.
  block_id BIGINT NOT NULL REFERENCES blocks(rowid),
  tx_id    BIGINT NULL REFERENCES tx_results(rowid),

  -- The application-defined type label for the event.
  type VARCHAR NOT NULL
);

-- The attributes table records event attributes.
CREATE TABLE attributes (
   event_id      BIGINT NOT NULL REFERENCES events(rowid),
   key           VARCHAR NOT NULL, -- bare key
   composite_key VARCHAR NOT NULL, -- composed type.key
   value         VARCHAR NULL,

   UNIQUE (event_id, key)
);

-- A joined view of events and their attributes. Events that do not have any
-- attributes are represented as a single row with empty key and value fields.
CREATE VIEW event_attributes AS
  SELECT block_id, tx_id, type, key, composite_key, value
  FROM events LEFT JOIN attributes ON (events.rowid = attributes.event_id);

-- A joined view of all block events (those having tx_id NULL).
CREATE VIEW block_events AS
  SELECT blocks.rowid as block_id, height, chain_id, type, key, composite_key, value
  FROM blocks JOIN event_attributes ON (blocks.rowid = event_attributes.block_id)
  WHERE event_attributes.tx_id IS NULL;

-- A joined view of all transaction events.
CREATE VIEW tx_events AS
  SELECT height, index, chain_id, type, key, composite_key, value, tx_results.created_at
  FROM blocks JOIN tx_results ON (blocks.rowid = tx_results.block_id)
  JOIN event_attributes ON (tx_results.rowid = event_attributes.tx_id)
  WHERE event_attributes.tx_id IS NOT NULL;
```

`PSQLEventSink` 将实现 `EventSink` 接口如下
(为简洁起见省略了一些细节):

```go
func NewEventSink(connStr, chainID string) (*EventSink, error) {
	db, err := sql.Open(driverName, connStr)
	// ...

	return &EventSink{
		store:   db,
		chainID: chainID,
	}, nil
}

func (es *EventSink) IndexBlockEvents(h types.EventDataNewBlockHeader) error {
	ts := time.Now().UTC()

	return runInTransaction(es.store, func(tx *sql.Tx) error {
		// Add the block to the blocks table and report back its row ID for use
		// in indexing the events for the block.
		blockID, err := queryWithID(tx, `
INSERT INTO blocks (height, chain_id, created_at)
  VALUES ($1, $2, $3)
  ON CONFLICT DO NOTHING
  RETURNING rowid;
`, h.Header.Height, es.chainID, ts)
		// ...

		// Insert the special block meta-event for height.
		if err := insertEvents(tx, blockID, 0, []abci.Event{
			makeIndexedEvent(types.BlockHeightKey, fmt.Sprint(h.Header.Height)),
		}); err != nil {
			return fmt.Errorf("block meta-events: %w", err)
		}
		// Insert all the block events. Order is important here,
		if err := insertEvents(tx, blockID, 0, h.ResultBeginBlock.Events); err != nil {
			return fmt.Errorf("begin-block events: %w", err)
		}
		if err := insertEvents(tx, blockID, 0, h.ResultEndBlock.Events); err != nil {
			return fmt.Errorf("end-block events: %w", err)
		}
		return nil
	})
}

func (es *EventSink) IndexTxEvents(txrs []*abci.TxResult) error {
	ts := time.Now().UTC()

	for _, txr := range txrs {
		// Encode the result message in protobuf wire format for indexing.
		resultData, err := proto.Marshal(txr)
		// ...

		// Index the hash of the underlying transaction as a hex string.
		txHash := fmt.Sprintf("%X", types.Tx(txr.Tx).Hash())

		if err := runInTransaction(es.store, func(tx *sql.Tx) error {
			// Find the block associated with this transaction.
			blockID, err := queryWithID(tx, `
SELECT rowid FROM blocks WHERE height = $1 AND chain_id = $2;
`, txr.Height, es.chainID)
			// ...

			// Insert a record for this tx_result and capture its ID for indexing events.
			txID, err := queryWithID(tx, `
INSERT INTO tx_results (block_id, index, created_at, tx_hash, tx_result)
  VALUES ($1, $2, $3, $4, $5)
  ON CONFLICT DO NOTHING
  RETURNING rowid;
`, blockID, txr.Index, ts, txHash, resultData)
			// ...

			// Insert the special transaction meta-events for hash and height.
			if err := insertEvents(tx, blockID, txID, []abci.Event{
				makeIndexedEvent(types.TxHashKey, txHash),
				makeIndexedEvent(types.TxHeightKey, fmt.Sprint(txr.Height)),
			}); err != nil {
				return fmt.Errorf("indexing transaction meta-events: %w", err)
			}
			// Index any events packaged with the transaction.
			if err := insertEvents(tx, blockID, txID, txr.Result.Events); err != nil {
				return fmt.Errorf("indexing transaction events: %w", err)
			}
			return nil

		}); err != nil {
			return err
		}
	}
	return nil
}

// SearchBlockEvents is not implemented by this sink, and reports an error for all queries.
func (es *EventSink) SearchBlockEvents(ctx context.Context, q *query.Query) ([]int64, error)

// SearchTxEvents is not implemented by this sink, and reports an error for all queries.
func (es *EventSink) SearchTxEvents(ctx context.Context, q *query.Query) ([]*abci.TxResult, error)

// GetTxByHash is not implemented by this sink, and reports an error for all queries.
func (es *EventSink) GetTxByHash(hash []byte) (*abci.TxResult, error)

// HasBlock is not implemented by this sink, and reports an error for all queries.
func (es *EventSink) HasBlock(h int64) (bool, error)
```

### 配置

当前的 `tx_index.indexer` 配置将更改为接受列表
支持的`EventSink` 类型而不是单个值.

例子:

```toml
[tx_index]

indexer = [
  "kv",
  "psql"
]
```

如果 `indexer` 列表包含 `null` 索引器，则不会使用任何索引器
不管可能存在什么其他值.

根据事件的不同，可能需要其他配置参数
sinks 被提供给 `tx_index.indexer`. `psql` 将需要一个额外的
连接配置.

```toml
[tx_index]

indexer = [
  "kv",
  "psql"
]

pqsql_conn = "postgresql://<user>:<password>@<host>:<port>/<db>?<opts>"
```

任何无效或错误配置的 `tx_index` 配置都应该产生一个错误
尽早.

## 未来的改进

虽然在技术上不需要保持与当前的功能相同
现有的 Tendermint 索引器，对于运营商来说有一个方法是有益的
执行“重新索引”.具体来说，Tendermint 运营商可以调用
RPC 方法，允许 Tendermint 节点执行所有块的重新索引
以及两个给定高度 H<sub>1</sub> 和 H<sub>2</sub> 之间的交易事件，
只要块存储包含所有的块和交易结果
在给定范围内指定的高度.

## 结果

### 积极的

- 用于索引和搜索的更强大和灵活的索引和查询引擎
  块和交易事件.
- 不必支持自定义索引和查询引擎的能力
  传统的 `kv` 类型.
- 卸载/代理索引和查询到底层接收器的能力.
- 可扩展性和可靠性基本上从底层“免费”而来
  下沉，如果它支持它.

### 消极的

- 需要支持多个且可能不断增长的自定义`EventSink`
  类型.

### 中性的

## 参考

- [Cosmos SDK ADR-038](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-038-state-listening.md)
- [PostgreSQL](https://www.postgresql.org/)
- [SQLite](https://www.sqlite.org/index.html)
