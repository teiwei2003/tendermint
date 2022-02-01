# Terraform & Ansible

> 注意:tendermint 团队目前不维护这些命令/文件.请小心使用它们.

自动化部署是使用
[Terraform](https://www.terraform.io/) 在 Digital 上创建服务器
Ocean然后[Ansible](http://www.ansible.com/)创建和管理
这些服务器上的测试网.

## 安装

注意:请参阅 [集成 bash
脚本](https://github.com/tendermint/tendermint/blob/master/networks/remote/integration.sh)
可以在新鲜的 DO 液滴上运行，并会自动旋转 4
节点测试网.该脚本或多或少完成了下面描述的所有事情.

- 安装 [Terraform](https://www.terraform.io/downloads.html) 和
  [Ansible](http://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html)
  在 Linux 机器上.
- 创建一个 [DigitalOcean API
  令牌](https://cloud.digitalocean.com/settings/api/tokens) 带读
  和写入能力.
- 安装 python dopy 包(`pip install dopy`)
- 创建 SSH 密钥 (`ssh-keygen`)
- 设置环境变量:

```sh
export DO_API_TOKEN="abcdef01234567890abcdef01234567890"
export SSH_KEY_FILE="$HOME/.ssh/id_rsa.pub"
```

这些将被 `terraform` 和 `ansible` 使用.

## 地形

这一步将创建四个数字海洋水滴. 首先，前往
正确的目录:

```sh
cd $GOPATH/src/github.com/tendermint/tendermint/networks/remote/terraform
```

then:

```sh
terraform init
terraform apply -var DO_API_TOKEN="$DO_API_TOKEN" -var SSH_KEY_FILE="$SSH_KEY_FILE"
```

您将获得属于您的 Droplet 的 IP 地址列表.

创建并运行 Droplet 后，让我们设置 Ansible.

## Ansible

[the ansible 中的剧本
目录](https://github.com/tendermint/tendermint/tree/master/networks/remote/ansible)
运行 ansible 角色来配置哨兵节点架构. 你必须
切换到这个目录运行ansible
(`cd $GOPATH/src/github.com/tendermint/tendermint/networks/remote/ansible`).

有几个角色是不言自明的:

首先，我们通过指定tendermint的路径来配置我们的droplet
(`BINARY`) 和节点文件 (`CONFIGDIR`). 后者期望任何
名为“node0、node1、...”等的目录数(等于
产生的液滴数).

要创建节点文件，请运行:

```sh
tendermint testnet
```

然后，配置我们的液滴运行:

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet config.yml -e BINARY=$GOPATH/src/github.com/tendermint/tendermint/build/tendermint -e CONFIGDIR=$GOPATH/src/github.com/tendermint/tendermint/networks/remote/ansible/mytestnet
```

瞧！ 你所有的 Droplets 现在都有 `tendermin` 二进制文件并且是必需的
配置文件来运行测试网.

接下来，我们运行安装角色:

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet install.yml
```

正如你将在下面看到的，它执行
所有 Droplet 上的 `tendermint node --proxy-app=kvstore`. 虽然我们会
很快就会修改这个角色并再次运行它，这是第一次执行
允许我们获取对应于每个的每个 `node_info.id`
`node_info.listen_addr`. (这部分将来会自动化). 在
您的浏览器(或使用 `curl`)，对于每个 Droplet，转到 IP:26657/status
并注意刚才提到的两个 `node_info` 字段. 注意块
没有被创建(`latest_block_height` 应该是零而不是
增加).

接下来，打开 `roles/install/templates/systemd.service.j2` 并查找
行 `ExecStart` 应该类似于:

```sh
ExecStart=/usr/bin/tendermint node --proxy-app=kvstore
```

并添加带有相关信息的 `--p2p.persistent-peers` 标志
对于每个节点. 生成的文件应该类似于:

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

在所有液滴上重新运行带有新标志的“tendermint 节点”. 这
`latest_block_hash` 现在应该改变，并且 `latest_block_height`
增加. 您的测试网现已启动并正在运行 :)

查看具有 status 角色的日志:

```sh
ansible-playbook -i inventory/digital_ocean.py -l sentrynet status.yml
```

## 记录

最粗暴的方法是上面描述的状态角色. 你也可以发货
记录到 Logz.io，一个弹性堆栈(弹性搜索、Logstash 和 Kibana)
服务提供者. 您可以将节点设置为自动登录.
创建一个帐户并从 [this
页面](https://app.logz.io/#/dashboard/data-sources/Filebeat)，然后:

```sh
yum install systemd-devel || echo "This will only work on RHEL-based systems."
apt-get install libsystemd-dev || echo "This will only work on Debian-based systems."

go get github.com/mheese/journalbeat
ansible-playbook -i inventory/digital_ocean.py -l sentrynet logzio.yml -e LOGZIO_TOKEN=ABCDEFGHIJKLMNOPQRSTUVWXYZ012345
```

## 清理

要删除您的液滴，请运行:

```sh
terraform destroy -var DO_API_TOKEN="$DO_API_TOKEN" -var SSH_KEY_FILE="$SSH_KEY_FILE"
```
