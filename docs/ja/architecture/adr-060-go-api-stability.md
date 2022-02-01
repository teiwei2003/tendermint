# ADR 060:GoAPIの安定性

## 変更ログ

-2020-09-08:初期バージョン. (@erikgrinaker)

-2020-09-09:受け入れられた変更を調整し、最初のパブリックAPIパッケージを追加して、結果を追加します. (@erikgrinaker)

-2020年9月17日:最初のパブリックAPIを明確にします. (@erikgrinaker)

## 環境

Tendermint 1.0のリリースに伴い、[Semantic Version Control](https://semver.org)を採用します.主な意味の1つは、Tendermint 2.0より前に下位互換性のない変更を行わないようにすることです(プレリリースバージョンを除く). Go APIにこの保証を提供するには、どのAPIが公開され、どの変更が下位互換性があると見なされるかを明確に定義する必要があります.

現在、公開されていると思われるパッケージを[README](https://github.com/tendermint/tendermint#versioning)にリストしていますが、まだバージョン0.xであるため、でのサポートは提供していません.すべて.互換性後の保証.

### 用語集

* **外部プロジェクト:**さまざまなGit/VCSリポジトリまたはコードベース.

* **外部パッケージ:**異なるGoパッケージは、同じプロジェクト内のサブパッケージまたは兄弟パッケージにすることができます.

* **内部コード:**外部プロジェクトには適用されないコード.

* **内部ディレクトリ:** `internal /`の下のコードを外部プロジェクトにインポートすることはできません.

* **エクスポート済み:**大文字で始まる識別子を使用して、外部パッケージからアクセスできるようにします.

* **プライベート:**小文字で始まるGo識別子.エクスポートされたフィールド、変数、または関数/メソッドを介して値が返されない限り、外部パッケージからアクセスすることはできません.

* **パブリックAPI:** `_test.go`ファイルのテストコードを除き、外部プロジェクトからインポートまたはアクセスできるすべてのGo識別子.

* **プライベートAPI:**内部カタログのすべてのコードを含む、パブリックAPIを介してアクセスできないすべてのGo識別子.

##代替方法

-すべてのパブリックAPIを別々のGitリポジトリ内の別々のGoモジュールに分割し、すべてのTendermintコードを考慮に入れて、APIの下位互換性の制限から完全に解放します. Tendermintプロジェクトが以前に試したことがあるため、これは拒否され、依存関係管理のオーバーヘッドが過剰になりました.

-パブリックなAPIとプライベートなAPIを記録するだけです.これは現在の方法ですが、ユーザーが自分でこの方法を実装することを期待するべきではありません.ドキュメントは常に最新であるとは限りません.いずれの場合も、外部プロジェクトは通常、最終的に内部コードに依存します.

## 決定

Tendermint 1.0以降、すべての内部コード(プライベートAPIを除く)はルートレベル[`internal`ディレクトリ](https://golang.org/cmd/go/#hdr-Internal_Directories)に配置され、Goコンパイラは外部プロジェクトで使用されるブロックを提供します. `_test.go`で終わるファイルを除いて、` internal`ディレクトリの外にエクスポートされたすべてのアイテムはパブリックAPIと見なされ、下位互換性が保証されます.

`crypto`パッケージは、個別のリポジトリ内の個別のモジュールに分割できます.これは、外部プロジェクトで使用される主要な共通パッケージであり、Tendermintの唯一の依存関係です.たとえば、IAVLとTendermintによっては、IAVLによってプロジェクトで問題が発生する場合があります.これは、さらなる議論の後に決定されます.

`tm-db`パッケージは、別のリポジトリに別のモジュールを維持します.これは他のプロジェクトで使用される主要な共通パッケージであるため、 `crypto`パッケージは分割され、さらなる議論を待つ可能性があります.

## 詳細設計

###パブリックAPI

1.0のパブリックAPIを準備するときは、次の原則に留意する必要があります.

-使用を開始する公開APIの数を制限します-いつでも新しいAPIを追加できますが、APIが公開されると、それらを変更または削除することはできません.

-APIを公開する前に、APIを徹底的にレビューして、将来のニーズを満たし、予想される変更に適応でき、優れたAPI設計慣行に従っていることを確認します.

以下は、何らかの形式で1.0に含まれているパブリックAPIの最小セットです.

-`abci`
-ノード `config`、` libs/log`、および `version`の構築に使用されるパッケージ
-クライアントAPI、つまり `rpc/client`、` light`、 `privval`.
-`crypto`(おそらく別のリポジトリとして)

社内および他の利害関係者とさらに話し合った後、追加のAPIを提供する場合もあります.ただし、カスタムコンポーネント(reactorやメモリプールなど)を提供するために使用されるパブリックAPIは、1.0で使用される予定はありませんが、提供したい場合は、将来の1.xバージョンで追加される可能性があります.

比較のために、以下はCosmos SDK(テストを除く)でのTendermintのインポート数です.これは、計画されたAPIの主な満足度であるはずです.
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

### 下位互換性の変更

Goでは、[ほとんどすべてのAPIの変更は下位互換性がありません](https://blog.golang.org/module-compatibility).したがって、パブリックAPIでのエクスポートは、通常、Tendermint2.0より前では変更できません.パブリックAPIに加えることができる下位互換性のある変更は次のとおりです.

-バッグを追加します.

-新しい識別子(const、var、func、struct、interfaceなど)をパッケージスコープに追加します.

-構造に新しいメソッドを追加します.

-ゼロ値が古い動作を保持している場合は、構造に新しいフィールドを追加します.

-構造内のフィールドの順序を変更します.

-関数型自体がパブリックAPI(コールバックなど)で割り当てできない場合は、名前付き関数または構造体メソッドに変数パラメーターを追加します.

-インターフェイスに新しいメソッドを追加するか、インターフェイスメソッドに変数パラメータを追加します(インターフェイスにすでにプライベートメソッドがある場合)(外部パッケージがそれを実装しないようにするため).

-名前付きタイプである限り、拡張数値タイプ(たとえば、 `type Numberint32`は` int64`に変更できますが、 `int8`または` uint32`には変更できません).

パブリックAPIはプライベートタイプを公開できることに注意してください(たとえば、エクスポートされた変数、フィールド、または関数/メソッドを介して戻り値).この場合、これらのプライベートタイプのエクスポートされたフィールドとメソッドもパブリックAPIの一部であり、それらによって制御されます.後方互換性カバレッジ保証.一般に、エクスポートされたインターフェイスでラップされていない限り、プライベートタイプはパブリックAPIを介してアクセスしないでください.

また、依存関係から型を受け入れる、返す、エクスポートする、または埋め込む場合、その依存関係の下位互換性について責任を負い、依存関係のアップグレードが上記の制約に準拠していることを確認する必要があることにも注意してください.

これを強制するには、マイナーバージョンブランチのCIリンターを実行する必要があります.たとえば、[apidiff](https://go.googlesource.com/exp/+/refs/heads/master/apidiff/README.md)、[breakcheck] (https://github.com/gbbr/breakcheck)および[apicombat](https://github.com/bradleyfalzon/apicompat).

#### 破損を受け入れる

上記の変更はまだいくつかの方法でプログラムを壊す可能性があります-これらの_not_は後方互換性のない変更と見なされ、ユーザーはこの使用を避けることをお勧めします:

-プログラムがキーレス構造化テキスト( `Foo {" bar "、" baz "}`など)を使用していて、フィールドを追加したり、フィールドの順序を変更したりすると、プログラムがコンパイルされなくなったり、論理エラーが発生したりする可能性があります.

-プログラムが構造体に2つの構造体を埋め込み、埋め込まれたTendermint構造体に新しいフィールドまたはメソッドを追加した場合、その構造体は別の埋め込み構造体にも存在します.プログラムはコンパイルされなくなります.

-プログラムが2つの構造(たとえば、 `==`)を比較し、比較できないタイプ(スライス、マップ、関数、またはこれらを含む構造)の新しいフィールドを比較するTendermint構造に追加すると、プログラムはコンパイルされなくなりました.

-プログラムがTendermint関数を識別子に割り当て、変数パラメーターを関数シグネチャに追加すると、プログラムはコンパイルされなくなります.

### API進化戦略

上記のAPIの保証は非常に厳しい場合がありますが、Go言語の設計を考えると、これは避けられません. APIに変更を加えることができるように、必要に応じて次の手法を使用できます.

-別の名前の新しい関数またはメソッド、追加のパラメーターを持つ関数またはメソッドを追加し、古い関数に新しい関数を呼び出させることができます.

-関数とメソッドは、個別のパラメーターの代わりにオプション構造を取り、新しいオプションを追加できます-これは、多くのパラメーターを取り、拡張したい関数、特に異なるパラメーターを持つ新しいメソッドを追加できないインターフェイスに特に適しています.

-インターフェイスには、 `interface {private()}`などのプライベートメソッドを含めることができるため、外部パッケージで実装することはできません.これにより、他のプログラムを壊すことなく、インターフェイスに新しいメソッドを追加できます.もちろん、これは外部に実装する必要のあるインターフェースには使用できません.

-[interface upgrade](https://avtok.com/2014/11/05/interface-upgrades.html)を使用して、古いインターフェースが引き続き使用できる限り、既存のインターフェースの実装者が新しいインターフェースも実装できるようにすることができます.たとえば、新しいインターフェイス `BetterReader`にはメソッド` ReadBetter() `が含まれている場合があります.これは、` Reader`インターフェイスを入力として受け取る関数で、実装者が `BetterReader`も実装しているかどうかを確認できます.この場合は` ReadBetter( ) ``読み取り() `.

## ステータス

受け入れられました

## 結果

### ポジティブ

-ユーザーは、アプリケーションの損傷を心配することなく安全にアップグレードでき、アップグレードにバグ修正または機能拡張のみが含まれるかどうかを知ることができます

-外部の開発者は、予測可能で明確に定義されたAPIを構築できます.これは、一定期間でサポートされます.

-変更された契約とスケジュールがより明確になり、発生頻度が低くなるため、チーム間の同期性が低下します

-移動するターゲットを追跡しないため、より多くのドキュメントが正確なままになります

-コミュニティと私たちのチームにとって、コードの変更に費やされる時間が減り、機能の改善に費やされる時間が増えます

### ネガティブ

-多くの改善、変更、バグ修正は次のメジャーバージョンに延期する必要があり、1年以上遅れる可能性があります

-既存のAPIの制約内で作業し、パブリックAPIの計画により多くの時間を費やす必要があるため、開発速度が低下します

-外部の開発者は、現在エクスポートされている一部のAPIおよび関数にアクセスできない場合があります

## 参照する

-[#4451:内部APIを内部パッケージに配置](https://github.com/tendermint/tendermint/issues/4451)

-[プラグイン可能性について](https://docs.google.com/document/d/1G08LnwSyb6BAuCVSMF3EKn47CGdhZ5wPZYJQr4-bw58/edit?ts=5f609f11)
