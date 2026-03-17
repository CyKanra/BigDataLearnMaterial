# ビッグデータ高速計算エンジンSpark

## 第１章　Sparkの紹介

### 第１節　Sparkとは

![image-20260309064401980](D:\OneDrive\picture\Typora\BigData\Spark\image-20260309064401980.png)

　Sparkとは、高速、汎用の計算エンジン。

　HadoopのMapReduce部分とは同じの役割で、データの計算を担当する計算フレームワークです。今のビッグデータの計算は、ほとんどMapReduceからSparkに差し替えている。

**高速処理**

　Sparkはメモリベースの処理により、MapReduceより大幅に高速にデータ処理ができる。DAG実行エンジンにより効率的にデータフローを処理する。

**使いやすい**

　Scala、Java、Python、RなどのAPIを提供し、80以上の高級アルゴリズムを利用できる。さらにPython・ScalaのインタラクティブShellで簡単に検証や開発ができる。

**汎用性が高い**

　バッチ処理、SQL分析、ストリーム処理、機械学習、グラフ計算などを同一プラットフォームで統合的に実行できる。

**高い互換性**

　YARNやMesosと連携可能で、HDFS、HBase、CassandraなどHadoop系のデータをそのまま利用できる。

またStandaloneモードでも動作するため、簡単にSparkクラスタを構築できる。

#### SparkとHadoop

　Hadoopは分散式フレームワーク、分散ファイルシステムHDFS、データ計算MapReduce、資源のスケジューリングYarnやCommon共用部分を共に組み合わせる。

　Sparkは、Hadoopのデータ計算MapReduceと対応するだけ。SparkをHadoopに嵌め込んで一緒に使うんです。

MapReduceの欠点

- **MapReduce表現力が限られている**。例えば、計算ライブラリは色々インタフェースを呼び出せるはずです。MapReduceは，MapとReduce2つ操作だけで、複雑の処理ロジックに追いついてない。
- **ディスクI/Oの負荷が大きい**。MapとReduceの間にデータがディスクで転送される。ディスクにアクセスするは大量のI/O資源を費やす。
- **遅延が高い**。タスク間の接続はディスクI/Oが発生する。一方で、前のタスクが完了するまで次のタスクを開始できないため、複雑で多段階の計算処理には向きにくい。

![Spark vs Hadoop MapReduce | Download Scientific Diagram](D:\OneDrive\picture\Typora\BigData\Spark\Spark-vs-Hadoop-MapReduce.png)

　あと、実際の大規模データ処理では、主に次の3種類がある。

- **バッチ処理（オフライン処理）**：処理時間は通常、数十分～数時間
- **インタラクティブクエリ**：処理時間は通常、数十秒～数分
- **ストリーム処理（リアルタイム処理）**：処理時間は通常、数百ミリ秒～数秒

　これら3種類の処理を同時に扱う場合、従来のHadoopでは複数のソフトウェアを組み合わせて使う必要がある。MapReduce / Hive / Impala / Storm などで、Sparkの自身は上記の3つの場合を対応できる。

**何でSparkはMapReduceより速い**

- Sparkはメモリを積極的に利用する。

　MapReduceでは、1つのJobは Map段階（1つまたは複数のMap Task） と Reduce段階（1つまたは複数のReduce Task） で構成される。処理ロジックが複雑な場合、複数のJobを組み合わせる必要がある。その際、前のJobの結果をHDFSに書き込んでから次のJobに渡す 必要があるため、複雑な処理ではディスクへの書き込み・読み込みが何度も発生する。

一方、Sparkでは複数の Map / Reduce処理を連続して実行でき、途中結果をディスクに保存する必要がない。

例：

　複雑なMR処理：MR + MR + MR + MR ...

　複雑なSpark処理：MR → MR → MR ...

- マルチプロセス（MR）とマルチスレッド（Spark）の違い

　MapReduceでは Map Task と Reduce Task はプロセス単位で実行される。

　つまり各Taskは JVMプロセスとして起動され、毎回リソースを再確保する必要があり、余分な時間がかかる。SparkではTaskをスレッドとして実行 し、スレッドプールを再利用することで、Taskの起動・終了に伴うシステムコストを削減している。

### 第２節　システム構成

　Sparkの実行アーキテクチャは次の要素で構成される。

- **Cluster Manager**：クラスターのリソースを管理するコンポーネント。HadoopのNameNode存在みたい。

　Sparkは3種類のクラスタ管理方式をサポートしている。Standalone、YarnとMesosである。もしHadoop上でSparkを動かす場合、HadoopのYarnにResourceManagerとして考えるもできる。以後のSpark講解は、主にYarnモードを基づいて進める。

- **Worker Node**：作業ノードであり、NodeManagerに対応する。各ノードのローカルリソースを管理する。

- **Driver Program**：アプリケーションのmain()を実行し、SparkContextを生成する。

　Cluster Managerがリソースを割り当て、SparkContext がTaskをExecutorに送って実行させる。これはアプリケーション向けのコンポーネントで、ノードやリソースを制御するためのものじゃない、役割はYarnのApplicationMasterに近い。

- **Executor**：Worker Node上で動作し、Driverから送られたTaskを実行し、計算結果をDriverに返す。YarnのContainerに対応すし、実際にプログラムを処理するコンポーネントです。

![image-20260313065726955](D:\OneDrive\picture\Typora\BigData\Spark\image-20260313065726955.png)

