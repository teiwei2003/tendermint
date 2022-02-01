# ADR 033:パブリッシュおよびサブスクライブ2.0

著者:アントン・カリアエフ(@melekes)

## 変更ログ

2018年2月10日:最初のドラフト

2019年1月16日:ジェとの会話に基づく2番目のバージョン

2017年1月17日:第3版では、新しい設計が現在の問題をどのように解決するかを説明しています

25-01-2019:4番目のバージョンは、バッファリングされたチャネルとバッファリングされていないチャネルを異なる方法で処理します

## 環境

pubsubの初期バージョン以来、多くの問題がありました
提出済み:[#951]、[#1879]、[#1880]. それらのいくつかは質問しています
コアデザインの選択が行われました. その他は二次的で、主に
`Subscribe()`/`Publish()`関数.

###同期および非同期

これで、サブスクライバーにメッセージを公開するときに、ゴルーチンでそれを行うことができます.

_using channels for data transmission_
```go
for each subscriber {
    out := subscriber.outc
    go func() {
        out <- msg
    }
}
```

_by invoking callback functions_
```go
for each subscriber {
    go subscriber.callbackFn()
}
```

これにより、パフォーマンスが向上し、「クライアントの速度低下の問題」を回避できます.
(他のサブスクライバーが遅いサブスクライバーを待たなければならない場合). プール
Goroutinesは、制御されていないメモリの増加を回避するために使用できます.

場合によっては、これが必要なものです. しかし、私たちの例では、
イベントの厳密な順序(イベントAがBの前に公開されている場合、それは保証されます
配信順序はA-> B)となり、毎回新しいゴルーチンでメッセージを公開することはできません.

注意が必要ですが、サブスクライバーごとにゴルーチンを設定することもできます.
そして加入者の数. 実装も難しい+
私たちがそれから利益を得るかどうかは明らかではありません(私たちはさらにNを作成することを余儀なくされるため)
これらのゴルーチンチャネルにメッセージを配布します).

### ノンブロッキング送信

ノンブロッキング送信を実行する必要があるときはいつでも、別の問題があります.
現在、送信はブロックされているため、クライアントへの投稿はブロックできます
別の人に公開します. これは、遅いまたは応答しないクライアントが停止する可能性があることを意味します
システム. 代わりに、ノンブロッキング送信を使用できます.

```go
for each subscriber {
    out := subscriber.outc
    select {
        case out <- msg:
        default:
            log("subscriber %v buffer is full, skipping...")
    }
}
```

これは「遅いクライアントの問題」を解決しますが、遅いクライアントはできません
メッセージを見逃したかどうかを確認します. 2番目のチャネルに戻って閉じることができます
サブスクリプションが終了したことを示します.一方、私たちが望むなら
送信のブロックを主張します.**開発者は常にサブスクライバーの処理コードを確認する必要があります
**をブロックしないでください.これは彼らの肩にかかる骨の折れる作業です.

一時的なオプションは、単一のメッセージに対してgoroutinesプールを実行し、すべてを待機することです.
完了するためのgoroutines.これは「遅いクライアントの問題」を解決しますが、それでも
次のメッセージを公開する前に、 `max(goroutine_X_time)`を待つ必要があります.

###チャネルとコールバック

もう1つの質問は、メッセージ送信にチャネルを使用する必要があるかどうかです.
サブスクライバーによって定義されたコールバック関数を呼び出します.加入者へのコールバック機能
柔軟性の向上-ミューテックス、チャネルを使用し、そこでゴルーチンを生成できます.
あなたが本当に欲しいものは何でも.しかし、それらにはローカルスコープもあり、これは
メモリリークおよび/またはメモリ使用量の増加.

Goチャネルは、ゴルーチン間でデータを転送するための事実上の標準です.

### `Subscribe()`が `out`チャネルを受け入れるのはなぜですか？

テストでは、バッファチャネルを作成したためです(上限:1).または私たち
容量をパラメータとして使用して、チャネルに戻ることができます.

## 決定

### MsgAndTags

サブスクライブされたチャネルで `MsgAndTags`構造を使用して、どのタグを示すか
msgが一致します.

```go
type MsgAndTags struct {
    Msg interface{}
    Tags TagMap
}
```

### サブスクリプション構造


`Subscribe()`関数を変更して、 `Subscription`構造を返します.

