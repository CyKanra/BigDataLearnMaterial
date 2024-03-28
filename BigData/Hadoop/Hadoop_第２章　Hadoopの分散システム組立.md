# 分散大規模データ処理システム -- Hadoop-2

## 第２章　Hadoopの分散システム組み立て

**バージョン情報**

- HadoopはJava言語で書かれるため、Java環境（JVM）が必要で、 JDKバージョン：JDK8。
- 統一的にVMwareの仮想マシンを使用してLinuxの４つの節点を準備しておき、LinuxOS：Centos7。
- Hadoopの構築モードの選べ
  - 単一節点モード：単一節点モードは非クラスタ、本番環境ではこの方法は使用されません。
  - 単一節点模擬分布モード：単一節点、マルチスレッド（multi-thread）でクラスタの効果を模擬し、本番環境ではこの方法は使用されません。
  - **完全分散モード**：複数節点を持ち、実際の生産環境のモードと一致で、本番環境ではこのモードが推奨されます。

### 第１節　環境準備

**クラスタ環境**

　　Hadoopの分散システム組立の前にクラスタの環境設定が必要で、ここで大体の要点を紹介してだけ、詳しい設定手続きが「」文章に移って了解できます。

- クラスタは全て4つのサービスを備えてあり、hostファイルにIPアドレスのマッピングが図のように示します。

![image-20240328120459559](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328120459559.png)

- firewall／selinux等の動きが停止です。

```
systemctl status firewalld
```

![image-20240212160400972](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212160400972.png)

- 4つのサーバの間にパスワード不要で互いに登録して通信することができます。

- 4つのサーバの時刻が一致に保証します。

**Hadoop節点の配置**

| 対象 | centos1           | centos2                      | centos3     | centos4     |
| ---- | ----------------- | ---------------------------- | ----------- | ----------- |
| HDFS | NameNode,DataNode | SecondaryNameNode,DataNode   | DataNode    | DataNode    |
| YARN | NodeManager       | NodeManager、ResourceManager | NodeManager | NodeManager |

### 第２節　Hadoopの組み立て

#### 2.1 Hadoopインストール

- 専門にソフトを格納の目録を作成

```
mkdir -p /opt/bigdata/servers
```

- rzコマンドでHadoopパッケージをその目録にアップロード

![image-20240319151748247](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240319151748247.png)

![image-20240319151940599](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240319151940599.png)

- パッケージ解凍

```
tar -zxvf hadoop-2.9.2.tar.gz

#元パッケージを消除
rm hadoop-2.9.2.tar.gz
```

![image-20240319153716817](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240319153716817.png)

- Hadoopの環境引数を設定

```
vim /etc/profile

##HADOOP_HOME
export HADOOP_HOME=/opt/bigdata/servers/hadoop-2.9.2
export PATH=$PATH:$HADOOP_HOME/bin
export PATH=$PATH:$HADOOP_HOME/sbin

#環境引数を有効にし
source /etc/profile
```

![image-20240319154915394](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240319154915394.png)

- Hadoop検証

```
hadoop version
```

![image-20240319155128592](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240319155128592.png)

**Hadoop目録説明**

![image-20240319155707625](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240319155707625.png)

```
drwxr-xr-x 2 501 dialout    194 Nov 13  2018 bin
drwxr-xr-x 3 501 dialout     20 Nov 13  2018 etc
drwxr-xr-x 2 501 dialout    106 Nov 13  2018 include
drwxr-xr-x 3 501 dialout     20 Nov 13  2018 lib
drwxr-xr-x 2 501 dialout    239 Nov 13  2018 libexec
drwxr-xr-x 3 501 dialout   4096 Nov 13  2018 sbin
drwxr-xr-x 4 501 dialout     31 Nov 13  2018 share
```

- bin：Hadoop操作関連のコマンドがある目録。例えばhadoop、hdfsなど
- etc：Hadoopの設定ファイル目録。例えばhdfs-site.xml、core-site.xmlなど
- lib：Hadoopの本地依頼
- sbin：Hadoopクラスタの起動・停止関連のスクリプトやコマンドを保存
- share：Hadoopのjar、公式の案例jar、ドキュメントなどを含み

