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

　　Hadoopの分散システム組立の前にクラスタの環境設定が必要で、ここで大体の要点を紹介し、詳しい設定手続きが「」文章に移って了解できます。

- クラスタは全て4つのサービスを備えてあり、hostファイルにIPアドレスのマッピングが図のように示します。

![image-20240212154807536](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212154807536.png)

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

#### Hadoopインストール

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

#### Hadoopクラスタの設定

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

 
