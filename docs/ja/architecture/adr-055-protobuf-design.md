# ADR 055:Protobufデザイン

## 変更ログ

-2020-4-15:作成(@ marbar3778)
-2020-6-18:更新(@ marbar3778)

## 環境

現在、Tendermintでは[go-amino](https://github.com/tendermint/go-amino)を使用しています. TendermintチームはAminoを保守しなくなり(2020年4月15日)、問題があることがわかりました.

-https://github.com/tendermint/go-amino/issues/286
-https://github.com/tendermint/go-amino/issues/230
-https://github.com/tendermint/go-amino/issues/121

これらは、ユーザーが遭遇する可能性のあるいくつかの既知の問題です.

アミノはラピッドプロトタイピングと機能開発をサポートします.これは良いことですが、Aminoは期待されるパフォーマンスと開発者の利便性を提供しません. TendermintをBFTプロトコルエンジンとして広く採用するには、採用されているエンコーディング形式に移行する必要があります.探索できるいくつかの可能なオプションがあります.

選択できるオプションはいくつかあります.

-`Protobuf`:プロトコルバッファはGoogleの言語に依存せず、プラットフォームに依存せず、拡張可能な構造化データのシリアル化メカニズムです.XMLについて考えてみてください.ただし、XMLはより小さく、より速く、よりシンプルです.それは無数の言語をサポートし、長年にわたって本番環境で証明されています.

-`FlatBuffers`:FlatBuffersは、効率的なクロスプラットフォームのシリアル化ライブラリです. Flatbuffersは、2番目の表現への解析/解凍の速度がないため、Protobufよりも効果的です. FlatBuffersはテストされ、本番環境で使用されていますが、広く採用されていません.

-`CapnProto`:Cap'n Protoは、非常に高速なデータ交換フォーマットであり、機能ベースのRPCシステムです. Cap'n Protoには、エンコード/デコードの手順はありません.まだ業界全体で広く採用されていません.

-@ erikgrinaker-https://github.com/tendermint/tendermint/pull/4623#discussion_r401163501
  「
  Cap'n'Protoは素晴らしいです.これは、元のProtobuf開発者の一人によって書かれ、問題のいくつかを修正し、たとえば、メモリにロードせずに多数のメッセージを処理するためのランダムアクセスをサポートし、決定論が必要な場合(たとえば、ステートマシン)をサポートします.非常に便利な(オプトイン)標準形.とはいえ、広く採用されているため、Protobufの方が適していると思いますが、Cap'n'Protoの方が技術的に優れているため、少し悲しくなります.
  「

## 決定

そのパフォーマンスとツールにより、TendermintからProtobufに移行します. Protobufの背後にあるエコシステムは非常に大きく、優れた[複数の言語のサポート](https://developers.google.com/protocol-buffers/docs/tutorials)を備えています.

これを実現するには、現在のタイプを現在の形式(手書き)で保持し、すべての `.proto`ファイルが保存される`/proto`ディレクトリを作成します.エンコーディングが必要な場合、ディスク上およびネットワーク上で、util関数を呼び出して、タイプを手書きのgoタイプからprotobufによって生成されたタイプに変換します.これは、[buf](https://buf.build)で推奨されているファイル構造に準拠しています.このファイルの構造の詳細については、[ここ](https://buf.build/docs/lint-checkers#file_layout)を参照してください.

この設計を採用することで、タイプの将来の変更をサポートし、よりモジュール化されたコードベースを可能にします.

## ステータス

実装

## 結果

### ポジティブ

-将来のモジュラータイプを許可する
-リファクタリングが少ない
-将来、元のドキュメントを仕様リポジトリにプルできるようにします.
- パフォーマンス
-複数の言語でのツールとサポート

### ネガティブ

-開発者がタイプを更新するときは、プロトタイプも更新する必要があります

### ニュートラル

## 参照する
