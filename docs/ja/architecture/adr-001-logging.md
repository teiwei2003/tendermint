# ADR 1:記録

## 環境

Tendermintの現在のロギングシステムは非常に静的であり、十分な柔軟性がありません.

問題:[358](https://github.com/tendermint/tendermint/issues/358)、[375](https://github.com/tendermint/tendermint/issues/375).

新しいシステムから得たいもの:

-各パッケージの動的ログレベル
-動的レコーダ設定(レコーダは処理構造にバインドされています)
-大会
-より視覚的に魅力的

ここでの「動的」とは、実行時にsmthを設定する機能を指します.

## 決定

### 1)1つのインターフェース

まず、すべてのライブラリ( `tmlibs`、Tendermintなど)のインターフェイスが必要です. 私の個人的な好みはgo-kitの `Logger`インターフェース(付録Aを参照)ですが、これは大きな変更です. さらに、まだレベルが必要です.

```go
# log.go
type Logger interface {
    Debug(msg string, keyvals ...interface{}) error
    Info(msg string, keyvals ...interface{}) error
    Error(msg string, keyvals ...interface{}) error

	  With(keyvals ...interface{}) Logger
}
```

補足:「情報」と「通知」の違いは微妙です. おそらく
「通知」はあり得ません. その一部として「パニック」や「致命的」が必要だとは思わないでください
インターフェース. これらの機能は、アシスタントとして実装できます. 実は
`tmlibs/common`にいくつかあります.

-`Debug`-開発者の拡張出力
-`Info`-ユーザーにとって役立つすべての情報
-`Error`-エラー

`Notice`は、メッセージ` Crit`-> `Error`に応じて、` Info`、 `Warn`、` Error`、または `Debug`になります.

このインターフェースは `tmlibs/log`と入力する必要があります. コア(tendermint/tendermint)の一部であるすべてのライブラリは、それに準拠する必要があります.

### 2)現在のフォーマットレコーダーを使用する

このインターフェイスに加えて、標準の出力ロガーを実装する必要があります.これは、ログを標準出力に出力するようにTendermintが構成されている場合に使用されます.

多くの人が現在の出力が好きだと言っているので、それを守りましょう.

```
NOTE[2017-04-25|14:45:08] ABCI Replay Blocks                       module=consensus appHeight=0 storeHeight=0 stateHeight=0
```

いくつかの小さな変更:

```
I[2017-04-25|14:45:08.322] ABCI Replay Blocks            module=consensus appHeight=0 storeHeight=0 stateHeight=0
```

レベルはエンコードに1文字とミリ秒のみを使用することに注意してください.

注:[logfmt](https://brandur.org/logfmt)など、他にも多くの形式があります.

このロガーは、任意のロガーを使用して実装できます-[logrus](https://github.com/sirupsen/logrus)、[go-kit/log](https://github.com/go-kit/kit/tree/master/log)、[zap](https://github.com/uber-go/zap)、log15はここで終了します

a)カラー出力をサポート<br>
b)中程度の速度(バッファー)<br>
c)新しいインターフェースに準拠するか、そのためのアダプターを作成できます<br>
d)ある程度構成可能<br>

これまでのところ、go-kitが私のお気に入りです. エラーを赤でペイントするのがいかに簡単かをご覧くださいhttps://github.com/go-kit/kit/blob/master/log/term/example_test.go#L12. ただし、色付けは文字列全体にのみ適用できます:(

```
go-kit +: flexible, modular
go-kit “-”: logfmt format https://brandur.org/logfmt

logrus +: popular, feature rich (hooks), API and output is more like what we want
logrus -: not so flexible
```

