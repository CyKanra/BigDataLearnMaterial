# データ採集工具 -- Flume-4

## 第４節　Flumeの案例

### 第１項　入門案例

　　実現目標：本サーバを監聴し、取り込みの情報が操作卓で表す。

　　内容紹介：telent工具を使って8888ポイントにデータを転送する。sourceがnetcat source調べて、channelがmemory channelを調べて、sinkが logger sink調べて操作卓に顕示してある。

　　所要の工具を準備しておく。

```
yum install telnet
```

![image-20230929152705148](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230929152705148.png)

　　8888ポイントが占めてあるかどうか確認する。

```
lsof -i:8888
```

　　Flume Agentを作成し、以下の内容をflume-netcat-logger.confに添加する。

　　先ずは任務目標によってどの類型のFlume Agent（source、channel、sink）を調べることを決まっている。次に、公式文献を参考して相関の配置を進行し、直接に公式の配置内容をコピーしてきて、その上に改修を行うのは便利です。太字の部分が必須の配置、他の変数を構わなくて大丈夫です。

![image-20230929162513057](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230929162513057.png)

　　以下の内容を任意の目録下に作成するflume-netcat-logger.confファイルに書き込む。

```
# a1はAgentの名称。source、channel、sink各名称はr1　c1　k1
a1.sources = r1
a1.channels = c1
a1.sinks = k1

# source
a1.sources.r1.type = netcat
a1.sources.r1.bind = centos4
a1.sources.r1.port = 8888

# channel
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000
a1.channels.c1.transactionCapacity = 100

# sink
a1.sinks.k1.type = logger

# source、channel、sinkの関係を定義
a1.sources.r1.channels = c1
a1.sinks.k1.channel = c1
```

![image-20230929194556793](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230929194556793.png)

　　全ての流れはこの配置ファイルを巡って展開することです。

 　　Agentを起動してる。若し命令が長すぎ、「\」を入力して改行になれる。

```
$FLUME_HOME/bin/flume-ng agent --name a1 --conf-file /root/data/flume-netcat-logger.conf -Dflume.root.logger=INFO,console
```

![image-20230929200419059](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230929200419059.png)

　　このようなログが表せるなら、Flume起動が成功になるんです。

　　新しく窓口を開けて、telnetを使用して本サーバの8888ポイントに情報を書き込む。

```
telnet centos4 8888
```

![image-20230929200816819](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230929200816819.png)

![image-20230929200836834](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230929200836834.png)

　　Flumeの監聴画面に受信したデータを順調に表す。Flumeのプロセスを終わると、telnetプロセスも終わってなる。