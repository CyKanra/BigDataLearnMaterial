# ビッグデータ高速計算エンジンSpark-2

# 第２章　Sparkの分散システム組み立て

## 第２節　Sparkの部署モード

　Apache Sparkには、いくつかの実行モードがある。

**ローカルモード**

　最も簡単なモードで、すべての処理が1台のマシンのJVM上で実行される。

**疑似分散モード**

　1台のマシン上でクラスター環境を擬似的に再現するモード。関連するプロセスもすべて同じマシン上で動く。

**分散モード**

　実際のクラスター上で動作するモード。主に次の3種類がある：

- Standalone：Spark標準のリソース管理を使用
- YARN：YARNのリソース管理を使用
- Mesos：Mesosのリソース管理を使用

### 2.1　ローカルモード

　ローカルモードは単一マシン上で動作し、Sparkクラスタを依存する必要がない。主にテストや検証用に使われる。最も簡単な実行方式で、すべての処理は1台のマシンのJVM内で行われる。

　このモードでは、1台のマシン上の複数スレッドを使って分散処理を疑似的に再現する。そのため、開発したアプリのロジック確認によく使われる。

　設定も簡単で、Sparkを解凍して基本設定を少し調整するだけで使える。MasterやWorkerの起動は不要で、Hadoopサービスも基本的には必要ない（HDFSを使う場合を除く）。

ローカルモードの起動オプション：

- local：1スレッドで実行
- local[N]：N個のスレッドで実行
- local[\*]：CPUの全コアを使用
- local[N, M]：Nは使用するコア数、Mはタスクの最大失敗許容回数

※ Mを指定しない場合は、デフォルトで1回となる。

- HadoopとSparkサービス停止

```
#Hadoop停止
stop-dfs.sh

#Sparkサービス停止
#もしHadoopのstop-all.sh名称を変わったら直接に実行できる
stop-all.sh

#検証
jps
```

![image-20260405115026196](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405115026196.png)

- Sparkローカルモード起動

```
spark-shell --master local[*]
```

![image-20260405111644720](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405111644720.png)

　起動が失敗した。原因はspark-defaults.confファイルにSparkのログがHDFSに格納されるのが設定され、HDFSサービスを見つけられないでエラーが発生した。

![image-20260405112123189](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405112123189.png)

　解決方法は下の2行を註解にしてHDFSを使わなくさせる。成功したらSparkSubmitスレッドが現れる。

```
#spark.eventLog.enabled           true
#spark.eventLog.dir               hdfs://centos1:9000/spark-eventlog
```

![image-20260405114658527](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405114658527.png)

![image-20260405114735997](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405114735997.png)

![image-20260405115052991](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405115052991.png)

### 2.2　疑似分散モード

　疑似分散モードは、1台のマシン上でクラスター環境を擬似的に再現するモード。関連するプロセスもすべて同じマシン上で動作する。

**起動形式**

- `local-cluster[N, cores, memory]`
  - **N**：仮想的に用意するWorker（Slave）ノードの数
  - **cores**：各Workerが持つCPUコア数
  - **memory**：各Workerに割り当てるメモリ容量

```
spark-shell --master local-cluster[4,2,1024]
```

　クラスター用のリソース管理サービスを起動する必要はない。そうして、`local-cluster[4,2,1024]`変数の値を大きくすぎると資源が足りなくなるので注意する。

![image-20260501105820104](D:\OneDrive\picture\Typora\BigData\Spark\image-20260501105820104.png)

![image-20260501122216498](D:\OneDrive\picture\Typora\BigData\Spark\image-20260501122216498.png)

　jps確認すると、SparkSubmitが1つ、CoarseGrainedExecutorBackendが4つ起動していることが分かる。SparkSubmitはこの場合、Client、Driver、資源管理の役割を兼ねる中心的なプロセスとして動作する。一方、4つのCoarseGrainedExecutorBackendは実際に処理を並列実行するプロセス。

