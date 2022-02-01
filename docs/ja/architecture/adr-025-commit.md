# ADR025送信

## 環境

現在、 `Commit`構造には、冗長または不要な可能性のあるデータが多数含まれています.
これには、各バリデーターからの事前コミットのリストが含まれています.
「投票」構造全体を含みます. したがって、各提出物の高さ、円、
typeとblockIDはバリデーターごとに繰り返され、重複排除できます.
これにより、ブロックサイズが大幅に節約されます.

```
type Commit struct {
    BlockID    BlockID `json:"block_id"`
    Precommits []*Vote `json:"precommits"`
}

type Vote struct {
    ValidatorAddress Address   `json:"validator_address"`
    ValidatorIndex   int       `json:"validator_index"`
    Height           int64     `json:"height"`
    Round            int       `json:"round"`
    Timestamp        time.Time `json:"timestamp"`
    Type             byte      `json:"type"`
    BlockID          BlockID   `json:"block_id"`
    Signature        []byte    `json:"signature"`
}
```

元の追跡の問題は[#1648](https://github.com/tendermint/tendermint/issues/1648)です.
`Commit`の` Vote`タイプを新しい `CommitSig`に置き換えることについてはすでに説明しました
タイプ.少なくとも投票署名が含まれます. `投票`タイプは
コンセンサスリアクターやその他の場所で引き続き使用されます.

主な質問は、 `CommitSig`に何を含めるべきかということです.
サイン.現在の制限の1つは、タイムスタンプを含める必要があることです.
これはBFT時間を計算する方法ですが、これを変更することもできます[in
将来](https://github.com/tendermint/tendermint/issues/2840).

ここでの他の問題は次のとおりです.

-ベリファイアアドレス[#3596](https://github.com/tendermint/tendermint/issues/3596)-
    CommitSigにベリファイアアドレスを含める必要がありますか？とても便利
    これを行いますが、必要ない場合もあります.これについては、[#2226](https://github.com/tendermint/tendermint/issues/2226)でも説明されています.
-不在者投票[#3591](https://github.com/tendermint/tendermint/issues/3591)-
    不在者投票をどのように表現するのですか？現在、これらは「nil」としてのみ表示されます.
    事前コミットのリストは、実際にはシリアル化に問題があります
-その他のBlockID [#3485](https://github.com/tendermint/tendermint/issues/3485)-
    ゼロおよびその他のブロックIDへの投票を表す方法は？現在許可しています
    nilに投票し、代替ブロックIDに投票しますが、無視します


## 決定

重複するフィールドを削除し、 `CommitSig`を導入します.

```
type Commit struct {
    Height  int64
    Round   int
    BlockID    BlockID      `json:"block_id"`
    Precommits []CommitSig `json:"precommits"`
}

type CommitSig struct {
    BlockID  BlockIDFlag
    ValidatorAddress Address
    Timestamp time.Time
    Signature []byte
}


// indicate which BlockID the signature is for
type BlockIDFlag int

const (
	BlockIDFlagAbsent BlockIDFlag = iota // vote is not included in the Commit.Precommits
	BlockIDFlagCommit                    // voted for the Commit.BlockID
	BlockIDFlagNil                       // voted for nil
)

```

文脈で概説された質問に関して:

**タイムスタンプ**:タイムスタンプを一時的に保持します.削除してに切り替えます
提案者の時間に基づいて、より多くの分析と作業が必要になり、
将来の画期的な変更.同時に、現在の方法に関する懸念
BFT時間[
緩和策](https://github.com/tendermint/tendermint/issues/2840#issuecomment-529122431).

** ValidatorAddress **:これを `CommitSig`に含めます.これが
ブロックサイズ(バリデーターあたり20バイト)が不必要に大きくなり、人間工学的およびデバッグ上の利点がいくつかあります.

-`Commit`には、 `[] Vote`を再構築するために必要なすべてのものが含まれており、` ValidatorSet`への追加アクセスに依存しません.
-Liteクライアントは、送信時にバリデーターを知っているかどうかを確認できます.
  バリデーターセットを再ダウンロードします
-提出物で直接署名していない検証者を簡単に確認できます
  バリデーターセットを取得する

たとえば、タイムスタンプを削除するなど、 `CommitSig`を再度変更すると、
ValidatorAddressを削除する必要があるかどうかを再検討できます.

**不在者投票**:署名なしで、または不在者投票を明示的に含めます
タイムスタンプですが、ValidatorAddressを使用します.これでシリアル化が解決するはずです
質問があり、どの検証者の投票が含まれていないかを簡単に確認できます.

**その他のブロックID **:どのブロックIDが `CommitSig`であるかを示すために1バイトを使用します
です.唯一のオプションは次のとおりです.
    -`Absent`-この検証者からの投票はなかったため、署名はありません
    -`Nil`-検証者はゼロに投票しました-彼らが時間内にポルカを見なかったことを意味します
    -`Commit`-バリデーターがこのブロックに投票します

これは、他のブロックIDに投票することが許可されていないことを意味することに注意してください.署名が
送信に含まれるのは、nilまたは正しいblockIDのいずれかです.によると
テンダーミントのプロトコルと仮定、正しい検証者はできません
同じラウンドの実際の送信で競合するブロックIDを事前にコミットする
作成.これはみんなのコンセンサスです
[#3485](https://github.com/tendermint/tendermint/issues/3485)

キャプチャする方法として、将来的に他のblockIDのサポートを検討する可能性があります
役立つ証拠があるかもしれません.これを行うかどうか/いつ/どのように行うかを明確にする必要があります
実際に最初に助けてください.これを実現するために、 `Commit.BlockID`を変更できます
スライスするフィールド.最初のエントリは正しいブロックIDで、もう1つのエントリは
エントリは、ベリファイアによって以前に送信されたもう1つのBlockIDです. BlockIDFlag
列挙型を拡張して、各ブロックのこれらの追加のブロックIDを表すことができます
ベース.

## ステータス

実装

## 結果

### ポジティブ

Type/Height/Round/IndexとBlockIDを削除すると、事前コミットごとに約80バイトを節約できます.
一部の整数はvarintであるため、これは異なります. BlockIDには、2つの32バイトハッシュと整数が含まれています.
高さは8バイトです.

100個のバリデーターを持つチェーンの場合、各ブロックで最大8kBを節約できます.


### ネガティブ

-ブロックとコミットの構造に対する主な変更
-VoteオブジェクトとCommitSigオブジェクトの間でコードを区別する必要があります.これにより、複雑さが増す可能性があります(検証とゴシップのために投票をリファクタリングする必要があります)

### ニュートラル

-Commit.Precommitsにnil値が含まれなくなりました
