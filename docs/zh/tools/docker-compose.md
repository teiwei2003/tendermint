# Docker 撰写

使用 Docker Compose，您可以使用单个命令启动本地测试网.

## 要求

1.[安装tendermint](../introduction/install.md)
2.[安装docker](https://docs.docker.com/engine/installation/)
3.[安装docker-compose](https://docs.docker.com/compose/install/)

## 建造

构建 `tendermint` 二进制文件和可选的 `tendermint/localnode`
码头工人形象.

请注意，二进制文件将挂载到容器中，因此无需进行更新即可
重建图像.

```sh
# Build the linux binary in ./build
make build-linux

# (optionally) Build tendermint/localnode image
make build-docker-localnode
```

## Run a testnet

To start a 4 node testnet run:

```sh
make localnet-start
```

节点将它们的 RPC 服务器绑定到端口 26657、26660、26662 和 26664 上
主持人.

此文件使用 localnode 映像创建一个 4 节点网络.

网络节点将其 P2P 和 RPC 端点暴露给主机
分别在端口 26656-26657、26659-26660、26661-26662 和 26663-26664 上.

第一个节点(`node0`)公开了两个额外的端口:6060 用于分析使用
[`pprof`](https://golang.org/pkg/net/http/pprof) 和 `9090` - 用于普罗米修斯
服务器(如果您不知道如何开始结帐 ["第一步 |
普罗米修斯"](https://prometheus.io/docs/introduction/first_steps/)).

要更新二进制文件，只需重建它并重新启动节点:

```sh
make build-linux
make localnet-start
```

## 配置

`make localnet-start` 为 `./build` 中的 4 节点测试网创建文件
调用 `tendermint testnet` 命令.

将`./build`目录挂载到`/tendermint`挂载点进行attach
二进制文件和配置文件到容器.

要更改验证器/非验证器的数量，请更改 `localnet-start` Makefile 目标 [此处](../../Makefile):

```makefile
localnet-start: localnet-stop
  @if ! [ -f build/node0/config/genesis.json ]; then docker run --rm -v $(CURDIR)/build:/tendermint:Z tendermint/localnode testnet --v 5 --n 3 --o . --populate-persistent-peers --starting-ip-address 192.167.10.2 ; fi
  docker-compose up
```

该命令现在将为 5 个验证器和 3 个生成配置文件
非验证者. 除了生成新的配置文件，还需要编辑 docker-compose 文件.
需要再添加 4 个节点才能充分利用生成的配置文件.

```yml
  node3: # bump by 1 for every node
    container_name: node3 # bump by 1 for every node
    image: "tendermint/localnode"
    environment:
      - ID=3
      - LOG=${LOG:-tendermint.log}
    ports:
      - "26663-26664:26656-26657" # Bump 26663-26664 by one for every node
    volumes:
      - ./build:/tendermint:Z
    networks:
      localnet:
        ipv4_address: 192.167.10.5 # bump the final digit by 1 for every node
```

Before running it, don't forget to cleanup the old files:

```sh
# Clear the build folder
rm -rf ./build/node*
```

## 配置 ABCI 容器

要在 4 节点设置中使用您自己的 ABCI 应用程序，请编辑 [docker-compose.yaml](https://github.com/tendermint/tendermint/blob/master/docker-compose.yml) 文件并将图像添加到您的 ABCI 应用.

```yml
 abci0:
    container_name: abci0
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.6

  abci1:
    container_name: abci1
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.7

  abci2:
    container_name: abci2
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.8

  abci3:
    container_name: abci3
    image: "abci-image"
    build:
      context: .
      dockerfile: abci.Dockerfile
    command: <insert command to run your abci application>
    networks:
      localnet:
        ipv4_address: 192.167.10.9

```

覆盖每个节点中的 [command](https://github.com/tendermint/tendermint/blob/master/networks/local/localnode/Dockerfile#L12) 以连接到它的 ABCI.

```yml
  node0:
    container_name: node0
    image: "tendermint/localnode"
    ports:
      - "26656-26657:26656-26657"
    environment:
      - ID=0
      - LOG=$${LOG:-tendermint.log}
    volumes:
      - ./build:/tendermint:Z
    command: node --proxy-app=tcp://abci0:26658
    networks:
      localnet:
        ipv4_address: 192.167.10.2
```

同样对 node1、node2 和 node3 做然后 [run testnet](https://github.com/tendermint/tendermint/blob/master/docs/networks/docker-compose.md#run-a-testnet)

## 记录

日志保存在附加卷下的“tendermint.log”文件中. 如果
`LOG` 环境变量在启动时设置为 `stdout`，不保存日志，
但印在屏幕上.

## 特殊二进制文件

如果您有多个名称不同的二进制文件，您可以指定哪一个
使用 `BINARY` 环境变量运行. 二进制的路径是相对的
到附加的卷.