　でも、このモードはあんまりお勧めしない。使いところは少ないし、不具合も多い。

### 2.3　クラスタモード--Standalone

　分散環境で実行してこそ、分散処理のメリットが活きるローカルモードと違い、事前にSparkのMasterとWorkerを起動しておく必要がある。このモードはYARNを使わない。Hadoopのサービスは使わないなら基本不要です。　

参考： http://spark.apache.org/docs/latest/spark-standalone.html

#### 2.3.1　Standaloneモードの起動

![image-20260502100747469](D:\OneDrive\picture\Typora\BigData\Spark\image-20260502100747469.png)

　ここSparkログ設定を有効にすると、HDFSを使うが必要です。

```
#Spark起動
#単独ノードのMaster、Workerの起動
sbin/start-master.sh / sbin/stop-master.sh
sbin/start-slave.sh / sbin/stop-slave.sh

#全体のWorkerの起動
sbin/start-slaves.sh / sbin/stop-slaves.sh

#Master、Worker一括の起動
sbin/start-all.sh / sbin/stop-all.sh
```

![image-20260503104045490](D:\OneDrive\picture\Typora\BigData\Spark\image-20260503104045490.png)

　`start-slave.sh`コマンドは後にmasterのアドレス`spark://hostname:port`変数を付ける必要です。後に幾つか変数設定`[options]`が提供される。

![image-20260503104126506](D:\OneDrive\picture\Typora\BigData\Spark\image-20260503104126506.png)

![image-20260503105117194](D:\OneDrive\picture\Typora\BigData\Spark\image-20260503105117194.png)

　`start-slaves.sh`全てのWorkerノードを起動し、変数は不要。

![image-20260503112314671](D:\OneDrive\picture\Typora\BigData\Spark\image-20260503112314671.png)

　 `start-all.sh`と`stop-all.sh`はMasterとWorkerを一緒に起動/停止でき、変数は不要。

![image-20260503122940755](D:\OneDrive\picture\Typora\BigData\Spark\image-20260503122940755.png)

#### 2.3.2　Standalone実行モード

Standalone実行モードはClusterとClientに分けて、不同点はDriverがどこで動くことです。

- Clientモード（デフォルト）：Driverは コマンドを実行した側（Client）で動く。そのため、実行結果をその場で確認できる。対話的な処理やデバッグ向き。	
- Clusterモード ：DriverはSparkクラスター内で動く。クライアント側からは直接結果が見えない。本番環境向き。

![image-20260505102550464](D:\OneDrive\picture\Typora\BigData\Spark\image-20260505102550464.png)

**Clientモード**

```
spark-submit --class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples_2.12-2.4.5.jar 1000
```

　円周率の計算結果は3.1415852714158525が出ているので、システムは正常に動いている。もし後のエラーがあったらSparkの自身の不整備で、放っておいていいんです。

![image-20260716060226367](D:\OneDrive\picture\Typora\BigData\Spark\image-20260716060226367.png)

　計算中にプロセスを検査して`SparkSubmit`と`CoarseGrainedExecutorBackend`2つ増えた。

　`CoarseGrainedExecutorBackend`のは実際に計算を実行してWorkerから生むExecutorプロセスです。`SparkSubmit`はアカウントがタスクを起動するプロセスです。Clientモードで`SparkSubmit`がClient端に運行されているので、実行ログはコンソールに表示される。最後の結果も`SparkSubmit`のDriverへ返却して画面に表示する。

![image-20260716055607479](D:\OneDrive\picture\Typora\BigData\Spark\image-20260716055607479.png)

```
spark-submit
    ↓
Driver起動（spark-submit側）
    ↓
DriverがMasterにExecutorを要求
    ↓
MasterがWorkerを選択
    ↓
Worker上でExecutor起動
    ↓
DriverがExecutorへTaskを送る
    ↓
ExecutorがTask実行
    ↓
結果をDriverへ返す
```

