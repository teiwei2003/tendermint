# ADR 059:証拠の構成とライフサイクル

## 変更ログ

2020年4月9日:最初のドラフト(要約なし)
-2020年7月9日:最初のバージョン
-13/03/2021:フォワードクレイジーアタックに対応するように変更
-29/06/2021:ABCI固有のフィールドに関する情報を追加

## スコープ

このドキュメントは、テンダーミントの証拠に関連するジレンマのいくつかを整理して明らかにすることを目的としています.その構成とライフサイクルです.次に、これらの問題の解決策を見つけることを目的としています.範囲は、特定の種類の証拠の検証またはテストには及びませんが、主に証拠の一般的な形式と、それが最初から適用にどのように適用されるかを含みます.

## バックグラウンド

長い間、コンセンサスリアクターで形成された「DuplicateVoteEvidence」は、テンダーミントが持っている唯一の証拠です.同じラウンドで同じバリデーターが2票投票されたときに生成されます
観察されたため、各証拠は単一の検証者向けに設計されています.より多くの形式の証拠が存在する可能性があると予測されるため、「DuplicateVoteEvidence」が「証拠」インターフェースのモデルおよびアプリケーションに送信される証拠データの形式として使用されます. Tendermintは証拠の検出と報告にのみ焦点を当てており、アプリケーションには罰する責任があることに注意してください.
```go
type Evidence interface { //existing
  Height() int64                                     // height of the offense
  Time() time.Time                                   // time of the offense
  Address() []byte                                   // address of the offending validator
  Bytes() []byte                                     // bytes which comprise the evidence
  Hash() []byte                                      // hash of the evidence
  Verify(chainID string, pubKey crypto.PubKey) error // verify the evidence
  Equal(Evidence) bool                               // check equality of evidence

  ValidateBasic() error
  String() string
}
```

```go
type DuplicateVoteEvidence struct {
  VoteA *Vote
  VoteB *Vote

  timestamp time.Time // taken from the block time
}
```