```go
type Subscription struct {
  // private fields
}

func (s *Subscription) Out() <-chan MsgAndTags
func (s *Subscription) Canceled() <-chan struct{}
func (s *Subscription) Err() error
```

`Out()`は、メッセージとラベルを公開するためのチャネルを返します.
`Unsubscribe` /` UnsubscribeAll`は、クライアントを避けるためにチャネルを閉じません
nilメッセージを受信しました.

`Canceled()`は、サブスクリプションが終了したときに閉じられたチャネルを返します
そして、選択ステートメントで使用する必要があります.

`Canceled()`によって返されたチャネルが閉じられていない場合、 `Err()`はnilを返します.
チャネルが閉じている場合、 `Err()`は理由を説明するnil以外のエラーを返します.
`ErrUnsubscribed`サブスクライバーがサブスクライブ解除を選択した場合、
`ErrOutOfCapacity`サブスクライバーがメッセージを十分に速くプルせず、` Out() `によって返されるチャネルがいっぱいの場合.
`Err()`がゼロ以外のエラーを返した後、 `Err()を連続して呼び出すと同じエラーが返されます.

```go
subscription, err := pubsub.Subscribe(...)
if err != nil {
  // ...
}
for {
select {
  case msgAndTags <- subscription.Out():
    // ...
  case <-subscription.Canceled():
    return subscription.Err()
}
```

### 容量とサブスクリプション

デフォルトでは、 `Out()`チャネルはバッファリングされます(容量は1). ほとんどの場合、
遅いサブスクライバーを終了します. ごくまれに、公開と購読を禁止したい場合があります
(たとえば、コンセンサスをデバッグする場合). これにより、公開と購読の機会が減るはずです
凍っていた.

```go
// outCap can be used to set capacity of Out channel
// (1 by default, must be greater than 0).
Subscribe(ctx context.Context, clientID string, query Query, outCap... int) (Subscription, error) {
```

Use a different function for an unbuffered channel:

```go
// Subscription uses an unbuffered channel. Publishing will block.
SubscribeUnbuffered(ctx context.Context, clientID string, query Query) (Subscription, error) {
```

SubscribeUnbuffered 不应向用户公开.

### ブロッキング/非ブロッキング

サイト運営者は、これらのタイプのチャネルを個別に扱う必要があります.
バッファリングされていないチャネルをブロックする必要があります(内部コンセンサスイベントの場合)
コンセンサステストでは)バッファリングを防ぐ代わりに. 顧客も
そのニュースに追いつくのは遅く、そのサブスクリプションは終了しました:

for each subscription {
    out := subscription.outChan
    if cap(out) == 0 {
        // block on unbuffered channel
        out <- msg
    } else {
        // don't block on buffered channels
        select {
            case out <- msg:
            default:
                // set the error, notify on the cancel chan
                subscription.err = fmt.Errorf("client is too slow for msg)
                close(subscription.cancelChan)

                // ... unsubscribe and close out
        }
    }
}

###この新しい設計は、現在の問題をどのように解決しますか？

[#951]([#1880]):

ノンブロッキング送信のため、デッドロックは発生しません
上.クライアントがメッセージの読み取りを停止すると、メッセージは削除されます.

[#1879]:

通常のメッセージの代わりにMsgAndTagsを使用するようになりました.

###将来の問題と考えられる解決策

[#2826]

私がまだ考えている質問:pubsubが遅くなるのを防ぐ方法
コンセンサスを作成します.パブリッシュ/サブスクライブキューのサイズを増やすことができます(現在は0).また、
サブスクライバーの総数を制限することをお勧めします.

これは自動的に行うことができます.キューサイズを1000に設定し、それが> =の場合を想定します.
80％がいっぱいで、新しいサブスクリプションを拒否します.

## ステータス

実装

## 結果

### ポジティブ

-より慣用的なインターフェース
-サブスクライバーは、どのタグmsgが公開されているかを知っています
-サブスクライバーは、サブスクリプションがキャンセルされた理由を知っています

### ネガティブ

-(v1以降)メッセージを公開するときに同時実行性はありません

### ニュートラル


[#951]:https://github.com/tendermint/tendermint/issues/951
[#1879]:https://github.com/tendermint/tendermint/issues/1879
[#1880]:https://github.com/tendermint/tendermint/issues/1880
[#2826]:https://github.com/tendermint/tendermint/issues/2826