**Clusterモード**

```
spark-submit --class org.apache.spark.examples.SparkPi --deploy-mode cluster $SPARK_HOME/examples/jars/spark-examples_2.12-2.4.5.jar 1000
```

　運行ログはコンソールに表示されてない。

![image-20260717065555614](D:\OneDrive\picture\Typora\BigData\Spark\image-20260717065555614.png)

　Client端に起動される`SparkSubmit`プロセスは、先ずApplicationをMasterへ送って、Masterが指示を受けるとMasterがWorkerへ通知してExecutorを起動する。その後、`SparkSubmit`が消えている。

![image-20260717065624144](D:\OneDrive\picture\Typora\BigData\Spark\image-20260717065624144.png)

　運行流れは大体下記そうです。`SparkSubmit`はアプリを提出する役割を完成したら終了される。

```
SparkSubmit
     ↓
ApplicationをMasterへ提出
     ↓
SparkSubmit終了
```

```
spark-submit
    ↓
ApplicationをMasterへ提出
    ↓
MasterがWorkerを選択
    ↓
Worker上でDriverWrapper起動
    ↓
Driver起動
    ↓
DriverがMasterにExecutorを要求
    ↓
MasterがWorkerを選択
    ↓
Worker上でExecutor起動
    ↓
DriverがExecutorへTaskを送る
    ↓
Task実行
    ↓
結果をDriverへ返す
```

　ClientモードとClusterモード最大区別は、Clusterモードが下記の流れがある。ClientモードはWorkerを選んでどこにDriverを起動する過程が必要ない。当のノードで直接にDriverを起動する。Clusterモードは先にDriverを起動するノード（Worker）を選んでおく。後の実際のExecutor作業プロセスも1つのノード（Worker）を選ぶ。つまり、Clusterモードに2回のWorker選びがあり、Clientモードはただ後の1つ。

```
ApplicationをMasterへ提出
    ↓
MasterがWorkerを選択
    ↓
Worker上でDriverWrapper起動
```

　`$SPARK_HOME/work/`ところに計算結果`app-20260722174227-0004`を見つけられる。具体的な計算結果は`stdout`に記録される。

![image-20260727064818054](D:\OneDrive\picture\Typora\BigData\Spark\image-20260727064818054.png)

![image-20260727064855241](D:\OneDrive\picture\Typora\BigData\Spark\image-20260727064855241.png)

　他の方法は、サイトに計算結果も見られる。

![image-20260727065018521](D:\OneDrive\picture\Typora\BigData\Spark\image-20260727065018521.png)

#### 2.3.3　History Server

　History Serverはログ集約機能です。このを使って各ノードのログをHDFSに保存して一元管理する。

　さっき円周率計算案例中に結果は計算を実行するノードのみに格納されている。もしこのノード潰れるならログもなくなる。完全、便利など考えるために統一にHDFSサービスに管理する。

　下の内容を`spark-defaults.conf`と`spark-env.sh`設定ファイルに追加する。

```
# spark-defaults.conf
# history server
spark.eventLog.enabled　true
spark.eventLog.dir　hdfs://centos1:8020/spark-eventlog
spark.eventLog.compress　true

# spark-env.sh
export SPARK_HISTORY_OPTS="-Dspark.history.ui.port=18080 -Dspark.history.retainedApplications=50 -Dspark.history.fs.logDirectory=hdfs://centos1:9000/spark-eventlog"
```

![image-20260807065102530](D:\OneDrive\picture\Typora\BigData\Spark\image-20260807065102530.png)

![image-20260807065137705](D:\OneDrive\picture\Typora\BigData\Spark\image-20260807065137705.png)

　設定が完了しては各ノードに割り当てる。

![image-20260807065328126](D:\OneDrive\picture\Typora\BigData\Spark\image-20260807065328126.png)

