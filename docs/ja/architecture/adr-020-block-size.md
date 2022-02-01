# ADR 020:ブロック内のtxのサイズを制限する

## 変更ログ

2018年8月13日:最初のドラフト
15-08-2018:開発レビュー後の2番目のバージョン
2018年8月28日:イーサンのコメントに続く3番目のバージョン
30-08-2018:AminoOverheadForBlock => MaxAminoOverheadForBlock
31-08-2018:境界証拠とチェーンID
13-01-2019:MaxBytesとMaxDataBytesに関する部分を追加しました

## 環境

現在、ブロックを提案するときに、MaxTxsを使用してメモリプールからtxを取得しています.
ただし、ブロックをアンマーシャリングするときにMaxBytesが適用されるため、簡単に
ブロックが大きすぎて有効ではありません.

MaxTxsを一緒に削除し、MaxBytesに固執する必要があります.
`mempool.ReapMaxBytes`.

ただし、MaxBytesはブロック全体を対象としているため、BlockSize.MaxBytesだけを収集することはできません.
ブロック内のtxには適用されません.追加のアミノオーバーヘッド+実際があります
実際の取引の上部にあるタイトル+証拠+最後の提出.
MaxBytesの代わりに、またはMaxBytesに加えてMaxDataBytesを使用することも検討できます.

## MaxBytesおよびMaxDataBytes

[PR#3045](https://github.com/tendermint/tendermint/pull/3045)提案
の使用を考慮して、ここで追加の説明/理由が必要です
MaxBytesに加えてまたはMaxBytesの代わりにMaxDataBytes.

MaxBytesは、ブロックの合計サイズに明確な制限を提供します.必要はありません.
追加の計算、リソース使用量を制限するためにそれを使用したい場合は、
1MBブロックでTendermintを最適化することについてはかなり多くの議論がありました.
いずれにせよ、回避できるように最大ブロックサイズが必要です
アンマーシャリングコンセンサス中に大きすぎるブロックは、より多くのようです
の代わりに固定番号を指定してください
「MaxDataBytes +スペースを確保するために必要なその他すべて」を計算します
(署名、証拠、タイトル) ".MaxBytesは単純な境界を提供するため、次のことができます.
常に「ブロックはXMBよりも小さい」と言ってください.

MaxBytesとMaxDataBytesを同時に持つことは、不必要な複雑さのように感じます.それは
MaxBytesは、最大サイズが特に驚くべきことではないことを意味します
ブロック全体(txsだけでなく)については、ブロックにタイトルが含まれていることだけを知っておく必要があります.
取引、証拠、投票.よりきめ細かい制御のために
ブロック、MaxGasがあります.実際には、MaxGasはほとんどのことを行う可能性があります
txスロットリングとMaxBytesは合計の上限にすぎません
サイズ.アプリケーションはMaxGasをMaxDataBytesとして使用でき、ガスを使用するだけです.
各txはそのサイズ(バイト単位)です.

## 推奨される解決策

したがって、

1)MaxTxsを取り除きます.
2)MaxTxsBytesの名前をMaxBytesに変更します.

メモリプールからReapMaxBytesを取得する必要がある場合、上限を次のように計算します.

```
ExactLastCommitBytes = {number of validators currently enabled} * {MaxVoteBytes}
MaxEvidenceBytesPerBlock = MaxBytes / 10
ExactEvidenceBytes = cs.evpool.PendingEvidence(MaxEvidenceBytesPerBlock) * MaxEvidenceBytes

mempool.ReapMaxBytes(MaxBytes - MaxAminoOverheadForBlock - ExactLastCommitBytes - ExactEvidenceBytes - MaxHeaderBytes)
```

それらの中で、MaxVoteBytes、MaxEvidenceBytes、MaxHeaderBytes、およびMaxAminoOverheadForBlock
定数は `types`パッケージで定義されていますか？

-MaxVoteBytes-170バイト
-MaxEvidenceBytes-364バイト
-MaxHeaderBytes-476バイト(〜276バイトのハッシュ+200バイト-50UTF-8エンコーディング
  チェーンIDのシンボルは、最悪の場合、それぞれ4バイト+アミノオーバーヘッド)
-MaxAminoOverheadForBlock-8バイト(MaxHeaderBytesにアミノが含まれていると仮定)
  エンコーディングヘッダーのオーバーヘッド、MaxVoteBytes-エンコーディング投票など)

ChainIDは最大50個のシンボルをバインドする必要があります.

証拠を収集するときは、MaxBytesを使用して上限を計算します(例:1/10)
トランザクションのためにいくらかのスペースを節約します.

メモリプールで `max int`バイトを取得するときは、それぞれを考慮する必要があることに注意してください
トランザクションは `len(tx)+ aminoOverhead`を使用します.ここでaminoOverhead = 1-4バイトです.

基礎となる構造が変更された場合、テストを作成する必要がありますが、失敗しますが、
MaxXXXは変更されません.

## ステータス

実装

## 結果

### ポジティブ

*ブロックサイズを制限する方法
*変動の少ない構成

### ネガティブ

*基礎となる構造が変更されたときに調整する必要がある定数

### ニュートラル
