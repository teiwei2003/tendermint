# ADR 065:カスタムイベントインデックス

-[ADR 065:カスタムイベントインデックス](#adr-065-custom-event-indexing)
  -[変更ログ](#changelog)
  -[状態](#State)
  -[コンテキスト](#context)
  -[代替方法](#alternative-approaches)
  -[決定](#decision)
  -[詳細設計](#detailed-design)
    -[EventSink](#eventsink)
    -[サポートされているシンク](#supported-sinks)
      -[`KVEventSink`](#kveventsink)
      -[`PSQLEventSink`](#psqleventsink)
    -[構成](#configuration)
  -[将来の改善](#future-improvements)
  -[結果](#consequences)
    -[正](#positive)
    -[ネガティブ](#ネガティブ)
    -[ニュートラル](#neutral)
  -[参照](#references)

## 変更ログ

-2021年4月1日:最初のドラフト(@alexanderbez)
-2021年4月28日:KVインデクサー(@ marbar3778)を介して指定された検索機能のみをサポートします
-2021年5月19日:SQLアーキテクチャとeventsinkインターフェースを更新(@ jayt106)
-2021年8月30日:SQLアーキテクチャとpsql実装を更新(@creachadair)
-2021年10月5日:目標を明確にし、変更を実装する(@creachadair)

## ステータス

受け入れられました

## 環境

現在、Tendermint Coreは、ブロックとトランザクションイベントによるインデックス作成をサポートしています
`tx_index.indexer`構成.イベントはトランザクションでキャプチャされ、
「TxIndexer」タイプによるインデックス作成.イベントは、特にブロックにキャプチャされます
「BeginBlock」および「EndBlock」アプリケーションからの応答、および合格
`BlockIndexer`タイプ.どちらのタイプも単一の「IndexerService」によって管理されます
イベントを消費して送信する責任があります
対応するタイプによるインデックス.

インデックスに加えて、TendermintCoreはクエリもサポートしています
TendermintのRPCレイヤーを介してトランザクションのインデックスを作成し、イベントをブロックします.能力
これらのインデックスイベントをクエリすると、多数のアップストリームクライアントに役立ちます
また、ブロックエクスプローラー、IBCリピーター、補助などのアプリケーション機能
データの可用性とインデックス作成サービス.

現在、Tendermintは、サポートされている `kv`インデクサーを介したインデックス作成のみをサポートしています.
基になる埋め込みキー/値を介してデータベースを保存します. `kv`インデクサーの実装
独自のインデックスとクエリメカニズム.前者は少し些細なことですが、
リッチで柔軟なクエリレイヤーを提供することは簡単ではなく、多くの原因となっています
アップストリームのクライアントとアプリケーションの問題、およびユーザーエクスペリエンスの問題.

独自の「kv」​​クエリエンジンの脆弱性と可能性
多数の消費者が使用する場合のパフォーマンスとスケーリングの問題
より堅牢で柔軟なインデックスとクエリの需要を刺激するために導入されました
解決.
## 代替方法

より強力なソリューションの代替案に関しては、唯一の深刻な
検討中の競合他社は、[SQLite](https://www.sqlite.org/index.html)の使用に移行しています.

この方法は実行可能ですが、特定のクエリ言語に固定されます
ストレージレイヤーなので、いくつかの点で、現在の方法よりも少しだけ優れています.
さらに、実装にはCGOの導入が必要になります
Tendermintコアスタック、そして現在CGOは
使用されたデータベース.

## 決定

CosmosSDKの `KVStore`状態と同様の方法を採用します
[ADR-038](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-038-state-listening.md)で説明されているリスニング.

次の変更を実装します.

-新しいインターフェース `EventSink`を導入します.すべてのデータシンクはこのインターフェースを実装する必要があります.
-既存の `tx_index.indexer`構成を追加し、一連の設定を受け入れるようになりました
  1つ以上のタイプのインデクサー、つまりレシーバー.
-現在の `TxIndexer`と` BlockIndexer`を `KVEventSink`に結合します
  `EventSink`インターフェースを実装しました.
-追加の `EventSink`実装を導入します.これはによって実装されます
  [PostgreSQL](https://www.postgresql.org/).
  -ブロックおよびトランザクションイベントのインデックス作成をサポートするために必要なパターンを実装します.
-一連の `EventSinks`を使用するように` IndexerService`を更新します.

また:

-Postgresインデクサーの実装は独自の `kv`を実装しません
  クエリ言語. Postgresインデクサーに対してクエリを作成したいユーザー
  基盤となるDBMSに直接接続し、に基づいて使用します
  インデックスモード.

  カスタムインデクサーの将来の実装はサポートを必要としません
  独自のクエリ言語.

-現在、既存の `kv`インデクサーは現在の位置のままになります
  クエリはサポートされていますが、以降のバージョンでは非推奨としてマークされ、
  ドキュメントは、問い合わせが必要なユーザーを奨励するために更新されます
  Postgresインデクサーに移行されるイベントインデックス.

-将来的には、 `kv`インデクサーを完全に削除するか、次のように置き換える可能性があります
  異なる認識;決定は将来の作業のために延期されました.

-将来的には、RPCサービスからインデックスクエリエンドポイントを削除する可能性があります
  完全に;決定は将来の作業のために延期されましたが、推奨されました.

## 詳細設計

###イベントレシーバー

サポートされているすべてのレシーバーが実装する必要のある `EventSink`インターフェースタイプを導入しました.
インターフェイスは次のように定義されています.

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

`IndexerService`は、` EventSink`タイプの1つ以上のリストを受け入れます.真ん中
`OnStart`メソッドは、各` EventSink`で適切なAPIを呼び出します.
インデックスブロックとトランザクションイベント.

### サポートされているレシーバー

最初は、2種類の「EventSink」をすぐにサポートします.

#### `KVEventSink`

このタイプの `EventSink`は、` TxIndexer`と `BlockIndexer`の組み合わせです.
インデクサーは、どちらも単一の組み込みキー/値データベースによってサポートされています.

既存のビジネスロジックのほとんどは同じままですが、既存のAPI
新しい `EventSink`APIにマッピングされます.単一をサポートするために両方のタイプが削除されます
`KVEventSink`タイプ.

`KVEventSink`はデフォルトで有効になっている唯一の` EventSink`になるので、UXから
観点から、オペレーターは構成外の違いに気付かないようにする必要があります
変化する.

かなり単純なはずなので、EventSinkの実装の詳細は省略しました
既存のビジネスロジックを新しいAPIにマッピングします.

#### `PSQLEventSink`

このタイプの `EventSink`は、ブロックイベントとトランザクションイベントを[PostgreSQL](https://www.postgresql.org/)にインデックス付けします.
データベース.次のアーキテクチャを定義して自動的に移行する場合
`IndexerService`が起動します.

postgresイベントレシーバーは、 `tx_search`、` block_search`、 `GetTxByHash`、および` HasBlock`をサポートしません.

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

### 構成

現在の `tx_index.indexer`構成は、リストを受け入れるように変更されます
サポートされている `EventSink`タイプは単一の値ではありません.

例:

```toml
[tx_index]

indexer = [
  "kv",
  "psql"
]
```

`インデクサー`リストに `null`インデクサーが含まれている場合、インデクサーは使用されません
他にどのような値が存在する可能性があるかに関係なく.

イベントによっては、他の構成パラメーターが必要になる場合があります
シンクは `tx_index.indexer`に提供されます. `psql`には追加が必要になります
接続構成.

```toml
[tx_index]

indexer = [
  "kv",
  "psql"
]

pqsql_conn = "postgresql://<user>:<password>@<host>:<port>/<db>?<opts>"
```

無効または誤って構成された `tx_index`構成は、エラーを生成する必要があります
できるだけ早く.

## 将来の改善

技術的には、現在の機能と同じである必要はありませんが
既存のテンダーミントインデクサー、オペレーターが有益になる方法があります
「インデックスの再作成」を実行します.具体的には、テンダーミントのオペレーターは
Tendermintノードがすべてのブロックのインデックスの再作成を実行できるようにするRPCメソッド
そして、2つの指定された高さH <sub> 1 </ sub>とH <sub> 2 </ sub>の間のトランザクションイベント、
ブロックストレージにすべてのブロックとトランザクション結果が含まれている限り
指定された範囲内の指定された高さ.

## 結果

### ポジティブ

-インデックス作成と検索のためのより強力で柔軟なインデックス作成とクエリエンジン
  ブロックおよびトランザクションイベント.
-カスタムインデックスおよびクエリエンジン機能をサポートする必要はありません
  従来の `kv`タイプ.
-インデックスをアンロード/プロキシし、基になるレシーバーにクエリを実行する機能.
-スケーラビリティと信頼性は基本的に最下層から「無料」です
  サポートされている場合はシンクします.

### ネガティブ

-複数の、場合によっては成長するカスタム `EventSink`をサポートする必要があります
  タイプ.

### ニュートラル

## 参照する

-[Cosmos SDK ADR-038](https://github.com/cosmos/cosmos-sdk/blob/master/docs/architecture/adr-038-state-listening.md)
-[PostgreSQL](https://www.postgresql.org/)
-[SQLite](https://www.sqlite.org/index.html)