　更新したSparkのサービスを改めてを起動する。その前にHDFSサービス運行中かどうか確認する。

![image-20260809103340480](D:\OneDrive\picture\Typora\BigData\Spark\image-20260809103340480.png)

　Historyserverサービスが単独に起動する必要がある。成功してはログサービスのプロセスが見える。

WEB側IPアドレス： http://centos1:18080/

```
$SPARK_HOME/sbin/start-history-server.sh
```

![image-20260809103641607](D:\OneDrive\picture\Typora\BigData\Spark\image-20260809103641607.png)

![image-20260809103837810](D:\OneDrive\picture\Typora\BigData\Spark\image-20260809103837810.png)

### 2.4　クラスタモード--Yarn

　SparkをYARNモードで実行する場合、HDFSとYARNのサービスを起動する必要がある。一方、Standaloneモードで使用していた Spark Master / Workerは停止する。tandaloneとYARNはそれぞれ別のリソース管理方式なので、基本的に同時には使用しない。

参考： http://spark.apache.org/docs/latest/running-on-yarn.html

#### 2.4.1　Yarnモードの起動

　Standaloneモードの対応のサービスを停止してHadoopのHDFS、Yarn、Historyserverサービスを起動する。StandaloneとYarnが互いに競合している。

```
#起動コマンド
start-dfs.sh
start-yarn.sh

#停止コマンド
stop-dfs.sh
stop-yarn.sh
```

![image-20260811103652747](D:\OneDrive\picture\Typora\BigData\Spark\image-20260811103652747.png)

![image-20260811103746556](D:\OneDrive\picture\Typora\BigData\Spark\image-20260811103746556.png)

　次は、Hadoopのyarn-site.xmlファイルに追加の設定がある。仮想マシンでメモリ不足が起こることがあって当のタスクが強制終了されている。このような状況を避けるため、制限を解除する必要がある。

```
<property>
	<name>yarn.nodemanager.pmem-check-enabled</name>
	<value>false</value>
</property>

<property>
	<name>yarn.nodemanager.vmem-check-enabled</name>
	<value>false</value>
</property>
```

**yarn.nodemanager.pmem-check-enabled**

　各タスクが使用している物理メモリ（Physical Memory）を監視するかどうかを設定する。使用量が割り当てられた上限を超えた場合、そのタスクは強制終了される。デフォルトは `true`。

**yarn.nodemanager.vmem-check-enabled**

　各タスクが使用している仮想メモリ（Virtual Memory）*を監視するかどうかを設定する。使用量が割り当てられた上限を超えた場合、そのタスクは強制終了される。デフォルトは `true`。

![image-20260811150820813](D:\OneDrive\picture\Typora\BigData\Spark\image-20260811150820813.png)

　追加したyarn-site.xmlファイルを各ノードに配布してYarnサービスを再起動する。

![image-20260811151206144](D:\OneDrive\picture\Typora\BigData\Spark\image-20260811151206144.png)

　Spark方は、spark-env.shにHadoopの設定ファイルのアドレスを追加する。

```
# spark-env.sh
export HADOOP_CONF_DIR=/opt/bigdata/servers/hadoop-2.9.2/etc/hadoop
```

![image-20260811162200647](D:\OneDrive\picture\Typora\BigData\Spark\image-20260811162200647.png)

　次は、お勧め設定です。

　SparkのプロジェクトはいるjarパッケージをHDFSにアップロードしておく。そのステップをしなくても実行中にアップロードし、時間がかかるので先に準備しておいたほうがいい。

```
spark.yarn.jars hdfs:///spark-yarn/jars/*.jar
```

![image-20260813102250537](D:\OneDrive\picture\Typora\BigData\Spark\image-20260813102250537.png)

```
# SparkのjarパッケージをHDFSにアップロード
hdfs dfs -mkdir -p /spark-yarn/jars/
cd $SPARK_HOME/jars
hdfs dfs -put * /spark-yarn/jars/
```

