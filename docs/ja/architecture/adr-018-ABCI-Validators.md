# ADR 018:ABCIバリデーターの改善

##変更ログ

016-08-2018:レビューとフォローアップ:-提出ラウンドへの変更を再開します-公開鍵を削除する理由を思い出させます-長所/短所を更新します
2018年5月8日:最初のドラフト

## 環境

ADR 009は、バリデーターと使用法に関してABCIに大幅な改善を加えました
アミノ. ここでは、命名を改善するためにいくつかの追加の変更をフォローアップしました
そして、バリデーターメッセージの使用目的.

## 決定

###バリデーター

現在、バリデーターには `address`と` pub_key`が含まれており、そのうちの1つは
オプション/非送信はユースケースによって異なります. 代わりに、
`Validator`(アドレスのみ、RequestBeginBlockに使用)
そして `ValidatorUpdate`(ResponseEndBlockの公開鍵付き):

```
message Validator {
    bytes address
    int64 power
}

message ValidatorUpdate {
    PubKey pub_key
    int64 power
}
```

[ADR-009](adr-009-ABCI-design.md)で説明されているように、
クォンタム公開鍵は
かなり大きく、ブロックごとにABCI全体に送信するのは無駄です.
したがって、BeginBlockの情報を使用したいアプリケーション
公開鍵を状態に保存するために_必要_(またははるかに効率の悪い怠惰な方法を使用する
BeginBlockデータを確認します).

### RequestBeginBlock

LastCommitInfoには現在、 `SigningValidator`配列があります.
バリデーター全体が、各バリデーターの情報を一元化します.
代わりに、これは「VoteInfo」と呼ばれるべきです.
検証者が投票します.

提出物のすべての投票は同じラウンドからのものでなければならないことに注意してください.

```
message LastCommitInfo {
  int64 round
  repeated VoteInfo commit_votes
}

message VoteInfo {
    Validator validator
    bool signed_last_block
}
```

### ResponseEndBlock

Validatorsの代わりにValidatorUpdatesを使用してください. それなら明らかに必要ありません
住所、公開鍵が必要です.

ここで住所と健全性チェックを依頼できますが、そうではないようです
必要.

###初期化チェーン

リクエストとレスポンスの両方にValidatorUpdatesを使用します. 初期チェーン
BeginBlockとは異なり、初期バリデーターセットの設定/更新に関するものです
これは情報提供のみです.

## ステータス

実装

## 結果

### ポジティブ

-ベリファイア情報のさまざまな使用法の違いを明確にしました

### ネガティブ

-アプリケーションは、RequestBeginBlock情報を使用するために、公開鍵を状態で保存する必要があります

### ニュートラル

-ResponseEndBlockはアドレスを必要としません

## 参照する

-[最新のABCI仕様](https://github.com/tendermint/tendermint/blob/v0.22.8/docs/app-dev/abci-spec.md)
-[ADR-009](https://github.com/tendermint/tendermint/blob/v0.22.8/docs/architecture/adr-009-ABCI-design.md)
-[問題#1712-PubKeyを送信しないでください
   RequestBeginBlock](https://github.com/tendermint/tendermint/issues/1712)
