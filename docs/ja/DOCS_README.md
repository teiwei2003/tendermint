# ドキュメントビルドワークフロー

Tendermint Coreのドキュメントは、次の場所でホストされています。

-<https://docs.tendermint.com/>

この[`master`の` docs`ディレクトリ内のファイルから構築](https://github.com/tendermint/tendermint/tree/master/docs)
およびその他のサポートされているリリースブランチ。

## 使い方

[GitHubアクションワークフロー](https://github.com/tendermint/docs/actions/workflows/deployment.yml)があります
ドキュメントのクローンを作成してビルドする `tendermint/docs`リポジトリ内
この `docs`ディレクトリの内容からのサイト、` master`、および
サポートされている各リリースのバックポートブランチ。内部では、このワークフローが実行されます
[Makefile](../Makefile#L214)から `makebuild-docs`。

サポートされているバージョンのリストは、[`config.js`](./。vuepress/config.js)で定義されています。
これは、ドキュメントサイトとでUIメニューを定義します
[`docs/version`](./versions)、これはどのブランチを構築するかを決定します。

`docs/versions`ファイルの最後のエントリは、リンクされているバージョンを決定します
デフォルトでは、生成された `index.html`から。これは一般的に最も多いはずです
「マスター」ではなく最近のリリースで、新しいユーザーが
未リリースの機能に関するドキュメント。

## README

[README.md](./README.md)は、ドキュメントのランディングページでもあります
ウェブサイトで。 Jenkinsのビルド中に、現在のコミットが下部に追加されます
READMEの。

## Config.js

[config.js](./。vuepress/config.js)は、サイドバーと目次を生成します
ウェブサイトのドキュメントで。相対リンクの使用との省略に注意してください
ファイル拡張子。外観を改善するための追加機能を利用できます
サイドバーの。

## リンク

**注:**既存のリンクを強く検討してください-両方ともこのディレクトリ内にあります
およびWebサイトのドキュメントへ-ファイルを移動または削除する場合。

ディレクトリへのリンク_MUST_は `/`で終わります。

相対リンクは、次のことを発見して評価した上で、ほぼすべての場所で使用する必要があります。

### 相対的

現在のファイルと比較して、他のファイルはどこにありますか？

-GitHubとVuePressビルドの両方で機能します
-混乱する/煩わしい: `../../../../myfile.md`
-ファイルが再シャッフルされるときに、より多くの更新が必要です

### 絶対

リポジトリのルートを指定すると、他のファイルはどこにありますか？

-GitHubで動作し、VuePressビルドでは動作しません
-これははるかに優れています: `/docs/hereitis/myfile.md`
-そのファイルを移動すると、その中のリンクは保持されます(もちろん、ファイルへのリンクは保持されません)。

### 満杯

ファイルまたはディレクトリへの完全なGitHubURL。意味があるときに時々使用されます
ユーザーをGitHubに送信します。

##ローカルで構築する

`docs`ディレクトリにいることを確認し、次のコマンドを実行します。

```bash
rm -rf node_modules
```

このコマンドは、古いバージョンのビジュアルテーマと必要なパッケージを削除します。このステップはオプションです。

```bash
npm install
```

テーマとすべての依存関係をインストールします。

```bash
npm run serve
```

<！-markdown-link-check-disable->

`pre`フックと` post`フックを実行し、ホットリロードWebサーバーを起動します。 URLについては、このコマンドの出力を参照してください(多くの場合、<https://localhost:8080>です)。

<！-markdown-link-check-enable->

ドキュメントを静的Webサイトとしてビルドするには、 `npm runbuild`を実行します。 Webサイトは `.vuepress/dist`ディレクトリにあります。

## 検索

全文検索を強化するために[Algolia](https://www.algolia.com)を使用しています。これは、 `config.js`のパブリックAPI検索専用キーと[tendermint.json](https://github.com/algolia/docsearch-configs/blob/master/configs/tendermint.json)を使用しますPRで更新できる構成ファイル。

## 一貫性

ビルドプロセスは(ここに含まれる情報と同様に)同一であるため、このファイルは次のように同期を維持する必要があります。
[Cosmos SDKリポジトリのカウンターパート](https://github.com/cosmos/cosmos-sdk/blob/master/docs/DOCS_README.md)で可能な限り。