```go
# tm_logger.go
//NewTmLogger returns a logger that encodes keyvals to the Writer in
//tm format.
func NewTmLogger(w io.Writer) Logger {
  return &tmLogger{kitlog.NewLogfmtLogger(w)}
}

func (l tmLogger) SetLevel(level string() {
  switch (level) {
    case "debug":
      l.sourceLogger = level.NewFilter(l.sourceLogger, level.AllowDebug())
  }
}

func (l tmLogger) Info(msg string, keyvals ...interface{}) error {
  l.sourceLogger.Log("msg", msg, keyvals...)
}

# log.go
func With(logger Logger, keyvals ...interface{}) Logger {
  kitlog.With(logger.sourceLogger, keyvals...)
}
```

Usage:

```go
logger := log.NewTmLogger(os.Stdout)
logger.SetLevel(config.GetString("log_level"))
node.SetLogger(log.With(logger, "node", Name))
```

**其他日志格式化程序**

将来，我们可能需要其他格式化程序，例如 JSONFormatter.

```
{ "level": "notice", "time": "2017-04-25 14:45:08.562471297 -0400 EDT", "module": "consensus", "msg": "ABCI Replay Blocks", "appHeight": 0, "storeHeight": 0, "stateHeight": 0 }
```

### 3) 动态记录器设置

https://dave.cheney.net/2017/01/23/the-package-level-logger-anti-pattern

这是最困难的部分，也是最多工作要做的地方. 如果将某些字段添加到记录器，记录器应与处理结构或上下文相关联.

```go
type BaseService struct {
    log     log15.Logger
    name    string
    started uint32//atomic
    stopped uint32//atomic
...
}
```

BaseServiceにはすでに `log`フィールドが含まれているため、BaseServiceに埋め込まれているほとんどの構造は問題ないはずです. 名前を `logger`に変更する必要があります.

足りないのは、ロガーを設定する機能だけです.

```
func (bs *BaseService) SetLogger(l log.Logger) {
  bs.logger = l
}
```

### 4)コンベンション

重要なキーバルブを最初に配置する必要があります. 例:

```
correct
I[2017-04-25|14:45:08.322] ABCI Replay Blocks                       module=consensus instance=1 appHeight=0 storeHeight=0 stateHeight=0
```

not

```
wrong
I[2017-04-25|14:45:08.322] ABCI Replay Blocks                       module=consensus appHeight=0 storeHeight=0 stateHeight=0 instance=1
```

このため、ほとんどの場合、特定のメッセージを記録するときではなく、ロガーを作成するときに「インスタンス」フィールドをロガーに追加する必要があります.

```go
colorFn := func(keyvals ...interface{}) term.FgBgColor {
		for i := 1; i < len(keyvals); i += 2 {
			if keyvals[i] == "instance" && keyvals[i+1] == "1" {
				return term.FgBgColor{Fg: term.Blue}
			} else if keyvals[i] == "instance" && keyvals[i+1] == "1" {
				return term.FgBgColor{Fg: term.Red}
			}
		}
		return term.FgBgColor{}
	}
logger := term.NewLogger(os.Stdout, log.NewTmLogger, colorFn)

c1 := NewConsensusReactor(...)
c1.SetLogger(log.With(logger, "instance", 1))

c2 := NewConsensusReactor(...)
c2.SetLogger(log.With(logger, "instance", 2))
```

## ステータス

実装

## 結果

### ポジティブ

一部のモジュールでは、実行時にダイナミックレコーダをオフにすることができます. Tendermintライブラリを使用する他のプロジェクトのパブリックインターフェイス.

### ネガティブ

キーと値のペアでキーに色を付ける機能が失われる可能性があります. go-kitを使用すると、文字列全体の前景色/背景色を簡単に変更できますが、文字列の一部は変更できません.

### ニュートラル

## 付録A.

私はgo-kitのロガーを使用する最小限の方法が本当に好きですhttps://github.com/go-kit/kit/tree/master/log:

```
type Logger interface {
    Log(keyvals ...interface{}) error
}
```

请参阅[寻找记录器接口](https://go-talks.appspot.com/github.com/ChrisHines/talks/structured-logging/structured-logging.slide). 优点是更大的可组合性(查看 go-kit 如何在此界面上定义彩色日志或日志级别的日志 https://github.com/go-kit/kit/tree/master/log).