Tendermintは、ライトクライアントを攻撃から保護するための新しいタイプの証拠を導入しました.この `LightClientAttackEvidence`([ここ](https://github.com/informalsystems/tendermint-rs/blob/31ca3e64ce90786c1734caf186e30595832297a4/docs/spec/lightclient/attacks/evidenceを参照)は、情報処理が異なると大きく異なります.`DuplicateVoteEvidence`物理的に大きく異なるため、完全な署名ヘッダーとベリファイアセットが含まれています.コンセンサスリアクタではなく、ライトクライアントで形成され、検証するために状態情報からさらに多くの情報が必要です( `VerifyLightClientAttack(commonHeader、trustedHeader * SignedHeader、 commonVals * ValidatorSet) `と` VerifyDuplicateVote(chainID string、pubKey PubKey) `)最後に、個別の証拠(各部分のエビデンスは、各高さの各検証ツールです).このエビデンスは、新しいタイプのエビデンスに対応するための既存のモデルを拡張し、エビデンスのフォーマットと処理の方法を再検討するよう促します.

```go
type LightClientAttackEvidence struct { // proposed struct in spec
  ConflictingBlock *LightBlock
  CommonHeight int64
  Type  AttackType     // enum: {Lunatic|Equivocation|Amnesia}

  timestamp time.Time // taken from the block time at the common height
}
```
*注:これらの3つの攻撃タイプは、調査チームによって網羅的であることが証明されています*

## 証拠の組み合わせの可能な方法

### パーソナルフレーム

証拠は、各検証者に基づいて残ります.これにより、現在のプロセスへの干渉は最小限に抑えられますが、「LightClientAttackEvidence」を悪意のあるバリデーターごとにいくつかの証拠に分解する必要があります.データベース操作の数がn倍であるため、これはパフォーマンスに影響を与えるだけでなく、証拠ゴシップは(各部分にヘッダーを必要とすることにより)より多くの帯域幅を必要とし、それを検証する能力に影響を与える可能性があります.バッチ処理の形式では、ノード全体がライトクライアントと同じプロセスを実行して、共通ブロックと競合ブロックの両方に検証能力の3分の1があり、悪意のあるバリデーターを開く可能性がないことを確認できます. 、この変更無実の人々に有害な証拠の偽造を独自に検証することはさらに困難です.それだけでなく、「LightClientAttackEvidence」は記憶喪失攻撃も処理します.残念ながら、この攻撃の特徴は、関係するバリデーターのセットを知っていることですが、それが実際に悪意のあるサブセットであるかどうかはわかりません(この後の手順で詳しく説明します).最後に、証拠を別々の部分に分割すると、攻撃の重大度(つまり、攻撃に関与する総投票権)を理解することが困難になります

#### 可能な実装パスの例

記憶喪失の証拠を無視し(個別に作成することが難しいため)、以前の最初の分割に戻ります.ここでは、「DuplicateVoteEvidence」が軽いクライアントのあいまいさの攻撃にも使用されるため、「LunaticEvidence」のみが必要です.また、これは実際には使用できないため、インターフェイスから「検証」を削除する必要がある可能性があります.

``` go
type LunaticEvidence struct { // individual lunatic attack
  header *Header
  commonHeight int64
  vote *Vote

  timestamp time.Time // once again taken from the block time at the height of the common header
}
```

### バッチ処理フレームワーク

このカテゴリの最後の方法は、バッチ証拠のみを考慮することです.これは「LightClientAttackEvidence」に適用されますが、「DuplicateVoteEvidence」を変更する必要があります.これは、コンセンサスが矛盾する投票を証拠モジュールのバッファーに送信し、すべての投票を高さでまとめてから、ゴシップすることを意味します.他のノードを使用して、チェーンに送信してみてください.一見すると、これによりIOと検証の速度が向上する可能性があり、さらに重要なことに、バリデーターをグループ化することで、アプリケーションとTendermintが攻撃の重大度をよりよく理解できるようになります.

ただし、単一の証明の利点は、ノードにすでに証明があるかどうかを簡単に確認できることです.つまり、ハッシュ値を確認するだけで、以前に証明を検証したことがわかります.バッチ証拠は、各ノードが繰り返し投票の異なる組み合わせを持っている可能性があることを意味し、問題を複雑にする可能性があります.

#### 可能な実装パスの例

`LightClientAttackEvidence`は変更されませんが、証拠インターフェースは上記で提案されたもののように見える必要があり、` DuplicateVoteEvidence`は複数の二重投票を含むように変更する必要があります.バッチ証拠に関する1つの問題は、人々が異なる順列を提出することを避けるために、それが一意である必要があるということです.

## 決定

決定は、ハイブリッド設計を採用することでした.

単一のエビデンスとエビデンスのバッチを共存させることができます.つまり、検証はエビデンスのタイプに従って実行され、ほとんどの作業はエビデンスプール自体で行われます(アプリケーションに送信されるエビデンスの形成を含む).


## 詳細設計

証拠には、次の単純なインターフェイスがあります.

```go
type Evidence interface {  //proposed
  Height() int64                                     // height of the offense
  Bytes() []byte                                     // bytes which comprise the evidence
  Hash() []byte                                      // hash of the evidence
  ValidateBasic() error
  String() string
}
```

これらのメソッドはすべて以前のバージョンのインターフェイスに存在するため、インターフェイスの変更には下位互換性があります. ただし、検証が変更されると、新しい証拠を処理するためにネットワークをアップグレードする必要があります.

このインターフェースを満たすために、2つの特定のタイプの証拠があります

```go
type LightClientAttackEvidence struct {
  ConflictingBlock *LightBlock
  CommonHeight int64 // the last height at which the primary provider and witness provider had the same header

  // abci specific information
	ByzantineValidators []*Validator // validators in the validator set that misbehaved in creating the conflicting block
	TotalVotingPower    int64        // total voting power of the validator set at the common height
	Timestamp           time.Time    // timestamp of the block at the common height
}
```
その中で、 `Hash()`はヘッダーとcommonHeightのハッシュ値です.

注:ヘッダーに署名するバリデーターをキャプチャするためにコミットハッシュを含めるかどうかについても説明します. ただし、これにより、同じ証拠の複数の順列を(異なる提出署名を介して)提示する機会が誰かに提供されるため、省略されます. したがって、ブロック内の証拠を検証する場合、「LightClientAttackEvidence」の場合、誰かが私たちと同じハッシュ値を持っている可能性があるため、ハッシュ値を確認するだけでなく、1未満の別の送信を送信することはできません./3検証者は、これが無効な証拠のバージョンになると投票しました. (詳細は「fastCheck」を参照してください)

```go
type DuplicateVoteEvidence {
  VoteA *Vote
  VoteB *Vote

  // abci specific information
	TotalVotingPower int64
	ValidatorPower   int64
	Timestamp        time.Time
}
```
ここで、 `Hash()`は2票のハッシュ値です.

これら2種類の証拠の場合、 `Bytes()`は証拠の元のエンコーディングバイト配列形式を表し、 `ValidateBasic`は
証拠が有効な構造を持っていることを確認するための最初の整合性チェック.

###証拠プール

`LightClientAttackEvidence`はライトクライアントで生成され、` DuplicateVoteEvidence`はコンセンサスで生成されます.両方とも「AddEvidence(evEvidence)エラー」を介してエビデンスプールに送信されます.エビデンスプールの主な目的は、エビデンスを検証することです.また、証拠ゴシップを他のノードの証拠プールに送信し、それをコンセンサスに提供してチェーンに提出し、関連情報を罰の申請に送信することもできます.エビデンスを追加する場合、プールは最初に「Has(ev Evidence)」を実行して(ハッシュ値を比較することにより)受信されたかどうかを確認し、次に「Verify(ev Evidence)error」を実行します.検証後、証拠プールはそれを保留中のデータベースとして保存します. 2つのデータベースがあります.1つはまだ提出されていない保留中の証拠に使用され、もう1つは提出された証拠に使用されます(証拠を2回提出することは避けてください)

#### 確認

`Verify()`は次のことを行います.

-ハッシュを使用して、この証拠が提出したデータベースにすでに存在するかどうかを確認します.

-高度を使用して、証拠の有効期限が切れていないかどうかを確認します.

-有効期限が切れている場合は、高さを使用してブロックヘッダーを検索し、期限も切れているかどうかを確認します.この場合、証拠を破棄します

-次に、2つの証拠のそれぞれについてswitchステートメントを作成します.

`DuplicateVote`の場合:

-高さ、円、タイプ、バリデーターアドレスが同じかどうかを確認します

-ブロックIDが異なるかどうかを確認します

-アドレスルックアップテーブルをチェックして、このバリデーターの証拠がないことを確認します

-バリデーターのセットを取得し、アドレスが攻撃の高さのセットに含まれていることを確認します

-チェーンIDと署名が有効かどうかを確認します.

`LightClientAttack`の場合

-パブリックハイトからパブリック署名ヘッダーとvalセットを取得し、スキップ検証を使用して競合するヘッダーを検証します

-競合ヘッダーと同じ高さの信頼できる署名ヘッダーを取得し、それを競合ヘッダーと比較して、攻撃の種類を判別します.その場合、悪意のあるベリファイアを返します.注:ノードに競合するヘッダーの高さに署名ヘッダーがない場合、ノードは最新のヘッダーを取得し、違反ヘッダーの時間に基づいて証拠を証明できるかどうかを確認します.これはフォワードクレイジーアタックと呼ばれます.

  -あいまいな場合は、信頼され署名されたヘッダーの署名を送信したバリデーターに戻ります

  -気が狂っている場合は、競合ブロックで署名された公開検証セットから検証者に戻ります

  -メモリが記憶喪失の場合、バリデーターは返されません(どのバリデーターが悪意があるかわからないため).これはまた、Tendermint Coreの将来のバージョンで記憶喪失のより強力な証拠を導入するものの、現在、アプリケーションに記憶喪失の証拠を送信しないことを意味します

-競合するヘッダーと信頼できるヘッダーのハッシュ値が異なるかどうかを確認します

-フォワードマッドマン攻撃の場合、信頼できるヘッドの高さは競合するヘッドの高さよりも低く、ノードは競合するヘッドよりも遅く信頼できるヘッドをチェックします.これは、競合するヘッダーが単調に増加する時間を中断することを証明しています.将来、ノードに信頼できるヘッダーがない場合、現在は証拠を検証できません.

-最後に、バリデーターごとに、ルックアップテーブルをチェックして、そのバリデーターの証拠がないことを確認します.

検証後、キーワード「height/hash」を含むエビデンスをエビデンスプールの保留中のエビデンスデータベースに保存します.

#### ABCIの証拠

どちらのタイプの証拠構造にも、アプリケーションに渡すために必要なデータ(タイムスタンプなど)が含まれていますが、厳密に言えば、これらは不適切な動作の証拠を構成するものではありません.したがって、最後にこれらのフィールドを確認してください.これらのフィールドのいずれかがノードに対して無効である場合、つまり、それらがそれらの状態に対応していない場合、ノードは既存のフィールドから新しい証拠構造を再構築し、abci固有のフィールドに独自の状態データを再入力します.

####ブロードキャストして証拠を受け取る

証拠プールもリアクターを実行し、新しく検証されたものをブロードキャストします
接続されているすべてのピアに証拠を提供します.

他のエビデンスリアクターからエビデンスを受け取る方法は、コンセンサスリアクターまたはライトクライアントからエビデンスを受け取る方法と同じです.


####ブロックに証拠を提案する

証拠を含む提案の事前投票と事前提出に関しては、ノード全体が再び
エビデンスプールを呼び出し、 `CheckEvidence(ev [] Evidence)`を使用してエビデンスを検証します.

これにより、次のことが行われます.

1.すべての証拠を調べて、重複がないことを確認します

2.エビデンスごとに、「fastCheck(evevidence)」を実行します.「Has」のように機能しますが、「LightClientAttackEvidence」がある場合は
次に、同じハッシュが、所有するバリデーターが競合するヘッダー送信のすべての署名者であるかどうかを引き続きチェックします.クイックチェックに失敗した場合(以前に証拠を見たことがないため)、証拠を検証する必要があります.

3. `Verify(ev Evidence)`を実行します-注:これにより、前述のように、証拠もデータベースに保存されます.


#### アプリケーションとプールを更新する

ライフサイクルの最後の部分はブロックを送信することであり、次に「BlockExecutor」が状態を更新します.このプロセスの一環として、「BlockExecutor」は証拠のプールを取得して、アプリケーションに送信される証拠の簡略化された形式を作成します.これは「ApplyBlock」で発生し、実行プログラムは「Update(Block、State)[] abci.Evidence」を呼び出します.

```go
abciResponses.BeginBlock.ByzantineValidators = evpool.Update(block, state)
```

以下は、アプリケーションが受け取る証拠の形式です. 上に示したように、これは「BeginBlock」に配列として格納されます.
証明タイプとして文字列の代わりに列挙型を使用することを除けば、アプリケーションへの変更は最小限です(悪意のあるバリデーターごとに1つを形成します).

```go
type Evidence struct {
  // either LightClientAttackEvidence or DuplicateVoteEvidence as an enum (abci.EvidenceType)
	Type EvidenceType `protobuf:"varint,1,opt,name=type,proto3,enum=tendermint.abci.EvidenceType" json:"type,omitempty"`
	// The offending validator
	Validator Validator `protobuf:"bytes,2,opt,name=validator,proto3" json:"validator"`
	// The height when the offense occurred
	Height int64 `protobuf:"varint,3,opt,name=height,proto3" json:"height,omitempty"`
	// The corresponding time where the offense occurred
	Time time.Time `protobuf:"bytes,4,opt,name=time,proto3,stdtime" json:"time"`
	// Total voting power of the validator set in case the ABCI application does
	// not store historical validators.
	// https://github.com/tendermint/tendermint/issues/4581
	TotalVotingPower int64 `protobuf:"varint,5,opt,name=total_voting_power,json=totalVotingPower,proto3" json:"total_voting_power,omitempty"`
}
```

`Update()`関数は次のことを行います.

-有効期限による現在の時間と高度を測定するために使用される増分ステータスを追跡します

-証​​拠を提出済みとしてマークし、データベースに保存します. これにより、検証者が提出された証拠を将来提示することを防ぎます
    注:dbは、高さとハッシュ値のみを保存します. 提出されたすべての証拠を保存する必要はありません

-そのようなABCIエビデンスを作成します:(「DuplicateVoteEvidence」に注意してください.バリデーター配列のサイズは1です)
  ```go
  for _, val := range evInfo.Validators {
    abciEv = append(abciEv, &abci.Evidence{
      Type: evType,   // either DuplicateVote or LightClientAttack
      Validator: val,   // the offending validator (which includes the address, pubkey and power)
      Height: evInfo.ev.Height(),    // the height when the offense happened
      Time: evInfo.time,      // the time when the offense happened
      TotalVotingPower: evInfo.totalVotingPower   // the total voting power of the validator set
    })
  }
  ```

-保留中およびコミット済みのデータベースから期限切れの証拠を削除します

次に、「BlockExecutor」を介してABCI証拠をアプリケーションに送信します.

#### 概要

全体として、証拠のライフサイクルは次のとおりであることがわかります.

！[evidence_lifecycle](../imgs/evidence_lifecycle.png)

まず、ライトクライアントとコンセンサスリアクターで証拠を検出して作成します.検証されて「EvidenceInfo」として保存され、他のノードの証拠プールに渡されます.コンセンサスリアクターは、後で証拠プールと通信して、ブロックに配置される証拠を取得するか、ブロック内のコンセンサスリアクターによって取得された証拠を検証します.最後に、ブロックがチェーンに追加されると、ブロックエグゼキュータは送信されたエビデンスをエビデンスプールに送り返します.これにより、エビデンスへのポインタをエビデンスプールに格納し、その高さと時間を更新できます.最後に、提出された証拠をABCI証拠に変換し、アプリケーションが処理できるように、ブロックエグゼキュータを介して証拠をアプリケーションに渡します.

## ステータス

実装

## 結果

<！->このセクションでは、決定を適用した場合の結果について説明します. 「ポジティブ」な結果だけでなく、すべての結果をここに要約する必要があります. ->

### ポジティブ

-エビデンスはエビデンスプール/モジュールによりよく含まれています
-LightClientAttackは一緒にとどまります(検証と帯域幅がより簡単です)
-LightClientAttackで送信されたシグナルを変更しても、複数の順列や複数の証拠は発生しません
-証​​拠マッピングアドレスは、DOS攻撃を防ぐことができます.この攻撃では、単一の検証者が多数の証拠を送信することにより、ネットワーク上でDOS攻撃を実行できます.

### ネガティブ

-`Evidence`のインターフェースを変更したので、ブロックを壊す変更です
-ABCI `Evidence`が変更されたため、ABCIにとって大きな変更です.
-証​​拠プールが住所/時刻を照会できないという証拠はありません

### ニュートラル


## 参照する

<！->関連するPRコメント、この問題の原因となった問題、または特定の設計を選択した理由に関する参考記事はありますか？もしそうなら、ここにそれらをリンクしてください！ ->

-[LightClientAttackEvidence](https://github.com/informalsystems/tendermint-rs/blob/31ca3e64ce90786c1734caf186e30595832297a4/docs/spec/lightclient/attacks/evidence-handling.md)
