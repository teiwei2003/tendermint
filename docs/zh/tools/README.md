# 概述

Tendermint 有一些与之相关的工具:

- [调试](./debugging/pro.md)
- [基准测试](#benchmarking)
- [测试网](#testnets)

## 基准测试

- <https://github.com/informalsystems/tm-load-test>

`tm-load-test` 是一个分布式负载测试工具(和框架)，用于负载
测试 Tendermint 网络.

## 测试网

- <https://github.com/informalsystems/testnets>

该存储库包含各种不同的测试网络配置，
并与 Tendermint 相关.

使用 [Docker Compose](./docker-compose.md) 在你的系统上启动 Tendermint 测试网
本地机器.

使用 [Terraform and Ansible](./terraform-and-ansible.md) 部署 Tendermint
测试网到云端.

有关初始化测试网的更多帮助，请参阅 `tendermint testnet --help` 命令.