#### 2.2 Hadoopクラスタの設定

Hadoopクラスタの設定 = HDFSクラスタの設定 + MapReduceクラスタの設定 + Yarnクラスタの設定 

- HDFSクラスタの設定
  - JDKのパスをHDFSに設定する（hadoop-env.sh変更）
  - NameNodeの節点及びデータの格納目録を指定する（core-site.xml変更）
  - SecondaryNameNodeの節点を指定する（hdfs-site.xml変更）
  - DataNodeの従節点を指定する（etc/hadoop/slavesファイルを変更し、各節点の設定情報を1行に配置）
- MapReduceクラスタの設定
  - JDKのパスをMapReduceに設定する（mapred-env.sh変更）
  - MapReduce計算フレームワークがYarnフレームワークを使用するように指定する（mapred-site.xml変更） 
- Yarnクラスタの設定
  - JDKのパスをYarnに設定する（yarn-env.sh変更）
  - ResourceManagerの主節点が配置されている節点アドレスを指定する（yarn-site.xml変更）
  - NodeManagerの節点を指定する（slavesファイルの内容により決定される）

**HDFSクラスタの設定**

```
cd /opt/bigdata/servers/hadoop-2.9.2/etc/hadoop/
```

- slaves設定

```
vim slaves

#従節点アドレスを書き込み
centos1
centos2
centos3
centos4
```

![image-20240320153823596](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320153823596.png)

*注意：空白や改行は一切許可されない

- hadoop-env.sh設定

```
vim hadoop-env.sh

#Javaアドレスを添加
export JAVA_HOME=/opt/bigdata/servers/jdk1.8.0_231
```

![image-20240320142536348](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320142536348.png)

- core-site.xml設定

```
vim core-site.xml

#NameNode節点やデータ格納のアドレスを指定
<!-- NameNodeアドレスの指定 -->
<property>
	<name>fs.defaultFS</name>
	<value>hdfs://centos1:9000</value>
</property>
<!-- Hadoop運行状態に生まれたデータのアドレス -->
<property>
	<name>hadoop.tmp.dir</name>
	<value>/opt/bigdata/servers/hadoop-2.9.2/data/tmp</value>
</property>
```

![image-20240320145907974](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320145907974.png)

core-site.xml資料：[hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-common/core-default.xml](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-common/core-default.xml)

- hdfs-site.xml設定

```
vim hdfs-site.xml

#secondarynamenodeの指定
<!-- Hadoop補助節点の設定 -->
<property>
	<name>dfs.namenode.secondary.http-address</name>
	<value>centos2:50090</value>
</property>
<!-- 節点数 -->
<property>
	<name>dfs.replication</name>
	<value>4</value>
</property>
```

![image-20240320152915814](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320152915814.png)

hdfs-site.xml資料：[hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)

**MapReduceクラスタの設定**

- mapred-env.sh設定

```
vim mapred-env.sh

#Javaアドレスを添加
export JAVA_HOME=/opt/bigdata/servers/jdk1.8.0_231
```

![image-20240320160050824](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320160050824.png)

- mapred-site.xml設定

```
#ファイル名を変更
mv mapred-site.xml.template mapred-site.xml

vim mapred-site.xml

#MapReduce計算フレームワークがYarn上に運行を指定
<!-- MRがYarn上に運行の指定 -->
<property>
	<name>mapreduce.framework.name</name>
	<value>yarn</value>
</property>
```

![image-20240320165212007](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320165212007.png)

