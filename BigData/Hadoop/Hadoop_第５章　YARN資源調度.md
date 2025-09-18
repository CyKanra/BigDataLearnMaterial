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

![image-20250828172350879](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250828172350879.png)

**Taskの提出**（1-5）

1. WordCount計算案例に`JavaPractice-1.0.0-SNAPSHOT.jar`運行パッケージがあり、それをあるノードにアップロードする。
2. HDFSコマンドでパッケージを運行し、内部的には `YarnRunner` が呼び出され、ジョブの提出処理が始まる。その時、ResourceManagerへ計算任務Applicationの申請を提出する。
3. ResourceManagerがApplicationIDと、HDFS上の一時ディレクトリのパスを返す。例えば、`hdfs://.../.staging/application_xxxx`。
4. ノードはジョブに必要なファイルを先ほどのディレクトリにアップロードする。`job.split`分割情報、`job.xml`ジョブの設定情報、`wc.jar`運行パッケージを含む。
5. 運行ファイル提出が完成したらResourceManagerに通知し、当のノードのApplicationMasterを起動する申請を提出する。

**ジョブの初期化**（6-9）

1. ResourceManagerは、申請されたApplicationを1つのTaskとして初期化し、余力があるノードを決める。
2. NodeManagerはResourceManagerからTaskを受ける。
3. NodeManagerがApplicationMasterを動かすためのContainerを作成する。
4. NodeManagerがHDFSからジョブ資源をダウンロードする。そこまでApplicationMasterがジョブ実行の準備は完成した。

**タスク割り当て**（10-11）

1. AppMasterはResourceManagerに複数の運行資源の要求を提出する。
2. ジョブの初期化の流れを繰り返す。複数の初期化されたApplicationTaskが隊列に読み込む。後でNodeManagerに順次割り当てる。
3. 別のノードが要るかどうかのは`Job.split`の切片情報によって決める。

**タスク運行**（12-15）

1. AppMasterがNodeManagerに対してMapTask実行の起動スクリプトを送信する。各 NodeManagerはContainerを起動し、その中でMapTaskを実行する。
2. 全MapTaskが完了したら、AppMasterは再度ResourceManagerにReduceTask用Containerの資源を申請する。