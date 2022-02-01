# ブロック構造

Tendermintコンセンサスエンジンはすべての合意を記録します
大多数のノードはブロックチェーンに入り、すべてのノード間で複製されます
ノード. ブロックチェーンには、主にさまざまなRPCエンドポイントを介してアクセスできます
`/block？height =`で完全なブロックを取得し、
`/blockchain？minHeight = _＆maxHeight = _`ヘッダーリストを取得します. しかし、何
これらのブロックに保存されていますか？

[仕様](https://github.com/tendermint/spec/blob/8dd2ed4c6fe12459edeb9b783bdaaaeb590ec15c/spec/core/data_structures.md)には、各コンポーネントの詳細な説明が含まれています.これは、開始するのに最適な場所です.

詳細については、[タイプパッケージドキュメント](https://godoc.org/github.com/tendermint/tendermint/types)を確認してください.
