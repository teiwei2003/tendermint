# Docs 构建工作流

Tendermint Core 的文档托管在:

- <https://docs.tendermint.com/>

从这个[`master`的`docs`目录中的文件构建](https://github.com/tendermint/tendermint/tree/master/docs)
和其他支持的发布分支。

## 怎么运行的

有一个 [GitHub Actions 工作流程](https://github.com/tendermint/docs/actions/workflows/deployment.yml)
在克隆和构建文档的 `tendermint/docs` 存储库中
来自这个`docs`目录的内容的站点，对于`master`和
每个受支持版本的 backport 分支。在后台，此工作流运行
`make build-docs` 来自 [Makefile](../Makefile#L214)。

支持的版本列表在 [`config.js`](./.vuepress/config.js) 中定义，
它定义了文档站点上的 UI 菜单，也在
[`docs/versions`](./versions)，它决定了构建哪些分支。

`docs/versions` 文件中的最后一个条目确定链接的版本
默认情况下来自生成的`index.html`。这通常应该是最
最近发布，而不是“master”，这样新用户就不会被
未发布功能的文档。

## 自述文件

[README.md](./README.md) 也是文档的登陆页面
在网站上。在 Jenkins 构建期间，当前提交被添加到底部
的自述文件。

## Config.js

[config.js](./.vuepress/config.js) 生成侧边栏和目录
在网站文档上。注意相对链接的使用和省略
文件扩展名。附加功能可用于改善外观
的侧边栏。

## 链接

**注意:** 强烈考虑现有链接 - 都在此目录中
和网站文档 - 移动或删除文件时。

目录的链接_必须_以`/`结尾。

相对链接应该在几乎所有地方使用，发现并权衡了以下内容:

### 相对的

相对于当前文件，另一个文件在哪里？

- 适用于 GitHub 和 VuePress 构建
- 令人困惑/烦人的事情是:`../../../../myfile.md`
- 重新混洗文件时需要更多更新

### 绝对

鉴于回购的根目录，另一个文件在哪里？

- 适用于 GitHub，不适用于 VuePress 构建
- 这更好:`/docs/hereitis/myfile.md`
- 如果你移动那个文件，里面的链接会被保留(当然不是它的链接)

### 满的

文件或目录的完整 GitHub URL。在有意义的时候偶尔使用
将用户发送到 GitHub。

## 本地构建

确保您在 `docs` 目录中并运行以下命令:

```bash
rm -rf node_modules
``

此命令将删除旧版本的视觉主题和所需的包。此步骤是可选的。

```bash
npm install
```

安装主题和所有依赖项。

```bash
npm run serve
```

<!-- markdown-link-check-disable -->

运行 `pre` 和 `post` 钩子并启动热重载网络服务器。请参阅此命令的输出以获取 URL(通常为 <https://localhost:8080>)。

<!-- markdown-link-check-enable -->

要将文档构建为静态网站，请运行 `npm run build`。你会在 `.vuepress/dist` 目录中找到该网站。

## 搜索

我们正在使用 [Algolia](https://www.algolia.com) 来支持全文搜索。这使用`config.js` 中的公共API 仅搜索键以及[tendermint.json](https://github.com/algolia/docsearch-configs/blob/master/configs/tendermint.json)我们可以用 PR 更新的配置文件。

## 一致性

因为构建过程是相同的(正如这里包含的信息)，这个文件应该保持同步，因为
尽可能使用它的 [Cosmos SDK 存储库中的对应物](https://github.com/cosmos/cosmos-sdk/blob/master/docs/DOCS_README.md)。
