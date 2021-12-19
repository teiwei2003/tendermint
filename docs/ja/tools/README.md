# 概要

Tendermintには、それに関連するいくつかのツールがあります。

-[デバッグ](./debugging/pro.md)
-[ベンチマーク](#benchmarking)
-[TESTNET](#testnets)

## ベンチマーク

-<https://github.com/informalsystems/tm-load-test>

`tm-load-test`は、負荷用の分散型負荷テストツール(およびフレームワーク)です。
Tendermintネットワークをテストします。

## テストネット

-<https://github.com/informalsystems/testnets>

このリポジトリには、さまざまなテストネットワーク構成が含まれています。
そしてテンダーミントに関連しています。

[Docker Compose](./docker-compose.md)を使用して、システムでTendermintテストネットを起動します
ローカルマシン。

[Terraform and Ansible](./terraform-and-ansible.md)を使用してTendermintをデプロイします
クラウドへのテストネット。

テストネットの初期化の詳細については、 `tendermint testnet--help`コマンドを参照してください。
