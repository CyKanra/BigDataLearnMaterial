# 分散大規模データ処理システム -- Hadoop-10

# 第５章　YARN資源調度-1

## 第１節　YARNフレームワーク

![image-20250827164414246](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250827164414246.png)

**ResourceManager（RM）**：
　クライアントからのリクエストを処理し、ApplicationMasterの起動・監視、NodeManager の監視、リソースの割り当てとスケジューリングを行う。

**NodeManager（NM）**：
　単一ノード上のリソースを管理し、ResourceManagerからの命令やApplicationMasterからの命令を処理する。

**ApplicationMaster（AM）**：
 　データを分割し、アプリケーションのためにリソースを要求し、内部タスクへ割り当て、さらにタスクの監視と障害対応を行う。

**Container**：
 　任意のタスクを実行するための抽象的な環境。CPU、メモリなどの複数のリソースや環境変数、起動コマンドなど、タスク実行に必要な情報をカプセル化して容器に纏める。

## 第２節　YARN運行仕組み

![image-20260115062223925](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260115062223925.png)

**Taskの提出**（1-5）

- WordCount計算案例に`JavaPractice-1.0.0-SNAPSHOT.jar`運行パッケージがあり、それをあるノードにアップロードする。
- HDFSコマンドでパッケージを運行し、内部的には `YarnRunner` が呼び出され、ジョブの提出処理が始まる。その時、ResourceManagerへ計算任務Applicationの申請を提出する。
- ResourceManagerがApplicationIDと、HDFS上の一時ディレクトリのパスを返す。例えば、`hdfs://.../.staging/application_xxxx`。
- ノードはジョブに必要なファイルを先ほどのディレクトリにアップロードする。`job.split`分割情報、`job.xml`ジョブの設定情報、`wc.jar`運行パッケージを含む。
- 運行ファイル提出が完成したらResourceManagerに通知し、当のノードのApplicationMasterを起動する申請を提出する。

**ジョブの初期化**（6-9）

- RMは、申請されたApplicationを1つのTaskとして初期化し、余力があるノードを決める。
- NodeManagerはRMからTaskを受ける。
- NodeManagerがAppMasterを動かすためのContainerを作成する。
- NodeManagerがHDFSからジョブ資源をダウンロードする。そこまでAppMasterがジョブ実行の準備は完成した。

![image-20250828172350879](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250828172350879.png)

**タスク割り当て**（10-11）

- AppMasterはRMに複数の運行資源の要求を提出する。
- ジョブの初期化の流れを繰り返す。複数の初期化されたAppTaskが隊列に読み込む。後でNodeManagerに順次に割り当てる。
- 別のノードが要るかどうかのは`Job.split`の切片情報によって決める。

**タスク運行**（12）

- AppMasterがNodeManagerに対してMapTask実行の起動スクリプトを送信する。各 NodeManagerはContainerを起動し、その中でMapTaskを実行する。
- 全MapTaskが完了したら、その出力結果はAppMasterに報告する。AppMasterは再度RMにReduceTask用Containerの資源を申請する。

![image-20251031113704010](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20251031113704010.png)

**ReduceTask資源の申請**（13）

- RMに対してReduceTask用のContainerを申請する。
- RMはクラスタ全体のNodeManagerの状態を確認し、空きのあるノードにContainerを優先的に割り当てる。

**ReduceTaskの実行開始**（14-15）

- RMからの指示を受けたNodeManagerは、ReduceTaskのContainerを作成し、その中でReduceTaskプロセスを起動される。
- ReduceTaskは、AppMasterを通じてすべてのMapTask出力ファイルの所在を問い合わせる。
- この時点で自分の担当するパーティションのデータを、各NodeManager上のMapTask出力からネットワーク経由でダウンロード (Shuffle)する。
- 因みに、その自分の担当するパーティション数はジョブの最初に決まっていた。

**ジョブの終了**

- 各タスクはNodeManagerに完了を報告し、AppMasterがそれを集約して状態を確認する。
- 「ジョブ完了」通知を受けたResourceManagerはAppMasterの登録を解除する。
- NodeManagerがAppMasterのコンテナも含め、一時ファイルや作業ディレクトリなどを削除し、メモリやCPUなどのリソースが解放される。

**進捗の確認**

　ジョブを実行してる間に進捗を追って出力、管理するが要る。大体4つのところがある。

- Task → AppMaster：各NodeManager上のTaskがAppMasterに進捗を報告する。
- AppMaster → RM：ジョブ全体の状態を報告する。
- Client → AppMaster：定期ポーリングで最新状態を取得する。
- Client → User：コンソールに進行率やタスク件数を出力する。

　Yarnはジョブの進捗と状態をAppMasterに報告する。クライアントはAppMasterにリクエストを発送し、取得する情報をユーザに展示する。リクエストの間隔はmapreduce.client.completion.pollinterval変数でコントロールできる。

　上述の方式を除いて、クライアントが手動でwaitForCompletion()メソッドを呼び出して進捗と状態をチェックできる。

　ジョブが終わったらAppMasterとContainer が清理される。実行後のMapReduceジョブ状態を構造化してJobHistoryServerに格納される。

## 第２節　YARN調度策略

　Hadoop / YARNの代表的な3つのスケジューラはFIFO、Capacity Scheduler和Fair Schedulerある。Hadoopデフォルト状態はCapacity Scheduler策略です。

　yarn-default.xml設定ファイルに「yarn.resourcemanager.scheduler.class」変数は対応のスケジューラをコントロールできる。

![image-20260115063839733](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260115063839733.png)

**FIFO Scheduler**

　一番理解しやすいスケジューラで、投入順に先に来るジョブを先ず実行するんです。

![image-20260115065739691](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260115065739691.png)