![image-20260813112610315](D:\OneDrive\picture\Typora\BigData\Spark\image-20260813112610315.png)

#### 2.3.2 Spark-Yarn実行モード

YARNモードも2つの実行方式がある。

- yarn-client：Driverはクライアント側で動作して、実行結果をその場で確認しやすく、対話的な操作やデバッグに向いている。
- yarn-cluster：DriverはYARNのApplicationMaster側で動作し、クライアントから切り離して実行できる。本番環境での実行に向いている

。

**yarn-clientモード**

　計算結果はコンソールに見える。

```
# client
spark-submit --master yarn --deploy-mode client --class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples_2.12-2.4.5.jar 2000
```

![image-20260813121154364](D:\OneDrive\picture\Typora\BigData\Spark\image-20260813121154364.png)

**yarn-clusterモード**

　計算結果がコンソールに表示されない。

```
# cluster
spark-submit --master yarn --deploy-mode cluster --class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples_2.12-2.4.5.jar 2000
```

![image-20260813125924705](D:\OneDrive\picture\Typora\BigData\Spark\image-20260813125924705.png)

　因みに、Yarnモードは関する情報がサイトに現れなく、これはStandalone専用のサイトです。

![image-20260813133049808](D:\OneDrive\picture\Typora\BigData\Spark\image-20260813133049808.png)

#### 2.4.3　Yarn ResourceManagerとSpark History Server連携

　先のyarn-clusterモードの計算の情報はサイトに見えなく、Standalone専用の用からです。今はYarnのResourceManagerとSparkのHistoryServerを連携してResourceManager管理画面にSparkのログ記録を見えるようにする。

　spark-default.confファイルに下記の設定を追加する。

　これ必須設定じゃない、設定しなくてもSparkのログがHDFSに格納されてYarnにYarnのResourceManagerに見える。区別のは、YarnからSparkのHistoryServerに移動できるように、そのURLアドレスをYarnに知らせる。

```
spark.yarn.historyServer.address centos1:18080
spark.history.ui.port            18080
```

![image-20260817062206858](D:\OneDrive\picture\Typora\BigData\Spark\image-20260817062206858.png)

　設定ファイルを他のノードに配布することを忘れないように、後SparkのHistoryServerサービスを起動する。

```
start-history-server.sh
stop-history-server.sh
```

![image-20260817063848653](D:\OneDrive\picture\Typora\BigData\Spark\image-20260817063848653.png)

　ジョブを投入して、動作を確認してみる。

```
spark-submit --master yarn --deploy-mode cluster --class org.apache.spark.examples.SparkPi $SPARK_HOME/examples/jars/spark-examples_2.12-2.4.5.jar 2000
```

![image-20260817064824837](D:\OneDrive\picture\Typora\BigData\Spark\image-20260817064824837.png)

　Spark対応ログ情報は「YarnサービスURL：http://192.168.31.136:8088/cluster」ページに見つけられる。

![image-20260817065916350](D:\OneDrive\picture\Typora\BigData\Spark\image-20260817065916350.png)

![image-20260818065757538](D:\OneDrive\picture\Typora\BigData\Spark\image-20260818065757538.png)

　下の`History`リンクを押下してSparkのHistoryServerに入れる。先spark-default.confの設定はそういう役割です。

![image-20260818065854586](D:\OneDrive\picture\Typora\BigData\Spark\image-20260818065854586.png)

### 2.5　開発環境の構築IDE

　次の章節はSparkのコーディングを紹介する。その前にScala言語の環境を準備しなきゃいけない。

　Scala開発環境準備はJavaみたいJDKをインストールする必要ない、既存のプロジェクトに関連のMAVEN依頼を追加して運行できる。

　先ず、IDEA開けてPluginsにScalaプラグインを探してインストールする。

![image-20260822125836682](D:\OneDrive\picture\Typora\BigData\Spark\image-20260822125836682.png)