### 第３節　Sparkクラスタ実行モード

Sparkは次の3つのクラスタ構成をサポートしている。

- Standalone
- YARN
- Mesos

**Standaloneモード**

- 独立したモードで、Spark自身が必要なサービスを持っている。そのため、他のリソース管理システムに依存せず、単独でクラスターに導入できる。

構成要素

- Cluster Manager：Master
- Worker Node：Worker
- リソース割り当ては粗い粒度（coarse-grained）の方式のみ対応

**Spark on YARN モード**

- 実際の運用環境で最もよく使われている構成で、YARNはコミュニティのサポートが強く、現在では大規模データクラスターのリソース管理システムの標準の一つになりつつある。

- Spark on YARNには次の2つの実行モードがある。

  - yarn-cluster：本番環境での利用に向いている。

  - yarn-client：対話的な処理やデバッグなど、すぐにアプリの出力を確認したい場合に向いている。

- リソース割り当ては粗い粒度（coarse-grained）方式のみ対応。

構成要素

- Cluster Manager：ResourceManager
- Worker Node：NodeManager

**Spark on Mesos モード**

- Spark公式が推奨している方式の一つ。Sparkは開発当初からMesosとの連携を考えて設計されている。
- SparkをMesos上で動かすと、YARN上で動かす場合よりも柔軟で自然なリソース管理ができる。
- 粗い粒度（coarse-grained）と細かい粒度（fine-grained）の両方のリソース割り当て方式に対応できる。

構成要素

- Cluster Manager：Mesos Master
- Worker Node：Mesos Slave

**粗い粒度モード（Coarse-grained Mode）**

　アプリケーションを実行する前に、必要なリソースをまとめて確保しておき、実行中は使っていなくてもそのリソースを占有し続ける。そして、アプリケーションが終了した時点でリソースが解放される。

**細かい粒度モード（Fine-grained Mode）**

　粗い粒度モードではリソースが無駄になりやすいため、Spark on Mesosでは細かい粒度のスケジューリング方式も用意されている。この方式は現在のクラウドコンピューティングの考え方に近く、必要な分だけリソースを割り当てる。

### 第４節　公式サイト紹介

公式サイトURL：[Apache Spark™ - Unified Engine for large-scale data analytics](https://spark.apache.org/)

ソフトウェアのダウンロードURL：[Apache Archive Distribution Directory](https://archive.apache.org/dist/spark/)

最新のドキュメントURL：[Overview - Spark 4.1.1 Documentation](https://spark.apache.org/docs/latest/)

用語集URL：[Cluster Mode Overview - Spark 4.1.1 Documentation](https://spark.apache.org/docs/latest/cluster-overview.html)

![image-20260316214418628](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316214418628.png)

　Downloadに入ってSparkソフトウェアのダウンロードができる。古いバージョンを取得したいは下の赤枠リンクをクリックしてダウロード画面に入る。

![image-20260316214713871](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316214713871.png)

　LibraryはSpark入門について簡単の紹介。

![image-20260316215336243](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316215336243.png)

一番使うのは公式ドキュメント、古いバージョンのドキュメントもある。

![image-20260316221605056](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316221605056.png)

　プログラミングガイド。開発者を助かってSpark特性を利用してアプリケーションを開発する。ほとんど離れない存在です。

![image-20260316222022473](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316222022473.png)

　公式APIドキュメント。実際にあんまり使ってない。Sparkはオープンソースなので、集成開発環境に見つけられる。

![image-20260316222617908](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316222617908.png)

　どうやってSparkクラスタを構築するのを紹介する。

![image-20260316223047155](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316223047155.png)

　常用のところ。Spark属性の配置、最適化、セキュリティなどここに書かれる。

![image-20260316223339203](D:\OneDrive\picture\Typora\BigData\Spark\image-20260316223339203.png)

### 第５節　用語集の紹介

用語集URL：[Cluster Mode Overview - Spark 4.1.1 Documentation](https://spark.apache.org/docs/latest/cluster-overview.html)

**Application**
　ユーザーが提出するSparkアプリケーション。単純アップロードのパッケージを指すことじゃない、クラスター内のDriverと複数のExecutorで構成される全体です。もしMapReduceを例えば、Spark ApplicationとMapReduce Jobが似ている。

**Application JAR**
　Sparkアプリケーションを含むJARファイル。SparkやHadoopのJARを含めずJARパッケージを作るお勧め。クラスタにその相関依頼があって実行時に追加される。なお、パッケージサイズが大きいなら各ノードに配布するのは時間がかかる。

**Driver Program**
　アプリケーションのmain()を実行し、SparkContextを作成するプログラム。

**Cluster Manager**
　クラスターのリソースを管理するサービス。

**Deploy Mode**？？？
　Driverプロセスをどこで実行するかを決めるモード。

- Clusterモード：Driverはクラスター内部で実行される
- Clientモード：Driverはクラスター外部で実行される

**Worker Node**
 　アプリケーションを実行する作業ノード。

**Executor**
　Taskを実行し、データを保持するプロセス。各アプリケーションは自分専用のExecutorを持ち、それぞれ独立して動作する。

**Task**
　Executor上で実行される最小の処理単位。

**Job**？？？
　ユーザープログラムでAction操作を呼び出すたびに、新しいJobが作られる。

**Stage**？？？
　1つのJobは複数のStageに分割され、各Stageは複数の Taskの集合 で構成される。