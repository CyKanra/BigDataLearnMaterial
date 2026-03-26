# ビッグデータ高速計算エンジンSpark-2

# 第２章　Sparkの分散システム組み立て

公式URL：[Apache Spark™ - Unified Engine for large-scale data analytics](https://spark.apache.org/)

ドキュメントURL：[Overview - Spark 4.1.1 Documentation](https://spark.apache.org/docs/latest/)

ダウロードURL：[Downloads | Apache Spark](https://spark.apache.org/downloads.html)

　今回使うSparkバージョンは、spark-2.4.5を選んでいる。　`spark-2.4.5-bin-without-hadoop-scala-2.12.tgz`インストールパッケージと`spark-2.4.5.tgz` ソースコード2つをダウンロードする必要です。

![image-20260320121043768](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320121043768.png)

spark-2.4.5ダウロード：[Apache Archive Distribution Directory](https://archive.apache.org/dist/spark/spark-2.4.5/)

spark-2.4.5ドキュメント：[Overview - Spark 2.4.5 Documentation](https://archive.apache.org/dist/spark/docs/2.4.5/)

　何でそのバージョンのインストールパッケージを選ぶ理由をちょっと説明する。

　先ず、Sparkの講解はHadoopを基づいているので、Hadoop文字が含めるインストールパッケージを選んだ。今使っているHadoopバージョンは2.9で、上の2.6、2.7のHadoopバージョンを除く。

　あと、scala-2.12が含めるのを選ぶ。下のScalaバージョンを書かないのはscala-2.11バージョンです。2.4.5ドキュメントにscala-2.11バージョンは Spark 3.0に捨てられると書いていたので、もっと高いscala-2.12を選ぶほうがいい。

![image-20260320125454319](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320125454319.png)

　その2つダウロードしたら、次はSparkの組立を紹介する。

![image-20260320132853997](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320132853997.png)

## 第１節　Sparkの組み立て

### 1.1　Sparkのインストール

- rzコマンドでインストールパッケージをアップロードする。

![image-20260320133857103](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320133857103.png)

- パッケージを解凍して短いファイル名に変更する。

```
tar zxvf spark-2.4.5-bin-without-hadoop-scala-2.12.tgz

mv spark-2.4.5-bin-without-hadoop-scala-2.12 spark2.4.5
```

![image-20260320134034532](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320134034532.png)

![image-20260320134543638](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320134543638.png)

- 環境変数の設定

```
vim /etc/profile

export SPARK_HOME=/opt/bigdata/servers/spark-2.4.5
export PATH=$PATH:$SPARK_HOME/bin:$SPARK_HOME/sbin

source /etc/profile

#入力して環境変数の設定が効くかどうか確認する
$SPARK_HOME
```

![image-20260320140951056](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320140951056.png)

![image-20260320135326435](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320135326435.png)

　続く前にSpark下のディレクトリを簡単に紹介する。

![image-20260320143457716](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320143457716.png)

　binファイルにSparkの色々実行スクリプトファイルが含める。`.cmd`接尾のファイルはWindows環境の実行ファイルです。数量がちょっと多いんで、はっきりしたいなら全て削除しても問題ない。

```
rm -rf *.cmd
```

![image-20260320144116283](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320144116283.png)

　`sbin`にはSpark起動、停止などの実行ファイルが含める。Hadoopを知っているなら`start-dfs.sh`、`start-yarn.sh`などクラスタの起動ファイルと類似する。

![image-20260320203119448](D:\OneDrive\picture\Typora\BigData\Spark\image-20260320203119448.png)

　jars、python、R、yarnに対応の依頼が含める。今はJavaの依頼（jars）のみを使うんです。

　`examples`フォルダにSparkの案例を提供してくれる。その対応のテストデータが`data`フォルダに格納される。Sparkはただ計算フレームワークで、Spark自体はデータを持ってなく、HDFSからデータを分割して計算する。その計算結果を返してHDFSに格納する。

　configフォルダにSparkに関する設定ファイルは全てここに置く。

![image-20260321124621745](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321124621745.png)

- 設定の変更

　以上の全てファイルの`.template`接尾を削除する。

![image-20260321125157720](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321125157720.png)

**savles**

　全てのノードのIPアドレスをsavlesに追加する。

```
centos1
centos2
centos3
centos4
centos5
```

![image-20260321125825914](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321125825914.png)

**spark-env.sh**

　環境変数を設定する。元々`spark-2.4.5-bin-without-hadoop`インストールパッケージを使って、自動的にシステムの環境変数を読み込むこともないし、Java、Hadoopなどの格納アドレスは単独にここに設定する必要です。

```
export JAVA_HOME=/opt/bigdata/servers/jdk1.8.0_231
export HADOOP_HOME=/opt/bigdata/servers/hadoop-2.9.2
export HADOOP_CONF_DIR=/opt/bigdata/servers/hadoop-2.9.2/etc/hadoop
export SPARK_DIST_CLASSPATH=$(/opt/bigdata/servers/hadoop-2.9.2/bin/hadoop classpath)
export SPARK_MASTER_HOST=centos1
export SPARK_MASTER_PORT=7077
```

　デフォルトPORT番号が`7077`で、エラーログに7077キーワードが出てくるならすぐにSparkが何か問題あるを知っているべき。

![image-20260321131215229](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321131215229.png)

**spark-defaults.conf**

　ファイルに設定の案例があって、上に変更していい。或いは下の内容をコピーして書く。

　このファイルに設定選択肢はデフォルト値があり、spark.masterを除いて他の設定を変更しなくてそのまま置いても問題ない。

```
spark.master                     spark://centos1:7077
spark.eventLog.enabled           true
spark.eventLog.dir               hdfs://centos1:9000/spark-eventlog
spark.serializer                 org.apache.spark.serializer.KryoSerializer
spark.driver.memory              512m
# spark.executor.extraJavaOptions  -XX:+PrintGCDetails -Dkey=value -Dnumbers="one two three"
```

![image-20260321134857948](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321134857948.png)

- Sparkの配布

　Sparkの設定が一応終わり、他のノードに配布する。念のため、各ノードを確認するを忘れないよ。

```
scp -r spark-2.4.5/ centos2:$PWD
scp -r spark-2.4.5/ centos3:$PWD
scp -r spark-2.4.5/ centos4:$PWD
scp -r spark-2.4.5/ centos5:$PWD
```

![image-20260321135643005](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321135643005.png)

- Sparkクラスタの起動

　`sbin`フォルダに入って`start-all.sh`を運行する。

```
start-all.sh
```

![image-20260321141119815](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321141119815.png)

　SparkサービスURL：http://192.168.31.135:8080/

　自分SparkマスタのIPアドレスに替えてアクセスする。画面が下のように現れてどのポイントも一つも取りこぼしていないんで、ある意味はSparkクラスタの組立が成功したと言える。

![image-20260321141337621](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321141337621.png)

　`start-all.sh`を運行するとき、必ず所在アドレスを付けて実行する。前に任意のところで`start-dfs.sh`を実行するみたいことができない。

```
/opt/bigdata/servers/spark-2.4.5/sbin/start-all.sh
```

![image-20260321164337769](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321164337769.png)

　理由はHadoopの`start-all.sh`とSparkの`start-all.sh`は互いに衝突された。誰が前に誰を先に実行する。解決方法はHadoopのstart-all.shとstop-all.shをstart-all-hadoop.sh 、 stop-all-hadoop.shに変更して解決できる。

- クラスタのテスト

　Sparkクラスタサービスが正常に起動されるけど、正常にタスクを処理するかどうかまだ確認する必要です。実際にSpark公式の提供する案例を運行する。

```
#Hadoopのサービスあるかどうか確認する
jsp

#sbinディレクトリに実行する
run-example SparkPi 10
```

![image-20260321221338381](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321221338381.png)

　エラー発生した。`hdfs://centos1:9000/spark-eventlog`Sparkログを格納するディレクトリが見つけない。

```
hdfs dfs -mkdir /spark-eventlog
```

![image-20260321222003082](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321222003082.png)

　正常に起動していた。

![image-20260321222250445](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321222250445.png)

![image-20260321222316118](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321222316118.png)

![image-20260321222444504](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321222444504.png)

　上述の方法を除いて、`spark-shell`を実行するのはクラスタが正常に起動されるかどうかをテストできる。

![image-20260321223112547](D:\OneDrive\picture\Typora\BigData\Spark\image-20260321223112547.png)
