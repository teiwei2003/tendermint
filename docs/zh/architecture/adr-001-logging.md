# ADR 1:记录

## 语境

Tendermint 中当前的日志系统非常静态且不够灵活。

问题:[358](https://github.com/tendermint/tendermint/issues/358)、[375](https://github.com/tendermint/tendermint/issues/375)。

我们希望从新系统中得到什么:

- 每个包动态日志级别
- 动态记录器设置(记录器绑定到处理结构)
- 公约
- 更具视觉吸引力

“动态”在这里是指在运行时设置 smth 的能力。

## 决定

### 1) 一个接口

首先，我们需要一个用于所有库(`tmlibs`、Tendermint 等)的接口。 我个人的偏好是 go-kit `Logger` 界面(见附录 A)，但这是一个太大的变化。 另外，我们仍然需要水平。

```go
# log.go
type Logger interface {
    Debug(msg string, keyvals ...interface{}) error
    Info(msg string, keyvals ...interface{}) error
    Error(msg string, keyvals ...interface{}) error

	  With(keyvals ...interface{}) Logger
}
```

附带说明:“Info”和“Notice”之间的区别很微妙。 我们大概
可以没有“通知”。 不要认为我们需要“恐慌”或“致命”作为其中的一部分
界面。 这些函数可以作为助手来实现。 其实我们已经
在`tmlibs/common`中有一些。

- `Debug` - 开发人员的扩展输出
- `Info` - 所有对用户有用的信息
- `错误` - 错误

`Notice` 应该变成 `Info`、`Warn` 或者 `Error` 或 `Debug`，具体取决于消息，`Crit` -> `Error`。

这个接口应该进入`tmlibs/log`。 作为核心(tendermint/tendermint)一部分的所有库都应该遵守它。

### 2) 使用我们当前格式的记录器

在这个接口之上，我们需要实现一个标准输出记录器，当 Tendermint 被配置为将日志输出到标准输出时将使用它。

很多人说他们喜欢当前的输出，所以让我们坚持下去。

```
NOTE[2017-04-25|14:45:08] ABCI Replay Blocks                       module=consensus appHeight=0 storeHeight=0 stateHeight=0
```

几个小改动:

```
I[2017-04-25|14:45:08.322] ABCI Replay Blocks            module=consensus appHeight=0 storeHeight=0 stateHeight=0
```

请注意，级别仅使用一个字符加毫秒进行编码。

注意:还有许多其他格式，如 [logfmt](https://brandur.org/logfmt)。

这个记录器可以使用任何记录器来实现 - [logrus](https://github.com/sirupsen/logrus), [go-kit/log](https://github.com/go-kit/kit/tree/ master/log), [zap](https://github.com/uber-go/zap), log15 到此为止

a) 支持着色输出<br>
b) 速度适中(缓冲)<br>
c) 符合新的接口或可以为其编写适配器 <br>
d) 有点可配置<br>

到目前为止，go-kit 是我最喜欢的。 看看将错误涂成红色是多么容易 https://github.com/go-kit/kit/blob/master/log/term/example_test.go#L12。 虽然，着色只能应用于整个字符串:(

```
go-kit +: flexible, modular
go-kit “-”: logfmt format https://brandur.org/logfmt

logrus +: popular, feature rich (hooks), API and output is more like what we want
logrus -: not so flexible
```

```go
# tm_logger.go
// NewTmLogger returns a logger that encodes keyvals to the Writer in
// tm format.
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

将来，我们可能需要其他格式化程序，例如 JSONFormatter。

```
{ "level": "notice", "time": "2017-04-25 14:45:08.562471297 -0400 EDT", "module": "consensus", "msg": "ABCI Replay Blocks", "appHeight": 0, "storeHeight": 0, "stateHeight": 0 }
```

### 3) 动态记录器设置

https://dave.cheney.net/2017/01/23/the-package-level-logger-anti-pattern

这是最困难的部分，也是最多工作要做的地方。 如果将某些字段添加到记录器，记录器应与处理结构或上下文相关联。

```go
type BaseService struct {
    log     log15.Logger
    name    string
    started uint32 // atomic
    stopped uint32 // atomic
...
}
```

BaseService 已经包含 `log` 字段，所以大多数嵌入它的结构应该没问题。 我们应该将它重命名为`logger`。

唯一缺少的是设置记录器的能力:

```
func (bs *BaseService) SetLogger(l log.Logger) {
  bs.logger = l
}
```

### 4) 约定

重要的 keyvals 应该先行。 例子:

```
correct
I[2017-04-25|14:45:08.322] ABCI Replay Blocks                       module=consensus instance=1 appHeight=0 storeHeight=0 stateHeight=0
```

not

```
wrong
I[2017-04-25|14:45:08.322] ABCI Replay Blocks                       module=consensus appHeight=0 storeHeight=0 stateHeight=0 instance=1
```

为此，在大多数情况下，您需要在创建时将 `instance` 字段添加到记录器，而不是在您记录特定消息时:

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

## 状态

实施的

## 结果

### 积极的

动态记录器，可以在运行时为某些模块关闭。 使用 Tendermint 库的其他项目的公共接口。

### 消极的

我们可能会失去为键值对中的键着色的能力。 go-kit 允许您轻松更改整个字符串的前景色/背景色，但不能更改其部分。

### 中性的

## 附录 A。

我真的很喜欢 go-kit 使用他的记录器 https://github.com/go-kit/kit/tree/master/log 的简约方法:

```
type Logger interface {
    Log(keyvals ...interface{}) error
}
```

请参阅[寻找记录器接口](https://go-talks.appspot.com/github.com/ChrisHines/talks/structured-logging/structured-logging.slide)。 优点是更大的可组合性(查看 go-kit 如何在此界面上定义彩色日志或日志级别的日志 https://github.com/go-kit/kit/tree/master/log)。
