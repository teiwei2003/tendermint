# 埋め込む

## ブロック同期リアクター

-同期のためにプールを調整します
-ストアの永続性を調整します
-sm.BlockExecutorを使用して、再生ブロックをアプリケーションに調整します
-高速同期とコンセンサスの切り替えを処理します
-それはp2p.BaseReactorです
-pool.Start()とそのpoolRoutine()を開始します
-シリアル化のためにすべての具体的なタイプとインターフェースを登録します

### poolRoutine

-これらのチャネルを聞いてください:
    -プールはrequestsChに送信することで特定のピアにブロックを要求し、ブロックリアクターは次に送信します
    特定の高さ＆bcBlockRequestMessage
    -プールは、timeoutsChに公開することにより、特定のピアのタイムアウトを示します
    -switchToConsensusTickerは定期的にコンセンサスへの切り替えを試みます
    -trySyncTickerは、遅れているかどうかを定期的にチェックしてから、同期に追いつきます
        -プールに使用可能な新しいブロックがない場合、同期をスキップします
-プールからダウンロードしたブロックを取得してアプリを同期し、アプリやストアに提供してみてください
  それらはディスク上にあります
-スイッチ/ピアによって呼び出された受信を実現します
    -ピアから新しいブロックを受信すると、プールでAddBlockを呼び出します

## ブロックプール

-ノードからブロックをダウンロードする責任があります
-makeRequestersRoutine()
    -タイムアウトピアを削除します
    -makeNextRequester()を呼び出して、新しいリクエスターを開始します
-リクエストroutine():
    -ピアを選択してリクエストを送信し、次の状態になるまでブロックします.
        -pool.Quitを聞いてプールを停止します
        -Quitを聞いてリクエスターを停止します
        -リクエストがやり直されました
        -ブロックを受け取りました
            -gotBlockChは奇妙です

## BlocksyncReactorでルーチンを実行する

！[Go Routine Diagram](img/bc-reactor-routines.png)
