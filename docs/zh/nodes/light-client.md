# 配置轻客户端

Tendermint 自带了一个内置的 `tendermint light` 命令，可以使用
运行轻客户端代理服务器，验证 Tendermint RPC。 都这么叫
可以通过之前验证的证明追溯到区块头
将它们传回给调用者。 除此之外，它将呈现相同的
接口作为一个完整的 Tendermint 节点。

您可以通过运行 `tendermint light <chainID>` 来启动轻客户端代理服务器，
用各种标志来指定主节点，见证节点(交叉检查
主提供的信息)，可信头的散列和高度，
和更多。

例如:

```bash
$ tendermint light supernova -p tcp://233.123.0.140:26657 \
  -w tcp://179.63.29.15:26657,tcp://144.165.223.135:26657 \
  --height=10 --hash=37E9A6DD3FA25E83B22C18835401E8E56088D0D7ABC6FD99FCDC920DD76C1C57
```

如需其他选项，请运行 `tendermint light --help`。

## 在哪里获得可信的高度和哈希值

获得半可信散列和高度的一种方法是查询多个完整节点
并比较它们的哈希值:

```bash
$ curl -s https://233.123.0.140:26657:26657/commit | jq "{height: .result.signed_header.header.height, hash: .result.signed_header.commit.block_id.hash}"
{
  "height": "273",
  "hash": "188F4F36CBCD2C91B57509BBF231C777E79B52EE3E0D90D06B1A25EB16E6E23D"
}
```
