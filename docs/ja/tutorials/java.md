# Javaでアプリケーションを作成する

## ガイド仮説

このガイドは、テンダーミントを使い始めたい初心者を対象としています.
ゼロからのコアアプリケーション.以前に持っていることを前提とはしていません
TendermintCoreの使用経験.

Tendermint Coreは、状態を採用するビザンチンフォールトトレラント(BFT)ミドルウェアです.
翻訳者(あなたのアプリケーション)-あらゆるプログラミング言語で書かれています-そして安全です
多くのマシンにコピーします.

このガイドに従うことで、Tendermintコアプロジェクトを作成します
kvstoreと呼ばれる、(非常に)単純な分散BFTキー値ストア.アプリケーション(
ブロックリンクインターフェース(ABCI)の実現はJavaで書かれます.

このガイドは、あなたがJVMの世界に精通していることを前提としています.初心者の場合は、[JVM最小サバイバルガイド](https://hadihariri.com/2013/12/29/jvm-minimal-survival-guide-for-the-dotnet-developer/#java-the -language -java-the-ecosystem-java-the-jvm)および[Gradle Docs](https://docs.gradle.org/current/userguide/userguide.html).

## 組み込みアプリケーションと外部アプリケーション

Golangを使用する場合は、アプリケーションとTendermintCoreを同じプロセスで実行して最大のパフォーマンスを得ることができます.
[Cosmos SDK](https://github.com/cosmos/cosmos-sdk)は次のように記述されています.
詳細については、[Goでの組み込みTendermint Coreアプリケーションの作成](./go-built-in.md)ガイドを参照してください.

このガイドで行ったように、別の言語を選択する場合は、別のアプリケーションを作成する必要があります.
ソケット(UNIXまたはTCP)またはgRPCを介してTendermintCoreと通信します.
このガイドでは、RPCサーバーを使用して外部アプリケーションを構築する方法を説明します.

別のアプリケーションを使用すると、セキュリティがより確実に保証される場合があります
プロセスは、確立されたバイナリプロトコルを介して通信します.肌の若返り
コアはアプリケーションの状態にアクセスできなくなります.

## 1.1JavaとGradleをインストールする

[OracleのJDKインストールガイド](https://www.oracle.com/technetwork/java/javase/downloads/index.html)を参照してください.

Javaが正常にインストールされたことを確認します.

```bash
$ java -version
java version "12.0.2" 2019-07-16
Java(TM) SE Runtime Environment (build 12.0.2+10)
Java HotSpot(TM) 64-Bit Server VM (build 12.0.2+10, mixed mode, sharing)
```

您可以选择任何高于或等于 8 的 Java 版本.
本指南使用 Java SE Development Kit 12 编写.

确保您设置了 `$JAVA_HOME` 环境变量:

```bash
$ echo $JAVA_HOME
/Library/Java/JavaVirtualMachines/jdk-12.0.2.jdk/Contents/Home
```

Gradleのインストールについては、[公式ガイド](https://gradle.org/install/)を参照してください.

## 1.2新しいJavaプロジェクトを作成する

まず、新しいGradleプロジェクトを作成します.

```bash
export KVSTORE_HOME=~/kvstore
mkdir $KVSTORE_HOME
cd $KVSTORE_HOME
```

サンプルディレクトリで実行します.

```bash
gradle init --dsl groovy --package io.example --project-name example --type java-application --test-framework junit
```

これにより、新しいプロジェクトが作成されます. ファイルツリーは次のようになります.

```bash
$ tree
.
|-- build.gradle
|-- gradle
|   `-- wrapper
|       |-- gradle-wrapper.jar
|       `-- gradle-wrapper.properties
|-- gradlew
|-- gradlew.bat
|-- settings.gradle
`-- src
    |-- main
    |   |-- java
    |   |   `-- io
    |   |       `-- example
    |   |           `-- App.java
    |   `-- resources
    `-- test
        |-- java
        |   `-- io
        |       `-- example
        |           `-- AppTest.java
        `-- resources
```

実行すると、「Helloworld」が出力されます. 標準出力へ.

```bash
$ ./gradlew run
> Task :run
Hello world.
```

## 1.3Tendermintコアアプリケーションの作成

Tendermint Coreは、アプリケーションを介してアプリケーションと通信します
ブロックリンクポート(ABCI). すべてのメッセージタイプは[protobuf
ファイル](https://github.com/tendermint/tendermint/blob/master/proto/tendermint/abci/types.proto).
これにより、TendermintCoreはプログラムで記述されたアプリケーションを実行できます.
言語.

### 1.3.1.protoファイルをコンパイルする

`build.gradle`の先頭に次のセクションを追加します.

```groovy
buildscript {
    repositories {
        mavenCentral()
    }
    dependencies {
        classpath 'com.google.protobuf:protobuf-gradle-plugin:0.8.8'
    }
}
```

build.gradleのプラグインセクションでprotobufプラグインを有効にします.

```groovy
plugins {
    id 'com.google.protobuf' version '0.8.8'
}
```

Add the following code to `build.gradle`:

```groovy
protobuf {
    protoc {
        artifact = "com.google.protobuf:protoc:3.7.1"
    }
    plugins {
        grpc {
            artifact = 'io.grpc:protoc-gen-grpc-java:1.22.1'
        }
    }
    generateProtoTasks {
        all()*.plugins {
            grpc {}
        }
    }
}
```

これで、 `* .proto`ファイルをコンパイルする準備が整いました.

必要な `.proto`ファイルをプロジェクトにコピーします.

```bash
mkdir -p \
  $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/abci \
  $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/version \
  $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/types \
  $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/crypto \
  $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/libs \
  $KVSTORE_HOME/src/main/proto/github.com/gogo/protobuf/gogoproto

cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/abci/types.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/abci/types.proto
cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/version/types.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/version/types.proto
cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/types/types.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/types/types.proto
cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/types/evidence.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/types/evidence.proto
cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/types/params.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/types/params.proto
cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/crypto/proof.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/crypto/proof.proto
cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/crypto/keys.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/crypto/keys.proto
cp $GOPATH/src/github.com/tendermint/tendermint/proto/tendermint/libs/types.proto \
   $KVSTORE_HOME/src/main/proto/github.com/tendermint/tendermint/proto/tendermint/libs/bits/types.proto
cp $GOPATH/src/github.com/gogo/protobuf/gogoproto/gogo.proto \
   $KVSTORE_HOME/src/main/proto/github.com/gogo/protobuf/gogoproto/gogo.proto
```

これらの依存関係を `build.gradle`に追加します.

```groovy
dependencies {
    implementation 'io.grpc:grpc-protobuf:1.22.1'
    implementation 'io.grpc:grpc-netty-shaded:1.22.1'
    implementation 'io.grpc:grpc-stub:1.22.1'
}
```

To generate all protobuf-type classes run:

```bash
./gradlew generateProto
```

すべてがうまくいったことを確認するために、 `build/generated/`ディレクトリをチェックすることができます.

```bash
$ tree build/generated/
build/generated/
|-- source
|   `-- proto
|       `-- main
|           |-- grpc
|           |   `-- types
|           |       `-- ABCIApplicationGrpc.java
|           `-- java
|               |-- com
|               |   `-- google
|               |       `-- protobuf
|               |           `-- GoGoProtos.java
|               |-- common
|               |   `-- Types.java
|               |-- proof
|               |   `-- Proof.java
|               `-- types
|                   `-- Types.java
```

### 1.3.2ABCIを実装する

生成された `$ KVSTORE_HOME/build/generated/source/proto/main/grpc/types/ABCIApplicationGrpc.java`ファイル
実装する必要のあるインターフェースである抽象クラス「ABCIApplicationImplBase」が含まれています.

次の内容の `$ KVSTORE_HOME/src/main/java/io/example/KVStoreApp.java`ファイルを作成します.

```java
package io.example;

import io.grpc.stub.StreamObserver;
import types.ABCIApplicationGrpc;
import types.Types.*;

class KVStoreApp extends ABCIApplicationGrpc.ABCIApplicationImplBase {

  //methods implementation

}
```

次に、ABCIApplicationImplBaseの各メソッドを介して呼び出され、追加されるタイミングについて説明します.
必要なビジネスロジック.

### 1.3.3 CheckTx

新しいトランザクションがTendermintCoreに追加されると、
それをチェックするためのアプリケーション(フォーマット、署名などを確認します).

```java
@Override
public void checkTx(RequestCheckTx req, StreamObserver<ResponseCheckTx> responseObserver) {
    var tx = req.getTx();
    int code = validate(tx);
    var resp = ResponseCheckTx.newBuilder()
            .setCode(code)
            .setGasWanted(1)
            .build();
    responseObserver.onNext(resp);
    responseObserver.onCompleted();
}

private int validate(ByteString tx) {
    List<byte[]> parts = split(tx, '=');
    if (parts.size() != 2) {
        return 1;
    }
    byte[] key = parts.get(0);
    byte[] value = parts.get(1);

  //check if the same key=value already exists
    var stored = getPersistedValue(key);
    if (stored != null && Arrays.equals(stored, value)) {
        return 2;
    }

    return 0;
}

private List<byte[]> split(ByteString tx, char separator) {
    var arr = tx.toByteArray();
    int i;
    for (i = 0; i < tx.size(); i++) {
        if (arr[i] == (byte)separator) {
            break;
        }
    }
    if (i == tx.size()) {
        return Collections.emptyList();
    }
    return List.of(
            tx.substring(0, i).toByteArray(),
            tx.substring(i + 1).toByteArray()
    );
}
```

これがまだコンパイルされていない場合でも、心配する必要はありません.

トランザクションの形式が `{bytes} = {bytes}`でない場合は、 `1`を返します.
コード. 同じkey = valueがすでに存在する場合(同じkeyとvalue)、 `2`を返します
コード. その他の場合は、有効であることを示すゼロコードを返します.

ゼロ以外のコードを含むコンテンツは無効と見なされることに注意してください( `-1`、` 100`、
など)テンダーミントコアによる.

有効なトランザクションは、大きすぎず、
十分なガス. 天然ガスの詳細については、["
仕様 "](https://docs.tendermint.com/master/spec/abci/apps.html#gas).

基になるKey-Valueストアには、
[JetBrains Xodus](https://github.com/JetBrains/xodus)、これはトランザクションモードなしでJavaで記述された組み込みの高性能データベースです.

`build.gradle`:

```groovy
dependencies {
    implementation 'org.jetbrains.xodus:xodus-environment:1.3.91'
}
```

```java
...
import jetbrains.exodus.ArrayByteIterable;
import jetbrains.exodus.ByteIterable;
import jetbrains.exodus.env.Environment;
import jetbrains.exodus.env.Store;
import jetbrains.exodus.env.StoreConfig;
import jetbrains.exodus.env.Transaction;

class KVStoreApp extends ABCIApplicationGrpc.ABCIApplicationImplBase {
    private Environment env;
    private Transaction txn = null;
    private Store store = null;

    KVStoreApp(Environment env) {
        this.env = env;
    }

    ...

    private byte[] getPersistedValue(byte[] k) {
        return env.computeInReadonlyTransaction(txn -> {
            var store = env.openStore("store", StoreConfig.WITHOUT_DUPLICATES, txn);
            ByteIterable byteIterable = store.get(txn, new ArrayByteIterable(k));
            if (byteIterable == null) {
                return null;
            }
            return byteIterable.getBytesUnsafe();
        });
    }
}
```

### 1.3.4 BeginBlock -> DeliverTx -> EndBlock -> Commit

Tendermint Coreがブロックを決定すると、ブロックはに転送されます
アプリケーションは3つの部分に分かれています: `BeginBlock`、トランザクションごとに1つの` DeliverTx`、
最後は `EndBlock`です. `DeliverTx`は非同期で送信していますが、
整然とした対応が期待されます.

```java
@Override
public void beginBlock(RequestBeginBlock req, StreamObserver<ResponseBeginBlock> responseObserver) {
    txn = env.beginTransaction();
    store = env.openStore("store", StoreConfig.WITHOUT_DUPLICATES, txn);
    var resp = ResponseBeginBlock.newBuilder().build();
    responseObserver.onNext(resp);
    responseObserver.onCompleted();
}
```

Here we begin a new transaction, which will accumulate the block's transactions and open the corresponding store.

```java
@Override
public void deliverTx(RequestDeliverTx req, StreamObserver<ResponseDeliverTx> responseObserver) {
    var tx = req.getTx();
    int code = validate(tx);
    if (code == 0) {
        List<byte[]> parts = split(tx, '=');
        var key = new ArrayByteIterable(parts.get(0));
        var value = new ArrayByteIterable(parts.get(1));
        store.put(txn, key, value);
    }
    var resp = ResponseDeliverTx.newBuilder()
            .setCode(code)
            .build();
    responseObserver.onNext(resp);
    responseObserver.onCompleted();
}
```

トランザクション形式が間違っているか、同じkey = valueがすでに存在する場合、
ゼロ以外のコードが再び返されます. それ以外の場合は、ストアに追加します.

現在の設計では、ブロックに誤ったトランザクションが含まれている可能性があります(これらのトランザクション
「CheckTx」は渡されましたが、「DeliverTx」に含まれるトランザクションまたは提案者が失敗しました
直接). これは、パフォーマンス上の理由から行われます.

この場合、 `DeliverTx`内でトランザクションをコミットできないことに注意してください
並行して呼び出すことができるクエリは、一貫性のないデータを返します(つまり、
実際のブロックが存在しない場合でも、特定の値がすでに存在していることが報告されます
まだ提出されていません).

`Commit`は、新しい状態を維持するようにアプリケーションに指示します.

```java
@Override
public void commit(RequestCommit req, StreamObserver<ResponseCommit> responseObserver) {
    txn.commit();
    var resp = ResponseCommit.newBuilder()
            .setData(ByteString.copyFrom(new byte[8]))
            .build();
    responseObserver.onNext(resp);
    responseObserver.onCompleted();
}
```

### 1.3.5クエリ

これで、クライアントが特定のキー/値がいつ存在するかを知りたい場合、
Tendermint Core RPC `/abci_query`エンドポイントを呼び出します.エンドポイントは次に呼び出します
アプリケーションの `Query`メソッド.

アプリケーションは独自のAPIを無料で提供できます. しかし、テンダーミントコアを使用することによって
プロキシとして、クライアント([ライトクライアントを含む
パッケージ](https://godoc.org/github.com/tendermint/tendermint/light))が利用可能
さまざまなアプリケーションにまたがる統合API. さらに、彼らは電話する必要はありません
それ以外の場合は、追加の証明のために別のTendermintコアAPIが使用されます.

ここには証拠が含まれていないことに注意してください.

```java
@Override
public void query(RequestQuery req, StreamObserver<ResponseQuery> responseObserver) {
    var k = req.getData().toByteArray();
    var v = getPersistedValue(k);
    var builder = ResponseQuery.newBuilder();
    if (v == null) {
        builder.setLog("does not exist");
    } else {
        builder.setLog("exists");
        builder.setKey(ByteString.copyFrom(k));
        builder.setValue(ByteString.copyFrom(v));
    }
    responseObserver.onNext(builder.build());
    responseObserver.onCompleted();
}
```

完全な仕様は見つけることができます
[こちら](https://docs.tendermint.com/master/spec/abci/).

## 1.4アプリケーションとTendermintCoreインスタンスを起動します

次のコードを `$ KVSTORE_HOME/src/main/java/io/example/App.java`ファイルに配置します.

```java
package io.example;

import jetbrains.exodus.env.Environment;
import jetbrains.exodus.env.Environments;

import java.io.IOException;

public class App {
    public static void main(String[] args) throws IOException, InterruptedException {
        try (Environment env = Environments.newInstance("tmp/storage")) {
            var app = new KVStoreApp(env);
            var server = new GrpcServer(app, 26658);
            server.start();
            server.blockUntilShutdown();
        }
    }
}
```

これは、アプリケーションのエントリポイントです.
ここでは、アプリケーションの状態を格納する場所を認識する特別なオブジェクト「Environment」を作成しました.
次に、GRPCサーバーを作成して起動し、TendermintCoreリクエストを処理します.

次の内容の `$ KVSTORE_HOME/src/main/java/io/example/GrpcServer.java`ファイルを作成します.
```java
package io.example;

import io.grpc.BindableService;
import io.grpc.Server;
import io.grpc.ServerBuilder;

import java.io.IOException;

class GrpcServer {
    private Server server;

    GrpcServer(BindableService service, int port) {
        this.server = ServerBuilder.forPort(port)
                .addService(service)
                .build();
    }

    void start() throws IOException {
        server.start();
        System.out.println("gRPC server started, listening on $port");
        Runtime.getRuntime().addShutdownHook(new Thread(() -> {
            System.out.println("shutting down gRPC server since JVM is shutting down");
            GrpcServer.this.stop();
            System.out.println("server shut down");
        }));
    }

    private void stop() {
        server.shutdown();
    }

  /**
     * Await termination on the main thread since the grpc library uses daemon threads.
     */
    void blockUntilShutdown() throws InterruptedException {
        server.awaitTermination();
    }
}
```

## 1.5起動して実行

デフォルト構成、nodeKey、およびプライベートバリデーターファイルを作成するには、
`tendermintinit`を実行します. ただし、その前に、インストールする必要があります
テンダーミントコア.

```bash
$ rm -rf/tmp/example
$ cd $GOPATH/src/github.com/tendermint/tendermint
$ make install
$ TMHOME="/tmp/example" tendermint init validator

I[2019-07-16|18:20:36.480] Generated private validator                  module=main keyFile=/tmp/example/config/priv_validator_key.json stateFile=/tmp/example2/data/priv_validator_state.json
I[2019-07-16|18:20:36.481] Generated node key                           module=main path=/tmp/example/config/node_key.json
I[2019-07-16|18:20:36.482] Generated genesis file                       module=main path=/tmp/example/config/genesis.json
I[2019-07-16|18:20:36.483] Generated config                             module=main mode=validator
```

Feel free to explore the generated files, which can be found at
`/tmp/example/config` directory. Documentation on the config can be found
[here](https://docs.tendermint.com/master/tendermint-core/configuration.html).

We are ready to start our application:

```bash
./gradlew run

gRPC server started, listening on 26658
```

次に、Tendermint Coreを起動して、アプリケーションをポイントする必要があります. 止まる
アプリケーションディレクトリで実行します.

```bash
$ TMHOME="/tmp/example" tendermint node --abci grpc --proxy-app tcp://127.0.0.1:26658

I[2019-07-28|15:44:53.632] Version info                                 module=main software=0.32.1 block=10 p2p=7
I[2019-07-28|15:44:53.677] Starting Node                                module=main impl=Node
I[2019-07-28|15:44:53.681] Started node                                 module=main nodeInfo="{ProtocolVersion:{P2P:7 Block:10 App:0} ID_:7639e2841ccd47d5ae0f5aad3011b14049d3f452 ListenAddr:tcp://0.0.0.0:26656 Network:test-chain-Nhl3zk Version:0.32.1 Channels:4020212223303800 Moniker:Ivans-MacBook-Pro.local Other:{TxIndex:on RPCAddress:tcp://127.0.0.1:26657}}"
I[2019-07-28|15:44:54.801] Executed block                               module=state height=8 validTxs=0 invalidTxs=0
I[2019-07-28|15:44:54.814] Committed state                              module=state height=8 txs=0 appHash=0000000000000000
```

Now open another tab in your terminal and try sending a transaction:

```bash
$ curl -s 'localhost:26657/broadcast_tx_commit?tx="tendermint=rocks"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "check_tx": {
      "gasWanted": "1"
    },
    "deliver_tx": {},
    "hash": "CDD3C6DFA0A08CAEDF546F9938A2EEC232209C24AA0E4201194E0AFB78A2C2BB",
    "height": "33"
}
```

Response should contain the height where this transaction was committed.

Now let's check if the given key now exists and its value:

```bash
$ curl -s 'localhost:26657/abci_query?data="tendermint"'
{
  "jsonrpc": "2.0",
  "id": "",
  "result": {
    "response": {
      "log": "exists",
      "key": "dGVuZGVybWludA==",
      "value": "cm9ja3My"
    }
  }
}
```

`dGVuZGVybWludA==` 和 `cm9ja3M=` 分别是 `tendermint` 和 `rocks` 的 ASCII 码的 base64 编码.

## 終わり

私はすべてがうまくいくことを願っています、あなたの最初ですが、最後ではないことを願っています、
TendermintCoreアプリケーションが稼働しています. そうでない場合は、[質問を開いてください
Github](https://github.com/tendermint/tendermint/issues/new/choose). 掘る
[ドキュメント](https://docs.tendermint.com/master/)を詳しく読んでください.

このサンプルプロジェクトの完全なソースコードは[ここ](https://github.com/climber73/tendermint-abci-grpc-java)にあります.
