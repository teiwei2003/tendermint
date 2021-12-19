# ADR 006:信頼測定設計

## 環境

提案された信頼インデックスにより、Tendermintは、直接対話するピアのローカル信頼ランキングを維持できます。これを使用して、ソフトセキュリティ制御を実装できます。計算は[TrustGuard](https://dl.acm.org/citation.cfm?id=1060808)プロジェクトから行われます。

### バックグラウンド

Tendermintコアプロジェクトの開発者は、ピアツーピアネットワークのピアによって表示される信頼性のレベルを追跡することにより、Tendermintのセキュリティと信頼性を向上させることを望んでいます。このように、ピアからの悪い結果がすぐにネットワークから削除されることはありません(大幅な変更につながる可能性があります)。代わりに、適切なインジケーターを使用してピアの動作を監視し、Tendermint Coreがピアが脅威をもたらすと判断した後、ピアの動作をネットワークから削除できます。たとえば、PEXReactorが既知のピアからピアネットワークアドレスを要求し、返されたネットワークアドレスに到達できない場合、この信頼できない動作を追跡する必要があります。間違ったネットワークアドレスを返してもピアが破棄されない場合があります。この動作が多すぎると、ピアが破棄されます。

悪意のあるノードは、戦略オシレーションテクノロジーを使用して信頼インジケーターを回避できます。このテクノロジーは、悪意のあるノードの動作パターンを調整して、目標を最大化します。たとえば、悪意のあるノードがTendermintの信頼メトリックについて学習する時間間隔が_X_時間である場合、悪意のあるアクティビティ間で_X_時間待機する可能性があります。間隔の長さを増やすことでこの問題の解決を試みることができますが、これにより、最近のイベントに対するシステムの適応性が低下します。

それどころか、間隔は短くなりますが、間隔値の履歴記録を保持することで、ネットワークの安定性を維持するために必要な柔軟性をインジケーターに提供すると同時に、Tendermintピアツーピアネットワークの戦略的な悪意のあるノードに抵抗することができます。 。さらに、このインジケーターは、最新の時間間隔の高精度を維持しながら、多数の時間間隔で古い履歴値を集約することによって履歴レコードサイズを大幅に増加させることなく、かなりの期間、信頼データにアクセスできます。この方法は色あせた記憶と呼ばれ、人間が自分の経験を覚える方法と非常によく似ています。履歴データを使用することのトレードオフは、ノードの実行間で間隔の値を維持する必要があることです。

### 参照する

S. Mudhakar、L。Xiong、およびL. Liu、「TrustGuard:分散型カバレッジネットワークのレピュテーション管理における脆弱性への対処」、第14回ワールドワイドウェブ国際会議の議事録、pp。422-431、2005年5月。

## 決定

提案された信頼メトリックにより、開発者は、ピアツーピアの動作に関連するすべての良いイベントと悪いイベントを信頼メトリックストアに通知でき、いつでもメトリックを照会して、ピアの現在の信頼ランキングを取得できます。

次の3つのサブセクションでは、信頼レベルの計算プロセス、信頼メトリックストレージの概念、および信頼メトリックのインターフェイスについて説明します。

### 提案されたプロセス

提案された信頼メトリックは、オブジェクトに関連する良いイベントと悪いイベントを計算し、事前定義された期間の間隔で良いカウンターのパーセンテージを計算します。これは、信頼測定のライフサイクルで継続するプロセスです。現在の**信頼値**の信頼メトリックを照会する場合、弾性方程式を使用して計算が実行されます。

提案された方程式は、制御システムで使用される比例積分微分(PID)コントローラーに似ています。比例コンポーネントを使用すると、最も近い間隔の値に敏感になり、積分コンポーネントを使用すると、履歴データに格納されている信頼値を組み込むことができ、微分コンポーネントを使用すると、動作の突然の変化に重みを付けることができます。ピア。現在の信頼レベル、間隔_i_(過去の_maxH_間隔)の前の信頼評価履歴、および信頼レベルの変動に基づいて、間隔iのピアの信頼値を計算します。方程式を3つの部分に分解します。

```math
(1) Proportional Value = a * R[i]
```

ここで、_R _ [* i *]は時間間隔_i_の元の信頼値(_i_ == 0は現在の時刻)を表し、_a_は現在のレポートの貢献に適用される重みです。 方程式の次のコンポーネントは、最後の_maxH_間隔の加重和を使用して、時間_i_の履歴値を計算します。

`H [i] =`！[formula1](img/ Formula1.png "加重和式")

重みは、楽観的または悲観的に選択できます。 楽観的な重みは、新しい履歴データ値に対してより大きな重みを作成しますが、悲観的な重みは、スコアが低い時間間隔に対してより大きな重みを作成します。 履歴値の計算プロセスで使用されるデフォルトの重みは楽観的であり、_Wk_ = 0.8 ^ _k_として計算され、時間間隔は_k_です。 履歴値を使用して、積分値の計算を完了することができます。

```math
(2) Integral Value = b * H[i]
```

ここで、_H _ [* i *]は時間間隔_i_の履歴値を表し、_b_は測定されたオブジェクトの過去のパフォーマンス寄与に適用された重みです。 微分成分の計算は次のとおりです。

```math
D[i] = R[i] – H[i]

(3) Derivative Value = c(D[i]) * D[i]
```

_c_の値は、ゼロを基準にした_D _ [* i *]の値に基づいて選択されます。 デフォルトの選択プロセスでは、_D _ [* i *]が負の値でない限り、_c_は0に等しくなります。負の値の場合、cは1に等しくなります。 その結果、現在の動作が以前に経験した動作よりも低い場合に最大のペナルティが適用されます。 現在の動作が以前の動作よりも優れている場合、派生値は信頼値に影響を与えません。 3つの要素を組み合わせると、信頼値の式は次のように計算されます。

```math
TrustValue[i] = a * R[i] + b * H[i] + c(D[i]) * D[i]
```

保存された元の間隔データの量を妥当なサイズ_m_に維持するためのパフォーマンスの最適化として、2 ^ _m_-1の履歴間隔を表現できるようにする一方で、フェージングメモリテクノロジーを使用できます。新しい値の数は、履歴データ値の精度を向上させるために使用されます。 上記の式は、最も多くの_maxH_(2 ^ _m_-1になる可能性があります)にアクセスしようとしますが、次の式4を使用して、これらの要求を_m_値にマップします。

```math
(4) j = index, where index > 0
```

ここで、_j_は、履歴間隔データにアクセスするために使用される_(0、1、2、…、m – 1)_インデックスの1つです。 これで、次の計算を使用して生の間隔にアクセスできます。

```math
R[0] = raw data for current time interval
```

`R[j] =` ![formula2](img/formula2.png "Fading Memories Formula")

### 信頼インジケーターストレージ

P2PサブシステムAddrBookと同様に、トラストメトリックストアはTendermintノードに関連する情報を維持します。 さらに、トラストメトリックストレージは、ノードが現在直接参加しているピアに対してのみトラストメトリックが有効であることを保証します。

Reactorは、関連するトラストメトリックを取得するために、トラストメトリックストアにピアキーを提供します。 信頼メトリックは、リアクターが経験した新しい正および負のイベントを記録し、メトリックによって計算された現在の信頼スコアを提供できます。

ノードがシャットダウンされると、トラストメトリックストアは、すべての既知のピアに関連付けられているトラストメトリックの履歴データを保存します。 この保存された情報により、ノード間で保存とピアエクスペリエンスを実行できます。これは、数日または数週間の追跡ウィンドウにまたがることができます。 信頼履歴データは、OnStart中に自動的にロードされます。

### インターフェースの詳細設計

各信頼インジケーターを使用すると、正/負のイベントを記録し、現在の信頼値/スコアを照会し、時間間隔で追跡を停止/一時停止できます。 これは以下で見ることができます:

```go
// TrustMetric - keeps track of peer reliability
type TrustMetric struct {
   // Private elements.
}

// Pause tells the metric to pause recording data over time intervals.
// All method calls that indicate events will unpause the metric
func (tm *TrustMetric) Pause() {}

// Stop tells the metric to stop recording data over time intervals
func (tm *TrustMetric) Stop() {}

// BadEvents indicates that an undesirable event(s) took place
func (tm *TrustMetric) BadEvents(num int) {}

// GoodEvents indicates that a desirable event(s) took place
func (tm *TrustMetric) GoodEvents(num int) {}

// TrustValue gets the dependable trust value; always between 0 and 1
func (tm *TrustMetric) TrustValue() float64 {}

// TrustScore gets a score based on the trust value always between 0 and 100
func (tm *TrustMetric) TrustScore() int {}

// NewMetric returns a trust metric with the default configuration
func NewMetric() *TrustMetric {}

//------------------------------------------------------------------------------------------------
// For example

tm := NewMetric()

tm.BadEvents(1)
score := tm.TrustScore()

tm.Stop()
```

一部の信頼度測定パラメーターを構成できます。 多くの場合、重み値は個別に保持される場合がありますが、追跡ウィンドウの期間と個別の時間間隔を考慮する必要があります。

```go
// TrustMetricConfig - Configures the weight functions and time intervals for the metric
type TrustMetricConfig struct {
   // Determines the percentage given to current behavior
    ProportionalWeight float64

   // Determines the percentage given to prior behavior
    IntegralWeight float64

   // The window of time that the trust metric will track events across.
   // This can be set to cover many days without issue
    TrackingWindow time.Duration

   // Each interval should be short for adapability.
   // Less than 30 seconds is too sensitive,
   // and greater than 5 minutes will make the metric numb
    IntervalLength time.Duration
}

// DefaultConfig returns a config with values that have been tested and produce desirable results
func DefaultConfig() TrustMetricConfig {}

// NewMetricWithConfig returns a trust metric with a custom configuration
func NewMetricWithConfig(tmc TrustMetricConfig) *TrustMetric {}

//------------------------------------------------------------------------------------------------
// For example

config := TrustMetricConfig{
    TrackingWindow: time.Minute * 60 * 24,// one day
    IntervalLength:    time.Minute * 2,
}

tm := NewMetricWithConfig(config)

tm.BadEvents(10)
tm.Pause()
tm.GoodEvents(1)// becomes active again
```

永続ストレージを備えたデータベースを使用してトラストメトリックストアを作成し、ノード間で履歴データを保存できるようにする必要があります。 ストアによってインスタンス化されるすべての信頼メトリックは、提供されたTrustMetricConfig構成を使用して作成されます。

ピアのトラストメトリックを取得しようとして、トラストメトリックストアにエントリがない場合、新しいメトリックが自動的に作成され、エントリがストアに作成されます。

getメソッドGetPeerTrustMetricに加えて、トラストメトリックストアは、ピアがノードから切断されたときに呼び出されるメソッドも提供します。 このようにして、ノードがピアを直接経験していない場合、インジケーターを一定期間一時停止できます(履歴データは保存されません)。

```go
// TrustMetricStore - Manages all trust metrics for peers
type TrustMetricStore struct {
    cmn.BaseService

   // Private elements
}

// OnStart implements Service
func (tms *TrustMetricStore) OnStart(context.Context) error { return nil }

// OnStop implements Service
func (tms *TrustMetricStore) OnStop() {}

// NewTrustMetricStore returns a store that saves data to the DB
// and uses the config when creating new trust metrics
func NewTrustMetricStore(db dbm.DB, tmc TrustMetricConfig) *TrustMetricStore {}

// Size returns the number of entries in the trust metric store
func (tms *TrustMetricStore) Size() int {}

// GetPeerTrustMetric returns a trust metric by peer key
func (tms *TrustMetricStore) GetPeerTrustMetric(key string) *TrustMetric {}

// PeerDisconnected pauses the trust metric associated with the peer identified by the key
func (tms *TrustMetricStore) PeerDisconnected(key string) {}

//------------------------------------------------------------------------------------------------
// For example

db := dbm.NewDB("trusthistory", "goleveldb", dirPathStr)
tms := NewTrustMetricStore(db, DefaultConfig())

tm := tms.GetPeerTrustMetric(key)
tm.BadEvents(1)

tms.PeerDisconnected(key)
```

## ステータス

公式に認められています。

## 結果

### ポジティブ

-トラストメトリックにより、Tendermintは非バイナリのセキュリティと信頼性の決定を行うことができます
-Tendermintが、ネットワークの停止を回避しながらソフトなセキュリティ制御を提供する抑止力を実装するのに役立ちます
-ピアツーピアの相互作用に関連する一定期間のパフォーマンスを分析するときに、有用な分析情報を提供します

### ネガティブ

-過去の信頼測定データを保存するには、ノード間の実行が必要です

### ニュートラル

-この実装を使用して、良いイベントを悪いイベントのように記録する必要があることを忘れないでください
