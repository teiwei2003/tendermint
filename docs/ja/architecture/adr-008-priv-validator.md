# ADR 008:SocketPV

Tendermintノードは、2つのインプロセスPrivValidatorsのみをサポートする必要があります
達成:

-FilePVは、「priv_validator.json」ファイルで暗号化されていない秘密鍵を使用します-いいえ
  構成する必要があります( `tendermint initvalidator`のみ).
-TCPValとIPCValは、それぞれTCPソケットとUnixソケットを使用して署名要求を送信します
  別のプロセスへ-ユーザーは自分でプロセスを開始する責任があります.

TCPValアドレスとIPCValアドレスの両方を、コマンドラインのフラグで指定できます
または構成ファイル内.TCPValアドレスは次の形式である必要があります
`tcp:// <ip_address>:<port>`およびIPCValアドレス `unix:///path/to/file.sock`-
これを行うと、Tendermintはプライベートバリデーターファイルを無視します.

TCPValは、指定されたアドレスで外部からの着信接続をリッスンします
プライベートバリデータープロセス.少なくとも1つの外部が存在するまで、すべての操作を停止します
プロセスは正常に接続されました.

外部のpriv_validatorプロセスは、接続するアドレスをダイヤルします
テンダーミント、次にテンダーミントはにリクエストを送信します
投票と提案に署名します.したがって、外部プロセスが接続を開始し、
ただし、Tendermintプロセスはすべてのリクエストを発行します.後の段階で、
フォールトトレランスのために複数のバリデーターをサポートします.二重署名を防ぐために、彼らは
同期が必要ですが、これは外部ソリューションに延期されます(#1185を参照).

代わりに、IPCValは既存の開いているソケットとのアウトバウンド接続を確立します
外部バリデータープロセスを渡します.

さらに、Tendermintはその中で実行できる実装を提供します
外部プロセス.これらには以下が含まれます:

-FilePVは秘密鍵を暗号化します.ユーザーは、次のパスワードを入力する必要があります.
  キーは、プロセスの開始時に復号化されます.
-LedgerPVは、Ledger NanoSを使用してすべての署名を処理します.
