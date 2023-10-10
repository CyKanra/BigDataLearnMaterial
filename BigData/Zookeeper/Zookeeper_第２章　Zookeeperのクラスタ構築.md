# 分散型協調サービスフレームワーク -- Zookeeper-1

## 第２章　Zookeeperのクラスタ構築

**Zookeeper模式**

　　ZooKeeperのインストール方法には、3つの異なる模式がある。それは、単一節点模式、クラスターモード、及び擬似クラスタ模式です。

- 単一節点模式: ZooKeeperは1台のサーバー上でのみ実行され、主にテスト環境に適している。

- 擬似クラスター模式: これは1台のサーバー上で複数のZooKeeperを実行する方法です。
- クラスタ模式: ZooKeeperは複数のサーバーで実行され、主に本番環境向けであるんです。

**サービスの分配**

| ソフト    | Centos1 | Centos2 | Centos3 | Centos4 |
| --------- | ------- | ------- | ------- | ------- |
| Hadoop    | √       | √       | √       | √       |
| Hive      |         | √       |         |         |
| MySQL     |         | √       |         |         |
| Hue       |         |         | √       |         |
| Flume     |         |         |         | √       |
| Zookeeper | √       | √       | √       | √       |

**ダウンロード**　

　　本章は勿論クラスタ模式を選べて紹介し、Zookeeperのバージョンがzookeeper-3.4.14版を使用するつもり。

ダウンロードURL：[Index of /dist/zookeeper (apache.org)](http://archive.apache.org/dist/zookeeper/)

**本地サーバにアップロード**

　　rz命令でインストールのパッケージをアップロード。

![image-20231009151708691](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231009151708691.png)

![image-20231009151847691](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231009151847691.png)

**パッケージ解凍**

```
tar zxfv zookeeper-3.4.14.tar.gz
```

**配置ファイル改修**

　　Zookeeperの保存目録やログ目録を添加する。図に目録下が色々なファイルがあるのはZookeeperも起動したため、最初は全部空です。

```
# zk保存目録を作成
mkdir -p /opt/bigdata/servers/zookeeper-3.4.14/data
# zkログ目録を作成
mkdir -p /opt/bigdata/servers/zookeeper-3.4.14/data/logs
```

![image-20231009155650712](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231009155650712.png)

　　zookeeper-3.4.14/conf目録に入って、配置ファイルを改修する。

```
# 修改zk配置文件
cd /opt/bigdata/servers/zookeeper-3.4.14/conf
# 文件改名
mv zoo_sample.cfg zoo.cfg
```

![image-20231009162500649](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231009162500649.png)

　　zoo.cfgに以下の内容を添加する。

```
# datadir目録を添加
dataDir=/opt/bigdata/servers/zookeeper-3.4.14/data

# logdir目録を添加
dataLogDir=/opt/bigdata/servers/zookeeper-3.4.14/data/logs

# クラスタ配置
# server.サーバID=サーバIP：サーバの通信ポイント：サーバの投票ポイント
server.1=centos1:2888:3888
server.2=centos2:2888:3888
server.3=centos3:2888:3888
server.4=centos4:2888:3888

# 注解を解く
clientPort=2181

# ZK自動的なゴミを片付け機能が提供してくれ，その変数は片付けの頻度、単位が時です。
autopurge.purgeInterval=1
```

![image-20231009205752568](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231009205752568.png)

```
# 他のサーバに分配
scp -r zookeeper-3.4.14/ centos2:$PWD
scp -r zookeeper-3.4.14/ centos3:$PWD
scp -r zookeeper-3.4.14/ centos3:$PWD
```

![image-20231009213957481](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231009213957481.png)

　　各サーバで先作成したdata/目録にmyid名称のファイルを作成し、Zookeeperの節点番号を書き込んでいる。

```
# centos1サーバ
echo 1 >/opt/bigdata/servers/zookeeper-3.4.14/data/myid

# centos2サーバ
echo 2 >/opt/bigdata/servers/zookeeper-3.4.14/data/myid

# centos3サーバ
echo 3 >/opt/bigdata/servers/zookeeper-3.4.14/data/myid

# centos4サーバ
echo 4 >/opt/bigdata/servers/zookeeper-3.4.14/data/myid
```

**Zookeeper起動**

　　毎サーバで以下の命令を実行する。

```
# 起動命令
/opt/bigdata/servers/zookeeper-3.4.14/bin/zkServer.sh start

# 状態検査
/opt/bigdata/servers/zookeeper-3.4.14/bin/zkServer.sh status
```

![image-20231010143009471](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231010143009471.png)

　　全ての節点が起動した後で、状態検査が正しい結果が表れられる。図から見えて、毎サーバには何の役が分配された。

![image-20231010143247938](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231010143247938.png)

![image-20231010143423262](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231010143423262.png)

![image-20231010143502254](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231010143502254.png)

![image-20231010143527514](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231010143527514.png)
