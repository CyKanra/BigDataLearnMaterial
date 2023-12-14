# HBase-分散型大規模非関係データベース -4

## 第４章　HBaseのクラスタ構築

　　HBaseのインストールが比較的に簡単で、添加する変数が少ない。唯一の注意して点は当のHBaseクラスタ構築が外部のZookeeperを依頼し、外部Zookeeperの連接を築く必要です。

**サービスの分配**

| ソフト    | Centos1 | Centos2 | Centos3 | Centos4 |
| --------- | ------- | ------- | ------- | ------- |
| Hadoop    | √       | √       | √       | √       |
| Hive      |         | √       |         |         |
| MySQL     |         | √       |         |         |
| Hue       |         |         | √       |         |
| Flume     |         |         |         | √       |
| Zookeeper | √       | √       | √       | √       |
| HBase     | √       | √       | √       | √       |

**ダウンロード**

　　ソースの間に互換性と考えて、今回hbase-1.3.1バージョンを選べる。

ダウンロードURL：[Index of /dist/hbase/1.3.1 (apache.org)](https://archive.apache.org/dist/hbase/1.3.1/)

**本地サーバにアップロード**

　　rz命令でインストールのパッケージをアップロード。

![image-20231212070449653](D:\OneDrive\picture\Typora\image-20231212070449653.png)

![image-20231212070707249](D:\OneDrive\picture\Typora\image-20231212070707249.png)

**パッケージ解凍**

```
tar -zxvf hbase-1.3.1-bin.tar.gz

# 指定目録下に解凍
tar -zxvf hbase-1.3.1-bin.tar.gz -C /opt/bigdata/servers
```

**配置ファイル改修**

　　Hadoopの配置ファイルcore-site.xml 、hdfs-site.xmlをHBaseの目録下にconfファイルにコーヒーする。

```
# Hadoopファイルのアドレスをconfにマッピング
ln -s /opt/bigdata/servers/hadoop-2.9.2/etc/hadoop/core-site.xml /opt/bigdata/servers/hbase-1.3.1/conf/core-site.xml
ln -s /opt/bigdata/servers/hadoop-2.9.2/etc/hadoop/hdfs-site.xml /opt/bigdata/servers/hbase-1.3.1/conf/hdfs-site.xml
```

![image-20231212072512413](D:\OneDrive\picture\Typora\image-20231212072512413.png)

　　hbase-env.sh改修

```
# Java環境変量を添加
export JAVA_HOME=/opt/bigdata/servers/jdk1.8.0_231

# 外部Zookeeperを指定
export HBASE_MANAGES_ZK=FALSE
```

![image-20231212074452147](D:\OneDrive\picture\Typora\image-20231212074452147.png)

![image-20231212074544725](D:\OneDrive\picture\Typora\image-20231212074544725.png)

　　hbase-site.xml改修

```
<configuration>
	<!-- HDFSにのhbaseのアドレス -->
	<property>
		<name>hbase.rootdir</name>
		<value>hdfs://centos1:9000/hbase</value>
	</property>
	<!-- hbaseは分散式を指定 -->
	<property>
		<name>hbase.cluster.distributed</name>
		<value>true</value>
	</property>
	<!-- zkのアドレス指定、複数のは「,」で分割 -->
	<property>
		<name>hbase.zookeeper.quorum</name>
		<value>centos1:2181,centos2:2181,centos3:2181,centos4:2181</value>
	</property>
</configuration>
```

![image-20231213064334948](D:\OneDrive\picture\Typora\image-20231213064334948.png)

　　regionservers改修

```
# regionserver節点を指定
centos1
centos2
centos3
centos4
```

　　backup-mastersファイルを作成し、ある節点をBackup Masterとして指定する。HMasterが故障など発生して信号が中断する場合、Backup MasterがHMasterを代わりにクラスター任務を処理する。

```
# backup-masters作成
vim backup-masters

# 節点を指定
centos3
```

![image-20231213072659048](D:\OneDrive\picture\Typora\image-20231213072659048.png)

　　HBaseの環境変量を配置

```
vim /etc/profile

# 以下の内容を最後に添加
export HBASE_HOME=/opt/bigdata/servers/hbase-1.3.1
export PATH=$PATH:$HBASE_HOME/bin
```

![image-20231213073758497](D:\OneDrive\picture\Typora\image-20231213073758497.png)

　　HBaseの全てのファイルと環境変量ファイルを他のサーバに分配

```
scp -r hbase-1.3.1/ centos2:$PWD
scp -r hbase-1.3.1/ centos3:$PWD
scp -r hbase-1.3.1/ centos4:$PWD

scp -r /etc/profile centos2:$PWD
scp -r /etc/profile centos3:$PWD
scp -r /etc/profile centos4:$PWD

# 各サーバーに環境変量ファイルを有効になる
source /etc/profile
```

![image-20231213074641721](D:\OneDrive\picture\Typora\image-20231213074641721.png)

**HBaseクラスタの起動と停止**

　　HBase起動前にHadoopとZookeeperサービスが運行してるかどうかを確認し、その二つサービスはHBase順調に起動するの前提条件。

![image-20231213075257641](D:\OneDrive\picture\Typora\image-20231213075257641.png)

　　hbase-1.3.1/bin目録下のstart-hbase.shを実行する。どっちのサーバに命令を実行しても全体のクラスターを運行／停止できる。

```
# 起動
start-hbase.sh

# 停止
stop-hbase.sh
```

![image-20231213080026842](D:\OneDrive\picture\Typora\image-20231213080026842.png)

![image-20231213080105364](D:\OneDrive\picture\Typora\image-20231213080105364.png)

![image-20231213080118692](D:\OneDrive\picture\Typora\image-20231213080118692.png)

![image-20231213080138392](D:\OneDrive\picture\Typora\image-20231213080138392.png)

![image-20231213222253514](D:\OneDrive\picture\Typora\image-20231213222253514.png)

　　centos3サーバにのHMasterはBackup Mastersファイルに設定されたのHMasterのバックアップです。centos1には実際に全てのクラスターを管理するHMasterです。HMasterは最初起動する時に節点を選んで決める。

![image-20231213224430414](D:\OneDrive\picture\Typora\image-20231213224430414.png)

**外部訪問**

　　クラスターを起動したら、外部からHBaseサービスを訪問できる。IPアドレスはHMaster又はBackup Master運行してるサービスです。HBaseサービス画面は多い情報を表してあり、後の章節に詳しく紹介する。

```
#IP:16010
http://centos4:16010/
```

![image-20231214082007369](D:\OneDrive\picture\Typora\image-20231214082007369.png)

　　ここHBaseの高信頼性についてもっと説明しておく。高信頼性は二つの所で発現でき、Backup MastersやZookeeperが高信頼を提供してくれ、間に直接関係がない。前者はHMaster失効になる場合に控えとして準備する。後者は、HMaster下の各Region Serverが故障を発生する時に、HMasterに通知にしてHMasterが失効のRegion Serverの資源を再分配し、それもZookeeperに節点情報を改修する。Backup MastersやZookeeperの一番区別はZookeeper本体を控えとして差し替えられない、HMasterに調整的な役割を果たすだけ。
