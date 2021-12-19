# ライトクライアントを構成する

Tendermintには `tendermintlight`コマンドが組み込まれています。
ライトクライアントプロキシサーバーを実行して、TendermintRPCを確認します。 彼らは皆それを呼んでいます
以前に検証された証明を介してブロックヘッダーまでさかのぼることができます
それらを発信者に返します。 それ以外の場合は、同じように表示されます
インターフェイスは完全なTendermintノードとして機能します。

`tendermint light <chainID>`を実行すると、ライトクライアントプロキシサーバーを起動できます。
さまざまな記号を使用して、マスターノード、監視ノードを指定します(クロスチェック
マスターから提供された情報)、信頼できるヘッダーのハッシュと高さ、
もっと。

例えば:

```bash
$ tendermint light supernova -p tcp://233.123.0.140:26657 \
  -w tcp://179.63.29.15:26657,tcp://144.165.223.135:26657 \
  --height=10 --hash=37E9A6DD3FA25E83B22C18835401E8E56088D0D7ABC6FD99FCDC920DD76C1C57
```

その他のオプションについては、 `tendermint light--help`を実行してください。

## 信頼できる高さとハッシュ値を取得する場所

半信頼のハッシュと高さを取得する1つの方法は、複数の完全なノードをクエリすることです。
そして、それらのハッシュ値を比較します。

```bash
$ curl -s https://233.123.0.140:26657:26657/commit | jq "{height: .result.signed_header.header.height, hash: .result.signed_header.commit.block_id.hash}"
{
  "height": "273",
  "hash": "188F4F36CBCD2C91B57509BBF231C777E79B52EE3E0D90D06B1A25EB16E6E23D"
}
```
