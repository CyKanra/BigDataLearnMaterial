# HBase-分散型大規模非関係データベース -4

## 第４章　HBaseのクラスタ構築

　　HBaseのインストールが比較的に簡単で、添加する変数が少ない。唯一の注意して点は当のHBaseクラスタ構築が外部のZookeeperを依頼して、外部Zookeeperの連接を築く必要です。

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

　　hbase-env.shファイルを改修する。

```
# Java環境変量を添加
export JAVA_HOME=/opt/bigdata/servers/jdk1.8.0_231

# 外部Zookeeperを指定
export HBASE_MANAGES_ZK=FALSE
```

![image-20231212074452147](D:\OneDrive\picture\Typora\image-20231212074452147.png)

![image-20231212074544725](D:\OneDrive\picture\Typora\image-20231212074544725.png)

　　hbase-site.xmlファイルを改修する。

