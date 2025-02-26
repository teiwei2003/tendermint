# インジケーター

Tendermintは、Prometheusメトリックをレポートおよび提供できます.これにより、Prometheusメトリックが可能になります.
プロメテウスコレクターによって消費されます.

この機能はデフォルトで無効になっています.

Prometheusメトリックを有効にするには、
構成ファイル. デフォルトでは、メトリックはポート26660の `/ metrics`の下に提供されます.
リスニングアドレスは、構成ファイルで変更できます(を参照).
`instrumentation.prometheus \ _listen \ _addr`).

## 利用可能なインジケーターのリスト

次のインジケーターが利用可能です.

| **Name**                               | **Type**  | **Tags**      | **Description**                                                        |
| -------------------------------------- | --------- | ------------- | ---------------------------------------------------------------------- |
| abci_connection_method_timing          | Histogram | method, type  | Timings for each of the ABCI methods                                   |
| consensus_height                       | Gauge     |               | Height of the chain                                                    |
| consensus_validators                   | Gauge     |               | Number of validators                                                   |
| consensus_validators_power             | Gauge     |               | Total voting power of all validators                                   |
| consensus_validator_power              | Gauge     |               | Voting power of the node if in the validator set                       |
| consensus_validator_last_signed_height | Gauge     |               | Last height the node signed a block, if the node is a validator        |
| consensus_validator_missed_blocks      | Gauge     |               | Total amount of blocks missed for the node, if the node is a validator |
| consensus_missing_validators           | Gauge     |               | Number of validators who did not sign                                  |
| consensus_missing_validators_power     | Gauge     |               | Total voting power of the missing validators                           |
| consensus_byzantine_validators         | Gauge     |               | Number of validators who tried to double sign                          |
| consensus_byzantine_validators_power   | Gauge     |               | Total voting power of the byzantine validators                         |
| consensus_block_interval_seconds       | Histogram |               | Time between this and last block (Block.Header.Time) in seconds        |
| consensus_rounds                       | Gauge     |               | Number of rounds                                                       |
| consensus_num_txs                      | Gauge     |               | Number of transactions                                                 |
| consensus_total_txs                    | Gauge     |               | Total number of transactions committed                                 |
| consensus_block_parts                  | counter   | peer_id       | number of blockparts transmitted by peer                               |
| consensus_latest_block_height          | gauge     |               |/status sync_info number                                               |
| consensus_fast_syncing                 | gauge     |               | either 0 (not fast syncing) or 1 (syncing)                             |
| consensus_state_syncing                | gauge     |               | either 0 (not state syncing) or 1 (syncing)                            |
| consensus_block_size_bytes             | Gauge     |               | Block size in bytes                                                    |
| p2p_peers                              | Gauge     |               | Number of peers node's connected to                                    |
| p2p_peer_receive_bytes_total           | counter   | peer_id, chID | number of bytes per channel received from a given peer                 |
| p2p_peer_send_bytes_total              | counter   | peer_id, chID | number of bytes per channel sent to a given peer                       |
| p2p_peer_pending_send_bytes            | gauge     | peer_id       | number of pending bytes to be sent to a given peer                     |
| p2p_num_txs                            | gauge     | peer_id       | number of transactions submitted by each peer_id                       |
| p2p_pending_send_bytes                 | gauge     | peer_id       | amount of data pending to be sent to peer                              |
| mempool_size                           | Gauge     |               | Number of uncommitted transactions                                     |
| mempool_tx_size_bytes                  | histogram |               | transaction sizes in bytes                                             |
| mempool_failed_txs                     | counter   |               | number of failed transactions                                          |
| mempool_recheck_times                  | counter   |               | number of transactions rechecked in the mempool                        |
| state_block_processing_time            | histogram |               | time between BeginBlock and EndBlock in ms                             |

## 便利なクエリ

欠落している+ビザンチンバリデーターの割合:

```prometheus
((consensus_byzantine_validators_power + consensus_missing_validators_power)/consensus_validators_power) * 100
```

Rate at which the application is responding to each ABCI method call.
```
sum(rate(tendermint_abci_connection_method_timing_count[5m])) by (method)
```

The 95th percentile response time for the application to the `deliver_tx` ABCI method call.
```
histogram_quantile(0.95, sum by(le) (rate(tendermint_abci_connection_method_timing_bucket{method="deliver_tx"}[5m])))
```