　既存のプロジェクトを開けて、右クリックして「Add Framework Support...」入ってScalaフレームワークを添加する。

![image-20260822160935420](D:\OneDrive\picture\Typora\BigData\Spark\image-20260822160935420.png)

![image-20260822161212115](D:\OneDrive\picture\Typora\BigData\Spark\image-20260822161212115.png)

　設定が問題ないなら「Scala Class」ファイルタイプが見つかれる。

![image-20260822183024206](D:\OneDrive\picture\Typora\BigData\Spark\image-20260822183024206.png)

　次は`pom.xml`にScalaに関する依頼とScalaプラグインを追加する。

　仮想マシンにSparkバージョンは`2.4.5`なので、Sparkの依頼は`2.4.5`を使う。Scalaのライブラリは`2.12.10`を使う。

```
# コード依頼
<dependency>
	<groupId>org.scala-lang</groupId>
	<artifactId>scala-library</artifactId>
	<version>2.12.10</version>
</dependency>
<dependency>
	<groupId>org.apache.spark</groupId>
	<artifactId>spark-core_2.12</artifactId>
	<version>2.4.5</version>
</dependency>
```

```
 # Scalaプラグイン
 <plugin>
    <groupId>net.alchim31.maven</groupId>
    <artifactId>scala-maven-plugin</artifactId>
    <version>3.2.2</version>
    <executions>
        <execution>
            <id>scala-compile-first</id>
            <phase>process-resources</phase>
            <goals>
                <goal>add-source</goal>
                <goal>compile</goal>
            </goals>
        </execution>
        <execution>
            <id>scala-test-compile</id>
            <phase>process-test-resources</phase>
            <goals>
                <goal>testCompile</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

　Mavenの依存関係とSDKの間で少し競合があるので、プロジェクトに元のScalaのSDKを取り去る。

![image-20260822190417074](D:\OneDrive\picture\Typora\BigData\Spark\image-20260822190417074.png)

　次はScalaコードを実行して、開発環境が正常に使えるか確認する。

　Scalaの`object`タイプファイルを新規し、下図見えて赤いエラーがあっては、JDKバージョン高すぎるわけかもしれ。

![image-20260825065228999](D:\OneDrive\picture\Typora\BigData\Spark\image-20260825065228999.png)

![image-20260822190844040](D:\OneDrive\picture\Typora\BigData\Spark\image-20260822190844040.png)

全体プロジェクトJDK環境バージョンを17から1.8に変更する。念のため17のを取り去る。

![image-20260824064127493](D:\OneDrive\picture\Typora\BigData\Spark\image-20260824064127493.png)

![image-20260825065958669](D:\OneDrive\picture\Typora\BigData\Spark\image-20260825065958669.png)

　赤いワーニングなくなって任意のローカルファイルを読み込み、ファイル内容は正常にコンソールに表示されたらScalaの環境設定に問題はないということです。

```
package com.scalapractice

import org.apache.spark.rdd.RDD
import org.apache.spark.{SparkConf, SparkContext}

object WordCount {

    def main(args: Array[String]): Unit = {
      val conf = new SparkConf().setMaster("local").setAppName("WordCount")
      val sc = new SparkContext(conf)
      //val lines: RDD[String] = sc.textFile("hdfs://centos1:9000/wcinput/wc.txt")
      val lines: RDD[String] = sc.textFile("D:/wc.txt")
      // val lines: RDD[String] = sc.textFile("data/wc.dat")
      lines.flatMap(_.split(" ")).map((_, 1)).reduceByKey(_+_).collect().foreach(println)
      sc.stop()
  }
}
```

![image-20260826064545433](D:\OneDrive\picture\Typora\BigData\Spark\image-20260826064545433.png)

![image-20260826064834250](D:\OneDrive\picture\Typora\BigData\Spark\image-20260826064834250.png)
