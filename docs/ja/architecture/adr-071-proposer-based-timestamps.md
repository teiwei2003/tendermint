# ADR 71:提案者のタイムスタンプに基づく

## 変更ログ

 -2021年7月15日:@williambanfieldによって作成されました
 -2021年8月4日:@williambanfieldがドラフトを完成
 -2021年8月5日:ドラフトを更新して、@ williambanfieldのデータ構造の変更を含めます
 -2021年8月20日:@williambanfieldが言語編集を完了
 -2021年10月25日:@casonからの@williambanfieldの更新された仕様に一致するようにADRを更新します
 -2021年11月10日:@casonのフィードバックによると、@ williambanfieldは他の言語を更新しました

## ステータス

 **承認済み**

## 環境

Tendermintは現在、[BFTTime](https://github.com/tendermint/spec/blob/master/spec/consensus/bft-time.md)と呼ばれる単調に増加するタイムソースを提供しています.
タイムソースを生成するためのこのメカニズムは非常に単純です.
すべての正しいバリデーターは、送信するすべての「pre-commit」メッセージにタイムスタンプを追加します.
送信するタイムスタンプは、ベリファイアが現在認識しているUnix時間か、どちらの値が大きいかに応じて、前のブロック時間より1ミリ秒長くなります.
ブロックが生成されると、提案者は、提案者が受信したすべての「事前コミット」メッセージの時間の加重中央値としてブロックタイムスタンプを選択します.
重みは、ネットワーク上の検証者の投票権または公平性に比例します.
タイムスタンプを生成するためのこのメカニズムは、決定論的であり、ビザンチンフォールトトレラントです.

タイムスタンプを生成するためのこの現在のメカニズムには、いくつかの欠点があります.
検証者は、選択したブロックのタイムスタンプが自分の現在の既知のUnix時間にどれだけ近いかについて合意する必要はありません.
さらに、任意の数の議決権 "> 1/3"で、ブロックのタイムスタンプを直接制御できます.
したがって、タイムスタンプはおそらく特に意味がありません.

これらの欠点は、Tendermintプロトコルでは問題があります.
ライトクライアントはタイムスタンプを使用してブロックを検証します.
ライトクライアントは、現在の既知のUnix時間とブロックタイムスタンプの間の対応に依存して、表示されるブロックを検証します.
ただし、「BFTTime」の制限により、現在知られているUnix時間はブロックタイムスタンプと大きく異なる場合があります.

提案者のタイムスタンプ仕様に基づいて、これらの問題を解決するために、ブロックタイムスタンプを生成する代替方法が提案されます.
提案者のタイムスタンプに基づいて、ブロックタイムスタンプを生成するための現在のメカニズムは、主に2つの方法で変更されます.

1.ブロック提案者は、現在の既知のUnix時刻を、「BFTTime」ではなく次のブロックのタイムスタンプとして提供するように変更されました.
1.提案されたブロックタイムスタンプが現在の既知のUnix時間に十分近い場合にのみ、正しいベリファイアが提案されたブロックタイムスタンプを承認します.

これらの変更の結果は、検証者の投票権の「<= 2/3」では制御できないより意味のあるタイムスタンプになります.
このドキュメントでは、対応する[プロポーザーベースのタイムスタンプ仕様](https://github.com/tendermint/spec/tree/master/spec/consensus/proposer-based-timestamp)を実装するためにTendermintで必要なコード変更の概要を説明します.

## 代替方法

### タイムスタンプを完全に削除します

さまざまな理由により、コンピュータの時計は必然的にずれます.
契約でタイムスタンプを使用するということは、タイムスタンプの信頼性が低いことを受け入れるか、契約の有効性の保証に影響を与えることを意味します.
この設計は、タイムスタンプの信頼性を高めるために、プロトコルの活性に影響を与える必要があります.
もう1つの方法は、ブロックプロトコルからタイムスタンプを完全に削除することです.
`BFTTime`は決定論的ですが、任意に不正確になる可能性があります.
ただし、信頼できる時間のソースがあると、ブロックチェーン上に構築されたアプリケーションやプロトコルに非常に役立ちます.

したがって、タイムスタンプを削除しないことにしました.
アプリケーションは通常、特定のトランザクションが特定の日、特定の期間、またはさまざまなイベントの後の特定の期間に発生することを想定しています.
これらすべてには、合意された時間の意味のある表現が必要です.
次のプロトコルとアプリケーション機能には、信頼できる時間のソースが必要です.
*テンダーミントライトクライアント[既知の時間に依存](https://github.com/tendermint/spec/blob/master/spec/light-client/verification/README.md#definitions-1)とブロック時間対応間の関係は、ブロック検証に使用されます.
* Tendermintの証拠の有効性は、[高さまたは時間](https://github.com/tendermint/spec/blob/8029cf7a0fcc89a5004e173ec065aa48ad5ba3c8/spec/consensus/evidence.md#verification)によって異なります.
* Cosmos Hubでの資産のステーキング解除[21日後に発生](https://github.com/cosmos/governance/blob/ce75de4019b0129f6efcbb0e752cd2cc9e6136d3/params-change/Staking.md#unbondingtime).
* IBCパケットは、[タイムスタンプまたは高さを使用してパケット送信をタイムアウトする](https://docs.cosmos.network/v0.43/ibc/overview.html#acknowledgements)を使用できます.

最後に、コスモスハブのインフレ分布は、時間の概算値を使用して年率を計算します.
この時間の概算値は、[ブロックの高さと1年間に生成されるブロックの推定数](https://github.com/cosmos/governance/blob/master/params-change/Mint.md#blocksperyear)を使用して計算されます.
提案者のタイムスタンプに基づいて、このインフレ計算でより意味のある正確な時間ソースを使用できるようになります.


## 決定

実装は提案者のタイムスタンプに基づいており、「BFTTime」を削除します.

## 詳細設計

### 概要

提案者に基づいてタイムスタンプを実装するには、Tendermintのコードにいくつかの変更を加える必要があります.
これらの変更は、次のコンポーネントを対象としています.
* `internal/consensus /`パッケージ.
* `state /`パッケージ.
* `Vote`、` CommitSig`および `Header`タイプ.
*コンセンサスパラメータ.

### `CommitSig`に変更

[CommitSig](https://github.com/tendermint/tendermint/blob/a419f4df76fe4aed668a6c74696deabb9fe73211/types/block.go#L604)構造には現在タイムスタンプが含まれています.
このタイムスタンプは、バリデーターがブロックに対して「pre-commit」を発行したときに認識された現在のUnix時間です.
このタイムスタンプは使用されなくなり、この変更で削除されます.

`CommitSig`は次のように更新されます.

```diff
type CommitSig struct {
	BlockIDFlag      BlockIDFlag `json:"block_id_flag"`
	ValidatorAddress Address     `json:"validator_address"`
--	Timestamp        time.Time   `json:"timestamp"`
	Signature        []byte      `json:"signature"`
}
```

### 「投票」メッセージへの変更

`Precommit`メッセージと` Prevote`メッセージは、共通の[投票構造](https://github.com/tendermint/tendermint/blob/a419f4df76fe4aed668a6c74696deabb9fe73211/types/vote.go#L50)を使用します.
この構造には現在、タイムスタンプが含まれています.
このタイムスタンプは[voteTime](https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/internal/consensus/state.go#L2241)関数を使用して設定されるため、投票時間は現在のUnix時間バリデーターに対応します.
事前コミットの場合、このタイムスタンプを使用して[LastCommitのブロックに含まれるCommitSig](https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/types/block.go#L754)を作成します
事前投票の場合、このフィールドは現在使用されていません.
提案者ベースのタイムスタンプは、ブロック内の提案者によって設定されたタイムスタンプを使用するため、投票メッセージにタイムスタンプを含める必要がなくなりました.
したがって、このタイムスタンプは使用できなくなり、削除されます.

`Vote`は次のように更新されます.

```diff
type Vote struct {
	Type             tmproto.SignedMsgType `json:"type"`
	Height           int64                 `json:"height"`
	Round            int32                 `json:"round"`
	BlockID          BlockID               `json:"block_id"` // zero if vote is nil.
--	Timestamp        time.Time             `json:"timestamp"`
	ValidatorAddress Address               `json:"validator_address"`
	ValidatorIndex   int32                 `json:"validator_index"`
	Signature        []byte                `json:"signature"`
}
```

### 新しいコンセンサスパラメータ

提案者ベースのタイムスタンプ仕様には、すべてのバリデーターで同じでなければならないいくつかの新しいパラメーターが含まれています.
これらのパラメータは、「PRECISION」、「MSGDELAY」、および「ACCURACY」です.

`PRECISION`および` MSGDELAY`パラメータは、提案されたタイムスタンプが受け入れ可能かどうかを判断するために使用されます.
プロポーザルのタイムスタンプが「タイムリー」であると見なされる場合、バリデーターはプロポーザルにのみ事前投票します.
プロポーザルのタイムスタンプが、バリデーターが認識しているUnix時間の `PRECISION`および` MSGDELAY`内にある場合、それは `timely`であると見なされます.
より具体的には、 `validatorLocalTime-PRECISION <proposalTime <validatorLocalTime + PRECISION + MSGDELAY`の場合、プロポーザルのタイムスタンプは` timely`です.

`PRECISION`パラメータと` MSGDELAY`パラメータはすべてのバリデータで同じである必要があるため、[コンセンサスパラメータ](https://github.com/tendermint/tendermint/blob/master/proto/tendermint/types/ params .proto#L13)as [duration](https://developers.google.com/protocol-buffers/docs/reference/google.protobuf#google.protobuf.Duration).

提案者ベースのタイムスタンプ仕様には、[新しい精度パラメーター](https://github.com/tendermint/spec/blob/master/spec/consensus/proposer-based-timestamp/pbts-sysmodel_001_draft.md# pbts-clocksync -external0).
直感的には、「精度」は、正しいベリファイアの「実際の」時間と現在の既知の時間との差を表します.
現在知られているバリデーターのUnix時間は、常にリアルタイムとは多少異なります.
`ACCURACY`は、絶対値としての、各検証者の時間と実際の時間の最大差です.
これは、コンピューターが独自に判断できるものではなく、Tendermintベースのチェーンを実行しているコミュニティによる見積もりとして指定する必要があります.
これは、[提案ステップのタイムアウトを計算する](https://github.com/tendermint/spec/blob/master/spec/consensus/proposer-based-timestamp/pbts-algorithm_001_draft.md#)ための新しいアルゴリズムで使用されます. pbts- alg-startround0).
すべてのバリデーターは同じ「精度」を持っていると想定されているため、コンセンサスパラメーターとして含める必要があります.

コンセンサスは、次のようにこの「タイムスタンプ」フィールドを含むように更新されます.

```diff
type ConsensusParams struct {
	Block     BlockParams     `json:"block"`
	Evidence  EvidenceParams  `json:"evidence"`
	Validator ValidatorParams `json:"validator"`
	Version   VersionParams   `json:"version"`
++	Timestamp TimestampParams `json:"timestamp"`
}
```

```go
type TimestampParams struct {
	Accuracy  time.Duration `json:"accuracy"`
	Precision time.Duration `json:"precision"`
	MsgDelay  time.Duration `json:"msg_delay"`
}
```

### ブロック提案ステップを変更する

#### 提案者はブロックタイムスタンプを選択します

Tendermintは現在、「BFTTime」アルゴリズムを使用してブロックの「Header.Timestamp」を生成しています.
[提案ロジック](https://github.com/tendermint/tendermint/blob/68ca65f5d79905abd55ea999536b1a3685f9f19d/internal/state/state.go#L269) `LastCommit.Commitの` sigs`の時間の加重中央値を提案されたブロックに設定します`Header.Timestamp`.

提案者に基づくタイムスタンプでは、提案者は引き続き `Header.Timestamp`にタイムスタンプを設定します.
`Header`で提案者が設定するタイムスタンプは、ブロックが[polka](https://github.com/tendermint/tendermint/blob/053651160f496bb44b107a434e3e6482530bb287/docs/introduction/what-is-tendermint.md)を受信したかどうかに基づきます. #consensus-overview)かどうか.

#### ポルカのブロックの提案は以前に受け取られていません

提案者が新しいブロックを提案している場合、提案者が現在知っているUnix時間を `Header.Timestamp`フィールドに設定します.
提案者は、この同じタイムスタンプを、送信する「提案」メッセージの「タイムスタンプ」フィールドにも設定します.

####以前にポルカを受け取った再提案ブロック

提案者が以前にネットワーク上でポルカを受信したブロックを再提案した場合、提案者はブロックのheader.Timestampを更新しません.
代わりに、提案者はまったく同じブロックを再提案します.
このように、提案されたブロックは、以前に提案されたブロックとまったく同じブロックIDを持ち、すでにブロックを受信した検証者は、ブロックを再度受信しようとする必要はありません.

提案者は、新しく提案されたブロックの「Header.Timestamp」を「Proposal」メッセージの「Timestamp」に設定します.

#### 提案者が待っています

ブロックタイムスタンプは単調に増加する必要があります.
「BFTTime」では、バリデーターの時計が遅れている場合、[バリデーターは前のブロックの時刻に1ミリ秒を追加し、投票メッセージで使用します](https://github.com/tendermint/tendermint/blob/ e8013281281985e3ada7819f42502b09623d24a0コンセンサス/state.go#L2246).
提案者に基づいてタイムスタンプを追加する目的は、ある程度のクロック同期を強制することであるため、バリデーターの時間を完全に無視するUnix時間のメカニズムは無効になります.

ベリファイアクロックは完全には同期されません.
したがって、提案者の現在の既知のUnix時間は、前のブロックのheader.Timeよりも短い場合があります.
提案者の現在の既知のUnix時間がheader.Timeよりも小さい場合、提案者は既知のUnix時間がそれを超えるまでスリープします.

この変更には、[defaultDecideProposal](https://github.com/tendermint/tendermint/blob/822893615564cb20b002dd5cf3b42b8d364cb7d9/internal/consensus/state.go#L1180)メソッドの変更が必要になります.
このメソッドは、提案者の時間が前のブロックのheader.Timeよりも大きい場合にトリガーされるタイムアウトをスケジュールする必要があります.
タイムアウトがトリガーされると、提案者は最終的に「提案」メッセージを送信します.

#### 提案されたステップタイムアウトの変更

現在、構成されたプロポーザルのタイムアウトに達し、プロポーザルが表示されない場合、プロポーザルを待機しているバリデーターはプロポーザルステップを通過します.
このタイムアウトロジックは、提案者のタイムスタンプに基づいて変更する必要があります.

提案者は、現在の既知のUnix時間がheader.Timeを超えるまで待機して、ブロックを提案します.
検証者は、推奨されるステップをいつタイムアウトするかを決定するときに、これと他の要因を考慮する必要があります.
具体的には、提案ステップのタイムアウトでは、検証者のクロックと提案者のクロックの潜在的な不正確さも考慮する必要があります.
さらに、提案者から他の検証者への提案メッセージの伝達が遅れる場合があります.

したがって、プロポーザルを待機しているバリデーターは、前のブロックの「Header.Time」がタイムアウトするまで待機する必要があります.
自身のクロックが不正確である可能性があり、提案者のクロックが不正確であり、メッセージが遅延していることを考慮すると、提案を待機しているバリデーターは、 `Header.Time + 2 * ACCURACY + MSGDELAY`の前のブロックまで待機します.
 仕様では、これを「waitingTime」と定義しています.

[提案されたステップのタイムアウトは、 `state.go`のenterPropose](https://github.com/tendermint/tendermint/blob/822893615564cb20b002dd5cf3b42b8d364cb7d9/internal/consensus/state.go#L1108)で設定されます.
`enterPropose`は、新しいコンセンサスパラメータを使用して待機時間を計算するように変更されます.
`enterPropose`のタイムアウトは、` waitingTime`と[構成されたプロポーザルステップのタイムアウト](https://github.com/tendermint/tendermint/blob/dc7c212c41a360bfe6eb38a6dd8c709bbc39aae7/config/config.go#L1013)の最大値に設定されます.

### 提案の検証ルールを変更する

提案されたブロックを検証するためのルールは、提案者に基づいてタイムスタンプを実装するように変更されます.
提案が「時間どおり」であることを確認するために、検証ロジックを変更します.

提案者に基づくタイムスタンプの仕様によると、ブロックがラウンドで+2/3の多数決を受け取らなかった場合にのみ、「適時性」をチェックする必要があります.
ブロックが前のラウンドで+2/3の過半数を獲得した場合、投票権の+2/3は、ブロックのタイムスタンプがラウンドの現在の既知のUnix時間に十分近いと見なします.

検証ロジックが更新され、前のラウンドで+2/3の事前投票を受け取っていないブロックの「適時性」がチェックされます.
ラウンドで予備投票の+2/3を獲得することは、しばしば「ポルカ」と呼ばれ、簡単にするためにこの用語を使用します.

#### 現在のタイムスタンプ検証ロジック

タイムスタンプ検証に必要な変更をよりよく理解するために、最初に、タイムスタンプ検証が現在Tendermintでどのように機能するかを詳細に説明します.

[validBlock関数](https://github.com/tendermint/tendermint/blob/c3ae6f5b58e07b29c62bfdc5715b6bf8ae5ee951/state/validation.go#L14)現在[提案されたブロックタイムスタンプを3つの方法で確認](https://github.com/tendermint /tendermint/blob/c3ae6f5b58e07b29c62bfdc5715b6bf8ae5ee951/state/validation.go#L118).
まず、検証ロジックは、このタイムスタンプが前のブロックのタイムスタンプよりも大きいかどうかをチェックします.

次に、ブロックのタイムスタンプが[ブロックのLastCommit]のタイムスタンプの加重中央値として正しく計算されていることを確認します(https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/types/block.go#L48) .

最後に、検証ロジックは「LastCommit.CommitSig」のタイムスタンプを検証します.
各 `CommitSig`の暗号署名は、投票バリデーターの秘密鍵を使用してブロック内のフィールドハッシュに署名することによって作成されます.
この `signedBytes`ハッシュの1つの項目は、` CommitSig`のタイムスタンプです.
`CommitSig`タイムスタンプを検証するために、検証者は投票を検証して、` CommitSig`タイムスタンプを含むフィールドのハッシュ値を作成し、このハッシュ値を署名と照合します.
これは[VerifyCommit関数](https://github.com/tendermint/tendermint/blob/e8013281281985e3ada7819f42502b09623d24a0/types/validation.go#L25)で発生します.

#### 未使用のタイムスタンプ検証ロジックを削除します

`BFTTime`検証は適用されなくなり、削除されます.
これは、バリデーターがブロックタイムスタンプが「LastCommit」タイムスタンプの加重中央値であるかどうかをチェックしなくなることを意味します.
具体的には、[validateBlock関数のMedianTime](https://github.com/tendermint/tendermint/blob/4db71da68e82d5cb732b235eeb2fd69d62114b45/state/validation.go#L117)の呼び出しを削除します.
`MedianTime`関数は完全に削除できます.

`CommitSig`にはタイムスタンプが含まれなくなるため、送信を検証するバリデーターは、暗号署名をチェックするために構築するフィールドハッシュに` CommitSig`タイムスタンプを含めなくなります.

#### ブロックがポルカを受け取らないときのタイムスタンプ検証

「提案」メッセージの[POLRound](https://github.com/tendermint/tendermint/blob/68ca65f5d79905abd55ea999536b1a3685f9f19d/types/proposal.go#L29)は、ブロックがポルカを受け取ったラウンドを示します.
`POLRound`フィールドの負の値は、ブロックが以前にネットワーク上で提案されたことがないことを示します.
したがって、検証ロジックは「POLRound <0」のときにチェックインします.

ベリファイアが `Proposal`メッセージを受信すると、ベリファイアは、` Proposal.Timestamp`がベリファイアが認識している現在のUnix時間よりも最大で `PRECISION`大きく、少なくとも現在の` PRECISION + MSGDELAY`よりも小さいことを確認します.既知のUnix時間.
タイムスタンプがこれらの範囲内にない場合、提案されたブロックは「時間内」とは見なされません.

「Proposal」メッセージに一致する完全なブロックが受信されると、バリデーターは、ブロックの「Header.Timestamp」のタイムスタンプがこの「Proposal.Timestamp」に一致するかどうかもチェックします.
`Proposal.Timestamp`を使用して` timely`をチェックすると、 `MSGDELAY`パラメータを微調整できます.これは、` Proposal`メッセージのサイズが変更されないため、ネットワーク上の完全なブロックよりも速くゴシップされる可能性があるためです.

バリデーターは、提案されたタイムスタンプが前の高さのブロックのタイムスタンプよりも大きいかどうかもチェックします.
タイムスタンプが前のブロックのタイムスタンプより大きくない場合、ブロックは有効とは見なされません.これは、現在のロジックと同じです.

#### ブロックがポルカを受け取ったときのタイムスタンプの検証

ブロックが再提案され、 `Prevote`の+2/3の過半数がネットワーク上で受信されると、再提案されたブロックの` Proposal`メッセージは `POLRound`として作成されます.つまり、`> = 0`.
プロポーザルメッセージに負でない「POLRound」が含まれている場合、バリデーターは「プロポーザル」が「インタイム」であるかどうかをチェックしません.
`POLRound`が負でない値の場合、各バリデーターは、` POLRound`で示されるラウンドで提案されたブロックの `Prevote`メッセージを受信することを確認します.

バリデーターが「POLRound」で提案されたブロックの「Prevote」メッセージを受信しない場合、バリデーターはnilをprevoteします.
バリデーターは、 `POLRound`で+2/3の事前投票が行われたことを確認したため、これは事前投票ロジックの変更を表すものではありません.

バリデーターは、提案されたタイムスタンプが前の高さのブロックのタイムスタンプよりも大きいかどうかもチェックします.
タイムスタンプが前のブロックのタイムスタンプより大きくない場合、ブロックは有効とは見なされません.これは、現在のロジックと同じです.

さらに、この検証ロジックを更新して、 `Proposal.Timestamp`が提案されたブロックの` Header.Timestamp`と一致するかどうかを確認できますが、投票が受信されたかどうかを確認するだけでブロックを確認できるため、あまり関係ありません.タイムスタンプは正しいです.

### 前のステップに変更

現在、バリデーターは次の3つの状況のいずれかで提案に投票します.

*ケース1:バリデーターにロックされたブロックがなく、有効なプロポーザルを受信しました.
*ケース2:ベリファイアにロックされたブロックがあり、ロックされたブロックに一致する有効なプロポーザルを受け取ります.
*ケース3:バリデーターにロックされたブロックがあり、ロックされたブロックと一致しない有効な提案が表示されますが、現在のラウンドまたはロックされたラウンド以上のラウンドで提案された領域が表示されますブロック+⅔はい、ブロックをロックするために投票してください.

上記のように、事前投票ステップに加える唯一の変更は、検証者が有効な提案であると見なすものです.

### プレコミットステップへの変更

提出前のステップは、多くの変更を必要としません.
「タイムリー」チェックを除いて、その提案検証ルールは、投票前のステップで検証が変更されるのと同じ方法で変更されます.コミット前検証では、タイムスタンプが「タイムリー」であるかどうかはチェックされません.

### 投票時間を完全に削除する

[voteTime](https://github.com/tendermint/tendermint/blob/822893615564cb20b002dd5cf3b42b8d364cb7d9/internal/consensus/state.go#L2229)は、現在の既知のバリデーター時間を指定し、次の「BFTTime」のタイムスタンプを計算するメカニズムです.前のブロック.
前のブロックのタイムスタンプがバリデーターの現在の既知のUnix時間よりも大きい場合、voteTimeは前のブロックのタイムスタンプよりも1ミリ秒大きい値を返します.
このロジックは複数の場所で使用され、提案者に基づくタイムスタンプには不要になりました.
したがって、完全に削除する必要があります.

## 将来の改善

* BLS署名の集約を実現します.
`Precommit`メッセージからフィールドを削除することで、署名を集約することができます.

## 結果

### ポジティブ

* `<2/3`のバリデーターは、ブロックのタイムスタンプに影響を与えなくなりました.
*ブロックタイムスタンプは、リアルタイムとの対応が強くなります.
*ライトクライアントブロック検証の信頼性を向上させます.
* BLS署名の集約を有効にします.
*証拠の有効性を確保するために、高さではなく時間を使用する証拠処理を有効にします.

### ニュートラル

*テンダーミントのアクティブな属性を変更します.
ライブネスでは、すべての正しいバリデーターが境界内でクロックを同期している必要があります.
ライブネスは、前進するためにバリデーターの時計も必要としますが、これは「BFTTime」では必要ありません.

### ネガティブ

*前の提案者と現在の提案者のローカルUnix時間の間に大きな偏差がある場合、提案ステップの長さが長くなる可能性があります.
このスキューは `PRECISION`の値によって制約されるため、大きすぎる可能性はほとんどありません.

*将来の現在のブロックタイムスタンプは、間違ったブロックタイムスタンプが終了するまでコンセンサスを一時停止する必要があります.そうでない場合は、同期されているが非常に不正確なクロックを維持する必要があります.

## 参照する

* [PBTS仕様](https://github.com/tendermint/spec/tree/master/spec/consensus/proposer-based-timestamp)
* [BFTTime仕様](https://github.com/tendermint/spec/blob/master/spec/consensus/bft-time.md)
