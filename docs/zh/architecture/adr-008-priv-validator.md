# ADR 008:SocketPV

Tendermint 节点应该只支持两个进程内 PrivValidator
实现:

- FilePV 在“priv_validator.json”文件中使用未加密的私钥 - 否
  需要配置(只是`tendermint init validator`).
- TCPVal 和 IPCVal 分别使用 TCP 和 Unix 套接字发送签名请求
  到另一个进程 - 用户负责自己启动该进程.

TCPVal 和 IPCVal 地址都可以通过命令行中的标志提供
或在配置文件中； TCPVal 地址必须采用以下形式
`tcp://<ip_address>:<port>` 和 IPCVal 地址 `unix:///path/to/file.sock` -
这样做会导致 Tendermint 忽略任何私有验证器文件.

TCPVal 将在给定地址上侦听来自外部的传入连接
私有验证器进程.它将停止任何操作，直到至少有一个外部
进程成功连接.

外部 priv_validator 进程将拨打要连接的地址
Tendermint，然后 Tendermint 将在随后的连接上发送请求到
签署投票和提案.因此外部进程发起连接，
但是 Tendermint 进程会发出所有请求.在稍后的阶段，我们将
支持多个验证器以实现容错.为了防止双重签名，他们
需要同步，这被推迟到外部解决方案(见#1185).

相反，IPCVal 将与打开的现有套接字建立出站连接
通过外部验证器过程.

此外，Tendermint 将提供可以在其中运行的实现
外部过程.这些包括:

- FilePV 会对私钥进行加密，用户必须输入密码才能
  进程启动时解密密钥.
- LedgerPV 使用 Ledger Nano S 处理所有签名.
