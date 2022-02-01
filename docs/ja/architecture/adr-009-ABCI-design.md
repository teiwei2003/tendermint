# ADR 009:ABCIユーザーエクスペリエンスの向上

##変更ログ

23-06-2018:レビューのいくつかのマイナーな修正
2018年7月6日:ジェとの話し合いに基づくいくつかの更新
2018年7月6日:ABCIv0.11で公開されたコンテンツと一致する最初のドラフト

## 環境

ABCIは2015年の終わりに最初に立ち上げられました.目的は次のとおりです.

-ステートマシンとそのレプリケーションエンジン間の共通インターフェイス
-ステートマシンが書かれている言語とは何の関係もありません
-それを駆動するレプリケーションエンジンとは何の関係もありません

これは、ABCIがプラグ可能なアプリケーションであり、
プラグ可能なコンセンサスエンジン.

これを実現するために、メッセージタイプとしてプロトコルバッファ(proto3)を使用します.リーディングで
実装はGoにあります.

githubでコミュニティと最近話し合った後、次のようになります.
課題として決定:

-アミノコーディングタイプ
-バリデーターセットを管理する
-protobufファイルにインポートします

詳細については、[参照](#references)を参照してください.

### 輸入

Goのネイティブprotoライブラリは、柔軟性がなく冗長なコードを生成します.
Goコミュニティの多くの人々はと呼ばれるシステムを採用しています
[gogoproto](https://github.com/gogo/protobuf)は
開発者エクスペリエンスを向上させるために設計されたさまざまな機能.
`gogoproto`は優れていますが、追加の依存関係を作成してコンパイルします
他の言語のProtobufタイプは、gogoprotoを使用すると失敗することが報告されています.

###アミノ

Aminoは、protobufの欠点を改善するために設計されたエンコーディングプロトコルです.
その目標はproto4になることです.

多くの人がprotobufの非互換性に不満を持っています.
また、ABCIではアミノ基を完全に使用する必要があります.

最終的にABCIに使用できるように、Aminoを十分に成功させるつもりです.
メッセージタイプは直接です.それまでに、proto4と呼ばれるはずです.同時に、
使いやすいものにしたいと思っています.

### 公開鍵

PubKeysはAminoエンコーディング(以前はgo-wire)を使用します.
理想的には、公開鍵はインターフェースタイプであり、すべてを知っているわけではありません
実装型であるため、 `oneof`または` enum`の使用には適していません.

### 住所

ED25519の公開鍵アドレスはAminoのRIPEMD160です.
エンコードされた公開鍵.これにより、アドレス生成にアミノ依存性が導入されます.
広く必要とされ、計算しやすい関数
可能.

###バリデーター

バリデーターセットを変更するために、アプリケーションはバリデーター更新リストに戻ることができます
そしてResponseEndBlock.これらの更新には、公開鍵を含める必要があります.
Tendermintは、検証者の署名を検証するために公開鍵を必要とするためです.この
これは、ABCI開発者がPubKeysを使用する必要があることを意味します.言い換えれば、これも
住所情報の処理は非常に便利で、操作も非常に簡単です.

### バリデーターがありません

Tendermintは、BeginBlockの符号なしベリファイアのリストも提供します
最後のピース.これにより、アプリケーションは可用性の動作を反映できます
投票を含まないバリデーターにペナルティを課すなどのアプリケーション
提出中.

### 初期化チェーン

Tendermintは、ここでバリデーターのリストを渡しますが、それ以上のものはありません.そうなる
アプリケーションがバリデーターの初期セットを制御できるようにします.にとって
たとえば、ジェネシスファイルにはアプリケーションベースの情報を含めることができます
アプリケーションが決定するために処理できるバリデーターの初期セット
初期バリデーターセット.さらに、InitChainはすべてを取得することで恩恵を受けます
作成情報.

### タイトル

ABCIはRequestBeginBlockでヘッダーを提供するため、アプリケーションは
ブロックチェーンの最新の状態に関する重要な情報.

## 決定

### 輸入

gogoprotoに近づかないでください.短期的には、1秒しか維持しません
protobufファイル、gogoprotoコメントはありません.中期的には
Golangのすべての構造をコピーし、前後にシャトルします.長い間
用語では、アミノを使用します.

###アミノ

短期的にABCIアプリケーション開発を簡素化するために、
AminoはABCIから完全に削除されます.

-公開鍵エンコーディングはそれを必要としません
-公開鍵アドレスを計算する必要はありません

言い換えれば、私たちはアミノを大成功させ、proto4になるために一生懸命取り組んでいます.
短期的に採用と言語間の互換性を促進するために、Amino
v1は:

-`oneof`を含まないproto3サブセットと完全に互換性があります
-`oneof`の代わりにAminoプレフィックスシステムを使用してインターフェイスタイプを提供します
  スタイル共用体タイプ.

言い換えれば、アミノv2のパフォーマンスを向上させるために働きます
暗号化されたアプリケーションでのフォーマットとその使いやすさ.


### 公開鍵

コーディングスキームはソフトウェアに感染する可能性があります.一般的なミドルウェアとして、ABCIの目標は
プログラム間の互換性.このため、不透明度を含める以外に選択肢はありません.
時々バイト.これらのアミノコードは強制しませんが
バイト数については、型システムを提供する必要があります.最も簡単な方法は
タイプ文字列を使用します.

公開鍵は次のようになります.
```
message PubKey {
    string type
    bytes data
}
```

`type`は次のようになります.

-"Ed225519" with `data = <raw 32byte pubkey>`
-"Secp256k1" with `data = <33バイトのOpenSSL圧縮公開鍵>`

ここで柔軟性を維持したいので、理想的には、PubKeyは
インターフェイスタイプ.`enum`または `oneof`は使用しません.

### 住所

アドレスの計算を簡素化および改善するために、アドレスをSHA256の最初の20バイトに変更します.
元の32バイトの公開鍵.

secp256k1キーには引き続きビットコインアドレススキームを使用します.

### バリデーター

「バイトアドレス」フィールドを追加します.

```
message Validator {
    bytes address
    PubKey pub_key
    int64 power
}
```

### RequestBeginBlockとAbsentValidators

これを単純化するために、RequestBeginBlockにはバリデーターの完全なセットが含まれます.
各バリデーターの住所と議決権を含め、
ブール値を使用して、投票したかどうかを示します.

```
message RequestBeginBlock {
  bytes hash
  Header header
  LastCommitInfo last_commit_info
  repeated Evidence byzantine_validators
}

message LastCommitInfo {
  int32 CommitRound
  repeated SigningValidator validators
}

message SigningValidator {
    Validator validator
    bool signed_last_block
}
```

RequestBeginBlockのオーセンティケーターには公開鍵が含まれていないことに注意してください. 公開鍵は
将来的には、量子コンピューターでアドレスよりも大きく、
より大きい. 特に高速同期中にそれらを渡すオーバーヘッドは、
重要.

さらに、アドレスの計算が容易になり、さらに削除されます
公開鍵をここに含める必要があります.

つまり、ABCI開発者は、アドレスと公開鍵を知っている必要があります.

### ResponseEndBlock

ResponseEndBlockにはバリデーターが含まれているため、それらのアドレスが含まれている必要があります.

###初期化チェーン

RequestInitChainを変更して、ジェネシスファイル内のすべての情報をアプリケーションに提供します.

```
message RequestInitChain {
    int64 time
    string chain_id
    ConsensusParams consensus_params
    repeated Validator validators
    bytes app_state_bytes
}
```

ResponseInitChainを変更して、アプリケーションがバリデーターの初期セットを指定できるようにします
そしてコンセンサスパラメータ.

```
message ResponseInitChain {
    ConsensusParams consensus_params
    repeated Validator validators
}
```

### タイトル

これで、TendermintAminoはABCIのヘッダーであるproto3と互換性があります.
Tendermintヘッダーと正確に一致する必要があります-その後、エンコードされます
ABCIとテンダーミントコアでも同じです.

## ステータス

実装

## 結果

### ポジティブ

-開発者はABCIに基づいて構築する方が簡単です
-ABCIヘッダーとTendermintヘッダーは同じシリアル化です

### ネガティブ

-代替コーディングスキームのメンテナンスオーバーヘッド
-各ブロックのすべてのバリデーター情報を渡すことによるパフォーマンスのオーバーヘッド(少なくとも
  公開鍵ではなく、アドレスのみ)
-繰り返されるタイプのメンテナンスオーバーヘッド

### ニュートラル

-ABCI開発者はバリデーターアドレスを知っている必要があります

## 参照する

-[ABCI v0.10.3仕様(以前
  提案)](https://github.com/tendermint/abci/blob/v0.10.3/specification.rst)
-[ABCI v0.11.0仕様(この仕様の最初のドラフトの実装)
  提案)](https://github.com/tendermint/abci/blob/v0.11.0/specification.md)
-[Ed25519アドレス](https://github.com/tendermint/go-crypto/issues/103)
-[InitChainには
  ジェネシス](https://github.com/tendermint/abci/issues/216)
-[公開鍵](https://github.com/tendermint/tendermint/issues/1524)
- [予防
  タイトル](https://github.com/tendermint/tendermint/issues/1605)
-[Gogoproto Issue](https://github.com/tendermint/abci/issues/256)
-[バリデーターがありません](https://github.com/tendermint/abci/issues/231)
