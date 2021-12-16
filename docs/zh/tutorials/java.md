# 在Java中创建一个应用程序

## 指南假设

本指南专为想要开始使用 Tendermint 的初学者而设计
从头开始的核心应用程序。它并不假设您有任何先前
使用 Tendermint Core 的经验。

Tendermint Core 是采用状态的拜占庭容错 (BFT) 中间件
转换机(您的应用程序) - 用任何编程语言编写 - 并且安全
在许多机器上复制它。

通过遵循本指南，您将创建一个 Tendermint 核心项目
称为 kvstore，一个(非常)简单的分布式 BFT 键值存储。应用程序(应该
实现区块链接口(ABCI))将用Java编写。

本指南假设您对 JVM 世界并不陌生。如果您是新手，请参阅 [JVM 最小生存指南](https://hadihariri.com/2013/12/29/jvm-minimal-survival-guide-for-the-dotnet-developer/#java-the-language- java-the-ecosystem-java-the-jvm) 和 [Gradle Docs](https://docs.gradle.org/current/userguide/userguide.html)。

## 内置应用与外部应用

如果您使用 Golang，您可以在同一进程中运行您的应用程序和 Tendermint Core 以获得最大性能。
[Cosmos SDK](https://github.com/cosmos/cosmos-sdk) 就是这样写的。
详情请参考[Writing a built-in Tendermint Core application in Go](./go-built-in.md) 指南。

如果您选择另一种语言，就像我们在本指南中所做的那样，您必须编写一个单独的应用程序，
它将通过套接字(UNIX 或 TCP)或 gRPC 与 Tendermint Core 通信。
本指南将向您展示如何使用 RPC 服务器构建外部应用程序。

拥有一个单独的应用程序可能会给你更好的安全保证
进程将通过已建立的二进制协议进行通信。嫩肤
核心将无法访问应用程序的状态。

## 1.1 安装 Java 和 Gradle

请参考【Oracle的JDK安装指南】(https://www.oracle.com/technetwork/java/javase/downloads/index.html)。

验证您是否已成功安装 Java:

```bash
$ java -version
java version "12.0.2" 2019-07-16
Java(TM) SE Runtime Environment (build 12.0.2+10)
Java HotSpot(TM) 64-Bit Server VM (build 12.0.2+10, mixed mode, sharing)
```

您可以选择任何高于或等于 8 的 Java 版本。
本指南使用 Java SE Development Kit 12 编写。

确保您设置了 `$JAVA_HOME` 环境变量:

```bash
$ echo $JAVA_HOME
/Library/Java/JavaVirtualMachines/jdk-12.0.2.jdk/Contents/Home
```

Gradle 安装请参考[他们的官方指南](https://gradle.org/install/)。

## 1.2 创建一个新的 Java 项目

我们将首先创建一个新的 Gradle 项目。

```bash
export KVSTORE_HOME=~/kvstore
mkdir $KVSTORE_HOME
cd $KVSTORE_HOME
```

在示例目录中运行:

```bash
gradle init --dsl groovy --package io.example --project-name example --type java-application --test-framework junit
```

这将为您创建一个新项目。 文件树应如下所示:

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

运行时，这应该打印“Hello world”。 到标准输出。

```bash
$ ./gradlew run
> Task :run
Hello world.
```

## 1.3 编写 Tendermint Core 应用程序

Tendermint Core 通过应用程序与应用程序通信
区块链接口(ABCI)。 所有消息类型都在 [protobuf
文件](https://github.com/tendermint/tendermint/blob/master/proto/tendermint/abci/types.proto)。
这允许 Tendermint Core 运行以任何编程方式编写的应用程序
语。

### 1.3.1 编译 .proto 文件

将以下部分添加到 `build.gradle` 的顶部:

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

在 build.gradle 的 plugins 部分启用 protobuf 插件:

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

现在我们应该准备好编译 `*.proto` 文件了。

将必要的 `.proto` 文件复制到您的项目中:

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

将这些依赖项添加到 `build.gradle`:

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

要验证一切顺利，您可以检查 `build/generated/` 目录:

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

### 1.3.2 实现 ABCI

生成的`$KVSTORE_HOME/build/generated/source/proto/main/grpc/types/ABCIApplicationGrpc.java`文件
包含抽象类“ABCIApplicationImplBase”，这是我们需要实现的接口。

使用以下内容创建 `$KVSTORE_HOME/src/main/java/io/example/KVStoreApp.java` 文件:

```java
package io.example;

import io.grpc.stub.StreamObserver;
import types.ABCIApplicationGrpc;
import types.Types.*;

class KVStoreApp extends ABCIApplicationGrpc.ABCIApplicationImplBase {

    // methods implementation

}
```

现在我将通过 ABCIApplicationImplBase 的每个方法解释它何时被调用并添加
所需的业务逻辑。

### 1.3.3 CheckTx

当一个新的交易被添加到 Tendermint Core 时，它会询问
应用程序来检查它(验证格式、签名等)。

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

    // check if the same key=value already exists
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

如果这还没有编译，请不要担心。

如果交易没有`{bytes}={bytes}`的形式，我们返回`1`
代码。 当相同的 key=value 已经存在(相同的 key 和 value)时，我们返回 `2`
代码。 对于其他人，我们返回一个零代码，表明它们是有效的。

请注意，任何具有非零代码的内容都将被视为无效(`-1`、`100`、
等)由 Tendermint 核心。

有效的交易最终将被提交，因为它们不是太大并且
有足够的气。 要了解有关天然气的更多信息，请查看 [“
规范"](https://docs.tendermint.com/master/spec/abci/apps.html#gas)。

对于我们将使用的底层键值存储
[JetBrains Xodus](https://github.com/JetBrains/xodus)，这是一个用Java编写的无事务模式的嵌入式高性能数据库。

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

当 Tendermint Core 决定区块时，它会被转移到
应用程序分为 3 个部分:`BeginBlock`，每笔交易一个 `DeliverTx` 和
最后是`EndBlock`。 `DeliverTx` 正在异步传输，但是
预计响应将有序进行。

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

如果交易格式错误或相同的 key=value 已经存在，我们
再次返回非零代码。 否则，我们将其添加到商店。

在当前的设计中，一个区块可能包含不正确的交易(那些
通过“CheckTx”，但“DeliverTx”或提议者包含的交易失败
直接地)。 这样做是出于性能原因。

请注意，我们不能在 `DeliverTx` 内提交事务，因为在这种情况下
可以并行调用的 `Query` 将返回不一致的数据(即
即使实际块不存在，它也会报告某些值已经存在
尚未提交)。

`Commit` 指示应用程序保持新状态。

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

### 1.3.5 查询

现在，当客户端想知道特定键/值何时存在时，它
将调用 Tendermint Core RPC `/abci_query` 端点，后者又会调用
应用程序的`Query` 方法。

应用程序可以免费提供自己的 API。 但是通过使用 Tendermint Core
作为代理，客户端(包括[轻客户端
包](https://godoc.org/github.com/tendermint/tendermint/light)) 可以利用
跨不同应用程序的统一 API。 此外，他们将不必致电
否则单独的 Tendermint 核心 API 用于额外的证明。

请注意，我们在此处不包含证明。

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

可以找到完整的规范
[此处](https://docs.tendermint.com/master/spec/abci/)。

## 1.4 启动应用程序和 Tendermint Core 实例

将以下代码放入`$KVSTORE_HOME/src/main/java/io/example/App.java`文件中:

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

它是应用程序的入口点。
在这里，我们创建了一个特殊的对象“Environment”，它知道在哪里存储应用程序状态。
然后我们创建并启动 gRPC 服务器来处理 Tendermint Core 请求。

使用以下内容创建 `$KVSTORE_HOME/src/main/java/io/example/GrpcServer.java` 文件:
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

## 1.5 启动和运行

要创建默认配置、nodeKey 和私有验证器文件，让我们
执行 `tendermint init`。 但在我们这样做之前，我们需要安装
Tendermint 核心。

```bash
$ rm -rf /tmp/example
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

然后我们需要启动 Tendermint Core 并将其指向我们的应用程序。 住宿
在应用程序目录中执行:

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

`dGVuZGVybWludA==` 和 `cm9ja3M=` 分别是 `tendermint` 和 `rocks` 的 ASCII 码的 base64 编码。

## 结尾

我希望一切顺利，你的第一个，但希望不是最后一个，
Tendermint Core 应用程序已启动并正在运行。 如果没有，请[打开一个问题
Github](https://github.com/tendermint/tendermint/issues/new/choose)。 挖
更深入地阅读 [文档](https://docs.tendermint.com/master/)。

这个示例项目的完整源代码可以在[这里](https://github.com/climber73/tendermint-abci-grpc-java)找到。
