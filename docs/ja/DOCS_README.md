# ドキュメントビルドワークフロー

Tendermint Coreのドキュメントは、次の場所でホストされています.

-<https://docs.tendermint.com/>

この[`master`の` docs`ディレクトリにあるドキュメント](https://github.com/tendermint/tendermint/tree/master/docs)からビルドします
およびその他のサポートされているリリースブランチ.

## 使い方

[GitHubアクションワークフロー](https://github.com/tendermint/docs/actions/workflows/deployment.yml)があります
ドキュメントが複製されて構築される `tendermint/docs`リポジトリ内
この `docs`ディレクトリのコンテンツのサイト.`master`と
サポートされている各バージョンのバックポートブランチ.バックグラウンドで、このワークフローは実行されます
`make build-docs`は[Makefile](../Makefile#L214)から来ています.

サポートされているバージョンのリストは、[`config.js`](./.vuepress/config.js)で定義されています.
ドキュメントサイトのUIメニューを定義します.
[`docs/version`](./versions)、構築するブランチを決定します.

`docs/versions`ファイルの最後のエントリは、リンクのバージョンを決定します
これは、デフォルトで生成された `index.html`から取得されます.これは通常最も多いはずです
「マスター」の代わりに最近リリースされたため、新しいユーザーは
未公開の機能のドキュメント.

## Readmeファイル

[README.md](./README.md)は、ドキュメントのランディングページでもあります
ウェブサイト上. Jenkinsのビルド中に、現在のコミットが下部に追加されます
READMEファイル.

## Config.js

[config.js](./.vuepress/config.js)サイドバーと目次を生成します
Webサイトのドキュメント.相対リンクの使用と省略に注意してください
ファイル拡張子.追加機能を使用して外観を改善できます
サイドバー.

## リンク

**注:**既存のリンクは強く考慮されます-すべてこのディレクトリにあります
そしてウェブサイトのドキュメント-ファイルを移動または削除するとき.

ディレクトリへのリンクは「/」で終わる必要があります.

相対リンクはほとんどすべての場所で使用する必要があり、次のものが見つかり、評価されました.

### 比較的

現在のファイルに対して、他のファイルはどこにありますか？

-GitHubおよびVuePressビルドに適しています
-紛らわしい/迷惑なことは次のとおりです: `../../../../myfile.md`
-ファイルを再シャッフルするときは、さらに更新が必要です

### 絶対

リポジトリのルートディレクトリを考えると、他のファイルはどこにありますか？

-VuePressビルドではなく、GitHubに適用されます
-これはより良いです: `/docs/hereitis/myfile.md`
-そのファイルを移動すると、内部のリンクは保持されます(もちろんそのリンクではありません)

### 満杯

ファイルまたはディレクトリの完全なGitHubURL.意味があるときに時々使用する
ユーザーをGitHubに送信します.

## ローカルビルド

`docs`ディレクトリにいることを確認し、次のコマンドを実行します.

```bash
rm -rf node_modules
```

このコマンドは、古いバージョンのビジュアルテーマと必要なパッケージを削除します.このステップはオプションです.

```bash
npm install
```

テーマとすべての依存関係をインストールします.

```bash
npm runserve
```

<！-markdown-link-check-disable->

`pre`フックと` post`フックを実行し、ホットリロードWebサーバーを起動します. URL(通常は<https://localhost:8080>)については、このコマンドの出力を参照してください.

<！-markdown-link-check-enable->

ドキュメントを静的Webサイトとしてビルドするには、 `npm runbuild`を実行します. Webサイトは `.vuepress/dist`ディレクトリにあります.

## 探す

全文検索をサポートするために[Algolia](https://www.algolia.com)を使用しています.これは、 `config.js`のパブリックAPIを使用して、キーと[tendermint.json](https://github.com/algolia/docsearch-configs/blob/master/configs/tendermint.json)のみを検索します.更新できますPR構成ファイルを使用します.

## 一貫性

ビルドプロセスは(ここに含まれる情報と)同じであるため、このファイルは同期を維持する必要があります.
可能な限り、[Cosmos SDKリポジトリに対応する](https://github.com/cosmos/cosmos-sdk/blob/master/docs/DOCS_README.md)を使用してください./
