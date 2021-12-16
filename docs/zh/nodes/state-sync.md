# 配置状态同步

状态同步将在后台持续工作，以在引导时为节点提供分块数据。

> 注意:在尝试使用状态同步之前，请查看您操作节点的应用程序是否支持它。

在“config.toml”的状态同步部分下，您将找到多个需要配置的设置，以便您的节点使用状态同步。

让我们分解设置:

- `enable`:启用是通知节点您将使用状态同步来引导您的节点。
- `rpc_servers`:需要 RPC 服务器，因为状态同步利用轻客户端进行验证。
    - 需要 2 个服务器，更多总是有帮助的。
- `temp_dir`: 临时目录是在机器本地存储中存储块，如果没有设置它会在 `/tmp` 中创建一个目录

您需要通过公开暴露的 RPC 或您信任的区块浏览器获取下一个信息。

- `trust_height`:可信高度定义了你的节点应该信任链的高度。
- `trust_hash`:可信哈希是可信高度对应的`BlockID`中的哈希值。
- `trust_period`:信任周期是可以验证头部的周期。
  > :warning: 这个值应该明显小于解绑期。

如果你依赖公开暴露的 RPC 来获取需要的信息，你可以使用 `curl`。

例子:

```bash
curl -s https://233.123.0.140:26657/commit | jq "{height: .result.signed_header.header.height, hash: .result.signed_header.commit.block_id.hash}"
```

响应将是:

```json
{
  "height": "273",
  "hash": "188F4F36CBCD2C91B57509BBF231C777E79B52EE3E0D90D06B1A25EB16E6E23D"
}
```
