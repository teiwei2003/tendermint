# ADR 060:Go API 稳定性

## 变更日志

- 2020-09-08:初始版本。 (@erikgrinaker)

- 2020-09-09:调整接受的更改，添加初始公共 API 包，添加结果。 (@erikgrinaker)

- 2020 年 9 月 17 日:澄清初始公共 API。 (@erikgrinaker)

## 语境

随着 Tendermint 1.0 的发布，我们将采用 [语义版本控制](https://semver.org)。一个主要含义是保证在 Tendermint 2.0 之前我们不会进行向后不兼容的更改(预发布版本除外)。为了为我们的 Go API 提供这种保证，我们必须明确定义哪些 API 是公开的，哪些更改被认为是向后兼容的。

目前，我们在 [README](https://github.com/tendermint/tendermint#versioning) 中列出了我们认为公开的软件包，但由于我们仍处于 0.x 版本，因此我们根本不提供任何向后兼容性保证.

### 词汇表

* **外部项目:** 不同的 Git/VCS 存储库或代码库。

* **外部包:** 不同的 Go 包，可以是同一个项目中的子包或兄弟包。

* **内部代码:** 不适用于外部项目的代码。

* **内部目录:**`internal/`下的代码，不能导入到外部项目中。

* **Exported:** 以大写字母开头的 Go 标识符，因此可以被外部包访问。

* **Private:** 一个以小写字母开头的 Go 标识符，因此不能被外部包访问，除非通过导出的字段、变量或函数/方法返回值。

* **公共 API:** 任何可以由外部项目导入或访问的 Go 标识符，`_test.go` 文件中的测试代码除外。

* **私有 API:** 任何无法通过公共 API 访问的 Go 标识符，包括内部目录中的所有代码。

## 替代方法

- 将所有公共 API 拆分为单独的 Git 存储库中的单独 Go 模块，并将所有 Tendermint 代码考虑在内，完全不受 API 向后兼容性的限制。这被拒绝了，因为 Tendermint 项目之前已经尝试过，导致依赖管理开销过多。

- 简单地记录哪些 API 是公共的，哪些是私有的。这是当前的方法，但不应期望用户自行执行此方法，文档并不总是最新的，无论如何，外部项目通常最终取决于内部代码。

## 决定

从 Tendermint 1.0 开始，所有内部代码(私有 API 除外)都将放置在根级 [`internal` 目录](https://golang.org/cmd/go/#hdr-Internal_Directories) 中，Go 编译器将供外部项目使用的块。 `internal` 目录之外的所有导出项目都被视为公共 API，并受向后兼容性保证的约束，但以 `_test.go` 结尾的文件除外。

`crypto` 包可以在单独的 repo 中拆分为单独的模块。这是外部项目使用的主要通用包，并且是唯一的 Tendermint 依赖项，例如IAVL 可能会导致项目出现一些问题，具体取决于 IAVL 和 Tendermint。这将在进一步讨论后决定。

`tm-db` 包将在一个单独的 repo 中保持一个单独的模块。 `crypto` 包可能会被拆分，等待进一步讨论，因为这是其他项目使用的主要通用包。

## 详细设计

###公共API

在为 1.0 准备公共 API 时，我们应该牢记以下原则:

- 限制我们开始使用的公共 API 的数量 - 我们可以随时添加新的 API，但一旦公开 API，我们就无法更改或删除它们。

- 在 API 公开之前，对 API 进行彻底审查，以确保它满足未来的任何需求，可以适应预期的变化，并遵循良好的 API 设计实践。

以下是以某种形式包含在 1.0 中的最小公共 API 集:

- `abci`
- 用于构建节点`config`、`libs/log`和`version`的包
- 客户端 API，即`rpc/client`、`light` 和`privval`。
- `crypto`(可能作为一个单独的仓库)

在内部和与其他利益相关者进一步讨论之后，我们也可能提供额外的 API。然而，用于提供自定义组件(例如反应器和内存池)的公共 API 并未计划在 1.0 中使用，但如果我们想要提供的话，可能会在以后的 1.x 版本中添加。

为了比较，以下是 Cosmos SDK 中 Tendermint 导入的数量(不包括测试)，应该是计划中的 API 主要满足的。

```
      1 github.com/tendermint/tendermint/abci/server
     73 github.com/tendermint/tendermint/abci/types
      2 github.com/tendermint/tendermint/cmd/tendermint/commands
      7 github.com/tendermint/tendermint/config
     68 github.com/tendermint/tendermint/crypto
      1 github.com/tendermint/tendermint/crypto/armor
     10 github.com/tendermint/tendermint/crypto/ed25519
      2 github.com/tendermint/tendermint/crypto/encoding
      3 github.com/tendermint/tendermint/crypto/merkle
      3 github.com/tendermint/tendermint/crypto/sr25519
      8 github.com/tendermint/tendermint/crypto/tmhash
      1 github.com/tendermint/tendermint/crypto/xsalsa20symmetric
     11 github.com/tendermint/tendermint/libs/bytes
      2 github.com/tendermint/tendermint/libs/bytes.HexBytes
     15 github.com/tendermint/tendermint/libs/cli
      2 github.com/tendermint/tendermint/libs/cli/flags
      2 github.com/tendermint/tendermint/libs/json
     30 github.com/tendermint/tendermint/libs/log
      1 github.com/tendermint/tendermint/libs/math
     11 github.com/tendermint/tendermint/libs/os
      4 github.com/tendermint/tendermint/libs/rand
      1 github.com/tendermint/tendermint/libs/strings
      5 github.com/tendermint/tendermint/light
      1 github.com/tendermint/tendermint/internal/mempool
      3 github.com/tendermint/tendermint/node
      5 github.com/tendermint/tendermint/internal/p2p
      4 github.com/tendermint/tendermint/privval
     10 github.com/tendermint/tendermint/proto/tendermint/crypto
      1 github.com/tendermint/tendermint/proto/tendermint/libs/bits
     24 github.com/tendermint/tendermint/proto/tendermint/types
      3 github.com/tendermint/tendermint/proto/tendermint/version
      2 github.com/tendermint/tendermint/proxy
      3 github.com/tendermint/tendermint/rpc/client
      1 github.com/tendermint/tendermint/rpc/client/http
      2 github.com/tendermint/tendermint/rpc/client/local
      3 github.com/tendermint/tendermint/rpc/core/types
      1 github.com/tendermint/tendermint/rpc/jsonrpc/server
     33 github.com/tendermint/tendermint/types
      2 github.com/tendermint/tendermint/types/time
      1 github.com/tendermint/tendermint/version
```

### 向后兼容的变化

在 Go 中，[几乎所有 API 更改都是向后不兼容的](https://blog.golang.org/module-compatibility)，因此公共 API 中的导出项通常在 Tendermint 2.0 之前无法更改。我们可以对公共 API 进行的唯一向后兼容更改是:

- 添加一个包。

- 向包范围添加新标识符(例如 const、var、func、struct、interface 等)。

- 向结构添加新方法。

- 如果零值保留任何旧行为，则向结构添加新字段。

- 更改结构中字段的顺序。

- 如果函数类型本身在任何公共 API(例如回调)中不可分配，则向命名函数或结构方法添加可变参数。

- 向接口添加新方法，或向接口方法添加可变参数，_如果接口已经有私有方法_(防止外部包实现它)。

- 扩展数字类型，只要它是命名类型(例如，`type Number int32` 可以更改为 `int64`，但不能更改为 `int8` 或 `uint32`)。

请注意，公共 API 可以公开私有类型(例如，通过导出的变量、字段或函数/方法返回值)，在这种情况下，这些私有类型上的导出字段和方法也是公共 API 的一部分，并由其向后兼容性覆盖保证。通常，除非包装在导出的接口中，否则不应通过公共 API 访问私有类型。

另请注意，如果我们从依赖项接受、返回、导出或嵌入类型，我们将承担该依赖项的向后兼容性责任，并且必须确保任何依赖项升级都符合上述约束。

我们应该为次要版本分支运行 CI linters 以强制执行此操作，例如[apidiff](https://go.googlesource.com/exp/+/refs/heads/master/apidiff/README.md)、[breakcheck](https://github.com/gbbr/breakcheck) 和 [ apicombat](https://github.com/bradleyfalzon/apicompat)。

#### 接受破损

上述更改仍然可以通过几种方式破坏程序 - 这些_not_被认为是向后不兼容的更改，建议用户避免这种用法:

- 如果程序使用无键结构文字(例如`Foo{"bar", "baz"}`)并且我们添加字段或更改字段顺序，程序将不再编译或可能出现逻辑错误。

- 如果一个程序在一个结构体中嵌入了两个结构体，并且我们向一个嵌入的 Tendermint 结构体添加了一个新的字段或方法，该结构体也存在于另一个嵌入的结构体中，该程序将不再编译。

- 如果一个程序比较两个结构体(例如用`==`)，并且我们将一个不可比较类型(slice、map、func 或包含这些的结构体)的新字段添加到要比较的 Tendermint 结构体，程序将不再编译。

- 如果程序将 Tendermint 函数分配给标识符，并且我们向函数签名添加可变参数，则程序将不再编译。

### API 演进策略

上面的 API 保证可能相当严格，但鉴于 Go 语言设计，这是不可避免的。可以在适当的情况下使用以下技巧来允许我们对 API 进行更改:

- 我们可以添加一个具有不同名称的新函数或方法，该函数或方法带有额外的参数，并让旧函数调用新函数。

- 函数和方法可以采用选项结构而不是单独的参数，以允许添加新选项 - 这特别适用于采用许多参数并希望扩展的函数，尤其是对于我们无法添加具有不同参数的新方法的接口根本。

- 接口可以包括私有方法，例如`interface { private() }`，使它们无法被外部包实现，从而允许我们在不破坏其他程序的情况下向接口添加新方法。当然，这不能用于应该可以在外部实现的接口。

- 我们可以使用[接口升级](https://avtok.com/2014/11/05/interface-upgrades.html) 允许现有接口的实现者也实现新接口，只要旧接口可以仍然被使用 - 例如新接口`BetterReader` 可能有一个方法`ReadBetter()`，一个将`Reader` 接口作为输入的函数可以检查实现者是否也实现了`BetterReader`，在这种情况下调用`ReadBetter()` `读()`。

## 状态

公认

## 结果

### 积极的

- 用户可以安全地升级而不必担心应用程序损坏，并且知道升级是否仅包括错误修复或功能增强

- 外部开发人员有一个可预测且定义明确的 API 来构建，该 API 将在一段时间内得到支持

- 团队之间的同步性降低，因为变更的合同和时间表更清晰，而且发生的频率更低

- 更多文档将保持准确，因为它不会追逐移动目标

- 为社区和我们的团队，将花在代码改动上的时间更少，而将更多的时间花在功能改进上

### 消极的

- 许多改进、更改和错误修复将不得不推迟到下一个主要版本，可能会推迟一年或更长时间

- 开发速度会变慢，因为我们必须在现有的 API 约束内工作，并花更多时间规划公共 API

- 外部开发人员可能无法访问某些当前导出的 API 和功能

## 参考

- [#4451: 将内部 API 放在内部包下](https://github.com/tendermint/tendermint/issues/4451)

- [关于可插拔性](https://docs.google.com/document/d/1G08LnwSyb6BAuCVSMF3EKn47CGdhZ5wPZYJQr4-bw58/edit?ts=5f609f11)