mapred-site.xml資料：[hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml](https://hadoop.apache.org/docs/r2.9.2/hadoop-mapreduce-client/hadoop-mapreduce-client-core/mapred-default.xml)

**Yarnクラスタの設定**

- yarn-env.sh設定

```
vim yarn-env.sh

#Javaアドレスを添加
export JAVA_HOME=/opt/bigdata/servers/jdk1.8.0_231
```

![image-20240320164330849](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320164330849.png)

- yarn-site.xml設定

```
vim yarn-site.xml

#ResourceMnagerの節点を指定
<!-- YARNのResourceManager節点の指定 -->
<property>
	<name>yarn.resourcemanager.hostname</name>
	<value>centos2</value>
</property>
<!-- Reducerデータを取得の方式 -->
<property>
	<name>yarn.nodemanager.aux-services</name>
	<value>mapreduce_shuffle</value>
</property>
```

![image-20240320165403701](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320165403701.png)

yarn-site.xml資料：[hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/hdfs-default.xml)

　　Hadoopのインストール目録の所有者と所有者グループの情報は黙認の501 dialoutです。Hadoopクラスタを操作するユーザーは仮想マシンのrootユーザーを使用しています。そのため、情報が混乱するのを避けるために、Hadoopのインストール目録の所有者と所有者グループを一致に変更します。

![image-20240320194216979](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320194216979.png)

```
chown -R root:root /opt/bigdata/servers/hadoop-2.9.2
```

![image-20240320194141302](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320194141302.png)

**Hadoopの分配**

　　Hadoop全体のファイルを他の３つのサーバに発送します。

```
cd /opt/bigdata/servers/

scp -r hadoop-2.9.2/ centos2:$PWD
scp -r hadoop-2.9.2/ centos3:$PWD
scp -r hadoop-2.9.2/ centos4:$PWD
```

![image-20240320195349159](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320195349159.png)

![image-20240320195429368](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240320195429368.png)

### 第３節　Hadoopクラスタの起動

 　　クラスタの起動方式は単節点起動とクラスタ起動の２種に分けます。単節点起動というのは一つ一つでサーバごとのHadoopサービスを起動する過程で、節点の回復、追加など使われます。クラスタ起動は全ての節点が一緒に起動、停止する場合に使われます。

**単節点起動**

HDFS単節点起動

- NameNodeの初期化

　　**特別注意：クラスタを初めて起動する場合は、NameNode節点で初期化する必要があります。初めてではない、又はNameNodeではない場合は、NameNodeの初期化を実行するのが全然ダメです！！！**

```
hadoop namenode -format
```

![image-20240321155033806](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240321155033806.png)

　　「Storage directory ～/name has been successfully formatted.」その様なメッセージが表れたら、Hadoopの初期化が成功になると言えます。二度と初期化操作を行いません。

![image-20240321155210816](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240321155210816.png)

　　Hadoopの初期化につれて「/opt/bigdata/servers/hadoop-2.9.2/data/tmp/dfs/name/current」下のファイルが生み出されました。データにの改修記録が一切ここに格納され、例えば、ある節点が最新データかどうかと確認がこれを根拠として検査する。若しその目録のファイルが初期化されたら、Hadoopサービスが再起動できません。

![image-20240321160224213](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240321160224213.png)

- HDFSのNameNode起動

```
cd /opt/bigdata/servers/hadoop-2.9.2/sbin/

hadoop-daemon.sh start namenode

#HDFSのプロセスを検査
jps
```

![image-20240325153741305](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325153741305.png)

- 全ての節点がDateNode起動

```
hadoop-daemon.sh start datanode
```

![image-20240325154733876](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325154733876.png)

![image-20240325155422023](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325155422023.png)

　　プロセスを検査してHDFSの起動を確認し、又はWebにHDFS画面に登録すると詳しい情報を見えます。

```
http://192.168.31.135:50070/dfshealth.html#tab-overview
```

![image-20240328115710036](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328115710036.png)

　　「live Nodes」をリンクして各節点の状態を表せます。全ての節点が順調に運行しているなら、ここまでHadoopのHDFS部分の設定や起動が完了しました。

![image-20240325160316339](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325160316339.png)

![image-20240325160356621](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325160356621.png)

YARN単節点起動

- centos2でresourcemanagerを起動

```
yarn-daemon.sh start resourcemanager
```

![image-20240325210729926](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325210729926.png)

- 全ての節点が以下のコマンドを実行

```
yarn-daemon.sh start nodemanager
```

![image-20240325211203100](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325211203100.png)

![image-20240325211229418](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240325211229418.png)

- 節点の停止

```
hadoop-daemon.sh stop namenode

hadoop-daemon.sh stop datanode

yarn-daemon.sh stop resourcemanager

yarn-daemon.sh stop nodemanager
```

**クラスタ起動**

HDFSクラスタ起動

　　**若し単節点起動の流れに初期化の操作を行わないなら、ここHDFSを初期化させるのが必要です。二度と初期化を実行が行けない、この点が注意します。**

```
hadoop namenode -format
```

- Hdfs起動

```
cd /opt/bigdata/servers/hadoop-2.9.2/sbin

start-dfs.sh

#Hdfs停止
stop-dfs.sh
```

![image-20240327161210120](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240327161210120.png)

YARNクラスタ起動

- centos2節点にYarn起動

```
start-yarn.sh

#Yarn停止
stop-yarn.sh
```

![image-20240327162244986](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240327162244986.png)

**コマンド纏め**

- 別々にHDFS起動／停止

```
hadoop-daemon.sh start / stop namenode / datanode / secondarynamenode
```

- YARN起動／停止

```
yarn-daemon.sh start / stop resourcemanager / nodemanager
```

- クラスタHDFS起動／停止

```
start-dfs.sh / stop-dfs.sh
```

- クラスタYARN起動／停止

```
start-yarn.sh / stop-yarn.sh
```

### 第４節　歴史記録検査の設定

　　終わりの計算任務には歴史のログを検査することが駄目で、別のサーバを通じてログ情報を見つかるしかありません。本節はこの歴史ログサーバを設定に関する内容を紹介します。

#### 4.1 歴史記録サーバの設定

- mapred-site.xml設定

```
cd /opt/bigdata/servers/hadoop-2.9.2/etc/hadoop/

vim mapred-site.xml

#以下の内容を追加
<!-- 歴史サーバのアドレス -->
<property>
	<name>mapreduce.jobhistory.address</name>
	<value>centos1:10020</value>
</property>
<!-- 歴史サーバのwebアドレス -->
<property>
	<name>mapreduce.jobhistory.webapp.address</name>
	<value>centos1:19888</value>
</property>
```

![image-20240328112548333](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328112548333.png)

- 他の節点に分配し、自動的に上書きして

```
scp mapred-site.xml centos2:$PWD
```

![image-20240328115024380](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328115024380.png)

- 歴史記録サーバの起動

```
cd /opt/bigdata/servers/hadoop-2.9.2/sbin

mr-jobhistory-daemon.sh start historyserver
```

![image-20240328115427835](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328115427835.png)

- JobHistoryのWebアドレス

```
http://192.168.31.135:19888/jobhistory
```

![image-20240328115849910](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328115849910.png)

　　今まで何も計算を実行しないので、表せる歴史記録がありません。

#### 4.2 ログの重合

　　MapReduceには一つ完全の任務を複数の小任務に分割されることがあり、そのために処理記録が各節点に散らばます。ログ重合の設定は一つの入口で完全のログ記録が検査できる結果を実現します。

- yarn-site.xml設定

```
cd /opt/bigdata/servers/hadoop-2.9.2/etc/hadoop/

vim yarn-site.xml

#以下の内容を追加
<!-- ログの重合機能 -->
<property>
	<name>yarn.log-aggregation-enable</name>
	<value>true</value>
</property>
<!-- ７日間の保留 -->
<property>
	<name>yarn.log-aggregation.retain-seconds</name>
	<value>604800</value>
</property>
```

![image-20240328154516760](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328154516760.png)

- 各節点に分配

```
scp yarn-site.xml centos2:$PWD
```

![image-20240328155931531](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240328155931531.png)

- 全てのサービスを再起動

```
stop-dfs.sh
stop-yarn.sh
mr-jobhistory-daemon.sh stop historyserver

start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver
```

### 第５節　クラスタのテスト
