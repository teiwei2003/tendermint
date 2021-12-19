# Terraform＆Ansible

>注:テンダーミントチームは現在、これらのコマンド/ファイルを管理していません。 慎重に使用してください。

自動展開は使用することです
[Terraform](https://www.terraform.io/)デジタルでサーバーを作成する
次にOcean [Ansible](http://www.ansible.com/)を作成して管理します
これらのサーバー上のテストネット。

## インストール

注:[Integratedbashを参照してください
スクリプト](https://github.com/tendermint/tendermint/blob/master/networks/remote/integration.sh)
新鮮なDO液滴で実行でき、自動的に回転します4
ノードテストネットワーク。 このスクリプトは、以下に説明するすべてのことを多かれ少なかれ達成します。

-[Terraform](https://www.terraform.io/downloads.html)をインストールして
   [Ansible](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
   Linuxマシンの場合。
-[DigitalOceanAPIを作成します
   トークン](https://cloud.digitalocean.com/settings/api/tokens)
   そして書き込み機能。
-python dopyパッケージをインストールします( `pip install dopy`)
-SSHキーを作成します( `ssh-keygen`)
-環境変数を設定します。

```sh
export DO_API_TOKEN="abcdef01234567890abcdef01234567890"
export SSH_KEY_FILE="$HOME/.ssh/id_rsa.pub"
```

これらは `terraform`と` ansible`によって使用されます。

## 地形

このステップでは、4つのデジタル海洋液滴を作成します。 まず、
正しいディレクトリ:

```sh
cd $GOPATH/src/github.com/tendermint/tendermint/networks/remote/terraform
```

then:

```sh
terraform init
terraform apply -var DO_API_TOKEN="$DO_API_TOKEN" -var SSH_KEY_FILE="$SSH_KEY_FILE"
```

ドロップレットに属するIPアドレスのリストが表示されます。

ドロップレットを作成して実行したら、Ansibleをセットアップしましょう。

## Ansible

[ansibleのスクリプト
ディレクトリ](https://github.com/tendermint/tendermint/tree/master/networks/remote/ansible)
ansibleロールを実行して、センチネルノードアーキテクチャを構成します。 絶対です
このディレクトリに切り替えて、ansibleを実行します
( `cd $ GOPATH/src/github.com/tendermint/tendermint/networks/remote/ansible`)。

いくつかの役割は自明です:

まず、テンダーミントのパスを指定して液滴を構成します
( `BINARY`)とノードファイル(` CONFIGDIR`)。 後者は
「node0、node1、...」などの名前のディレクトリの数(
生成された液滴の数)。

ノードファイルを作成するには、次のコマンドを実行します。

```sh
tendermint testnet
```

次に、ドロップレットを実行するように構成します。

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet config.yml -e BINARY=$GOPATH/src/github.com/tendermint/tendermint/build/tendermint -e CONFIGDIR=$GOPATH/src/github.com/tendermint/tendermint/networks/remote/ansible/mytestnet
```

見て！ すべてのドロップレットに `tendermin`バイナリが含まれるようになり、必須になります
テストネットを実行するようにファイルを構成します。

次に、インストールの役割を実行します。

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet install.yml
```

以下に示すように、実行されます
すべてのドロップレットの `tendermint node --proxy-app = kvstore`。 私たちはしますが
この役割をすぐに変更して再実行します。これが最初の実行です
それぞれに対応する各node_info.idを取得できます
`node_info.listen_addr`。 (この部分は将来自動化されます)。 存在
ブラウザ(または `curl`を使用)で、ドロップレットごとにIP:26657/statusに移動します
そして、今述べた2つの `node_info`フィールドに注意してください。 注意ブロック
作成されていません( `latest_block_height`はゼロではなくゼロにする必要があります
増加)。

次に、 `roles/install/templates/systemd.service.j2`を開いて、
行 `ExecStart`は次のようになります。

```sh
ExecStart=/usr/bin/tendermint node --proxy-app=kvstore
```

そして、関連情報とともに `--p2p.persistent-peers`のフラグを追加します
ノードごと。 結果のファイルは次のようになります。

```sh
[Unit]
Description={{service}}
Requires=network-online.target
After=network-online.target

[Service]
Restart=on-failure
User={{service}}
Group={{service}}
PermissionsStartOnly=true
ExecStart=/usr/bin/tendermint node --proxy-app=kvstore --p2p.persistent-peers=167b80242c300bf0ccfb3ced3dec60dc2a81776e@165.227.41.206:26656,3c7a5920811550c04bf7a0b2f1e02ab52317b5e6@165.227.43.146:26656,303a1a4312c30525c99ba66522dd81cca56a361a@159.89.115.32:26656,b686c2a7f4b1b46dca96af3a0f31a6a7beae0be4@159.89.119.125:26656
ExecReload=/bin/kill -HUP $MAINPID
KillSignal=SIGTERM

[Install]
WantedBy=multi-user.target
```

Then, stop the nodes:

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet stop.yml
```

Finally, we run the install role again:

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet install.yml
```

すべての液滴に新しいロゴを付けて「テンダーミントノード」を再実行します。 この
`latest_block_hash`が変更され、` latest_block_height`が変更されます。
増加。 これでテストネットが稼働しています:)

ステータスロールでログを表示します。

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet status.yml
```

## 記録

最も粗雑な方法は、上記のステータスロールです。 発送も可能です
エラスティックスタックであるLogz.ioにログインします(Elastic Search、Logstash、Kibana)
サービスプロバイダー。 自動的にログインするようにノードを設定できます。
アカウントを作成し、[this
ページ](https://app.logz.io/#/dashboard/data-sources/Filebeat)、次に:

```sh
yum install systemd-devel || echo "This will only work on RHEL-based systems."
apt-get install libsystemd-dev || echo "This will only work on Debian-based systems."

go get github.com/mheese/journalbeat
ansible-playbook -i inventory/digital_ocean.py -l sentrynet logzio.yml -e LOGZIO_TOKEN=ABCDEFGHIJKLMNOPQRSTUVWXYZ012345
```

## 掃除

ドロップレットを削除するには、次のコマンドを実行します。

```sh
terraform destroy -var DO_API_TOKEN="$DO_API_TOKEN" -var SSH_KEY_FILE="$SSH_KEY_FILE"
```
