# ビッグデータ高速計算エンジンSpark

## 第１章　Sparkの紹介

### Sparkとは

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

### Spark と Hadoop

　Hadoopは分散式フレームワーク、分散ファイルシステムHDFS、データ計算MapReduce、資源のスケジューリングYarnやCommon共用部分を共に組み合わせる。

　Sparkは、Hadoopのデータ計算MapReduceと対応するだけ。SparkをHadoopに嵌め込んで一緒に使うんです。

**MapReduceの欠点**

- **MapReduce表現力が限られている**。例えば、計算ライブラリは色々インタフェースを呼び出せるはずです。MapReduceは，MapとReduce2つ操作だけで、複雑の処理ロジックに追いついてない。
- **ディスクI/Oの負荷が大きい**。MapとReduceの間にデータがディスクで転送される。ディスクにアクセスするは大量のI/O資源を費やす。
- **遅延が高い**。タスク間の接続はディスクI/Oが発生する。一方で、前のタスクが完了するまで次のタスクを開始できないため、複雑で多段階の計算処理には向きにくい。

![Spark vs Hadoop MapReduce | Download Scientific Diagram](D:\OneDrive\picture\Typora\BigData\Spark\Spark-vs-Hadoop-MapReduce.png)

　あと、実際の大規模データ処理では、主に次の3種類がある。

- **バッチ処理（オフライン処理）**：処理時間は通常、数十分～数時間
- **インタラクティブクエリ**：処理時間は通常、数十秒～数分
- **ストリーム処理（リアルタイム処理）**：処理時間は通常、数百ミリ秒～数秒

　これら3種類の処理を同時に扱う場合、従来のHadoopでは複数のソフトウェアを組み合わせて使う必要がある。例：MapReduce / Hive / Impala / Storm など。Sparkの自身は上記の3つの場合を対応できる。

**何でSparkはMapReduceより速い**

- Sparkはメモリを積極的に利用する。

　MapReduceでは、1つのJobは Map段階（1つまたは複数のMap Task） と Reduce段階（1つまたは複数のReduce Task） で構成される。処理ロジックが複雑な場合、複数のJobを組み合わせる必要がある。その際、前のJobの結果を HDFSに書き込んでから次のJobに渡す 必要があるため、複雑な処理ではディスクへの書き込み・読み込みが何度も発生する。

一方、Sparkでは複数の Map / Reduce処理を連続して実行 でき、途中結果をディスクに保存する必要がない。

例：

　複雑なMR処理：`MR + MR + MR + MR ...`

　複雑なSpark処理：`MR → MR → MR ...`

- マルチプロセス（MR）とマルチスレッド（Spark）の違い

　MapReduceでは Map Task と Reduce Task はプロセス単位で実行される。

　つまり各Taskは JVMプロセス として起動され、毎回リソースを再確保する必要があり、余分な時間がかかる。Sparkでは Taskをスレッドとして実行 し、スレッドプールを再利用することで、Taskの起動・終了に伴うシステムコストを削減している。