# データ採集工具 -- Flume-3

## 第３節　Flumeの応用

　　Flumeは使える場合が中々多い、その性質に代わって相関の配置がちょっと複雑です。先ず基本的知識を紹介し、後は案例に基づいて深く了解していく。

#### **Flume Source**

**Avro Source**：Avroポートで監聴し、外部のAvroクライアントから序列化の事件を受信する。別のFlumeエージェントの内蔵Avro Sinkと一対にすると階層型の集合を作成できる。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 4141
```

**Thrift Source**：Thriftポートで監聴し、外部のThriftクライアントから事件を受信する。Thrift Sourceは安全模式にしてKerberosと言う安全認証システムを使用できる。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = thrift
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 4141
```

**Exec Source**：Linux命令を使っての出力の情報を受信できる。でも、一旦命令のプロセスが何でも原因に退出すると、対応のSourceも退出してある。例えば、catとtail命令が希望の結果を取られる。dateにはexec sourceが適してない。前者は流れ（stream）形式でSourceに送信して、後者は一つの事件だけ、情報を出力する時プロセスも終わった。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = exec
a1.sources.r1.command = tail -F /var/log/secure
a1.sources.r1.channels = c1
```

**JMS Source**：JMS（Java Message Service）を情報源として受信する。でも、今はJavaのActiveMQを支持できるだけ。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = jms
a1.sources.r1.channels = c1
a1.sources.r1.initialContextFactory =　org.apache.activemq.jndi.ActiveMQInitialContextFactory
a1.sources.r1.connectionFactory = GenericConnectionFactory
a1.sources.r1.providerURL = tcp://mqserver:61616
a1.sources.r1.destinationName = BUSINESS_DATA
a1.sources.r1.destinationType = QUEUE
```

**Spooling Directory Source**：指定されたファイルを目録に追加すると、Flumeはその目録を持続に監聴し、ファイルをSourceとして処理する。でも、一度ファイル目録に追加されると、そのファイルを変更するなどことはできず、変更を試みるとFlumeはエラーを報告する。

```
a1.channels = ch-1
a1.sources = src-1

a1.sources.src-1.type = spooldir
a1.sources.src-1.channels = ch-1
a1.sources.src-1.spoolDir = /var/log/apache/flumeSpool
a1.sources.src-1.fileHeader = true
```

**Taildir Source**：指定された複数のファイルを監聴し、これらのファイルにデータが書き込まれるとSourceがその新しいデータを読み込んで。Taildir Sourceには、データの信頼性が高い、毎回変更する内容がSinkに転送される。Flumeが故障になるといえども、Sourceが最後の読み込んだ位置から再読み込まれる。データの損失ことが発生してない。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = TAILDIR
a1.sources.r1.channels = c1
a1.sources.r1.positionFile = /var/log/flume/taildir_position.json
a1.sources.r1.filegroups = f1 f2
a1.sources.r1.filegroups.f1 = /var/log/test1/example.log
a1.sources.r1.headers.f1.headerKey1 = value1
a1.sources.r1.filegroups.f2 = /var/log/test2/.*log.*
a1.sources.r1.headers.f2.headerKey1 = value2
a1.sources.r1.headers.f2.headerKey2 = value2-2
a1.sources.r1.fileHeader = true
a1.sources.ri.maxBatchCount = 1000
```

**NetCat TCP/UDP Source**：特定のポートで監聴し、テキストの各行を受信する。nc -k -l [host] [port] のように動作します。指定されたデータは改行で区切られた、Flume事件と接続されたチャネル経由で送信される。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = netcat/netcatudp
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 6666
a1.sources.r1.channels = c1
```

#### Flume Channel

**Memory Channel**：事件は配置される可能な最大サイズのメモリに格納される。高いスループット（throughput）に対して非常に適する。でも、障害が発生した場合は格納された一部分のデータを失う可能性があり、ここで注意してください。

```
a1.channels = c1
a1.channels.c1.type = memory
a1.channels.c1.capacity = 10000
a1.channels.c1.transactionCapacity = 10000
a1.channels.c1.byteCapacityBufferPercentage = 20
a1.channels.c1.byteCapacity = 800000
```

**File Channel**：Memory Channelに対応してFile Channelはディスクに書き込んで、つまり、障害が発生するになってもデータを失わない。代わりに書き込みの速度がより遅い。

```
a1.channels = c1
a1.channels.c1.type = fi　le
a1.channels.c1.checkpointDir = /mnt/flume/checkpoint
a1.channels.c1.dataDirs = /mnt/flume/data
```

**JDBC channel**：関係データベースをchannelとしてSourceからデータを受信して、JDBC channelが持続的に格納でき、データを失うことがない。唯一支持できるデータベースDerbyだけ。

```
a1.channels = c1
a1.channels.c1.type = jdbc
```

**Kafka Channel**：JDBC channelに似て、channel中のデータがKafkaに格納される。

```
a1.channels.channel1.type = org.apache.flume.channel.kafka.KafkaChannel
a1.channels.channel1.kafka.bootstrap.servers = kafka-1:9092,kafka-2:9092,kafka-3:9092
a1.channels.channel1.kafka.topic = channel1
a1.channels.channel1.kafka.consumer.group.id = flume-consumer
```

#### Flume Sink

**HDFS sink**：このSinkは、事件をHDFSに書き込みます。現在、テキストファイル（text file）とシーケンスファイル（sequence file）の作成を支持している。また、2つの圧縮形式も支持している。ファイルを定期的に更新（roll）してあり、現在のファイルを閉じて新しいファイルを作成するような動作です。その時間、データ大小、又は事件の数に基づいて設定できる。また、タイムスタンプ（timestamp）などの属性に基づいてデータを仕切って保存することもできる。

```
a1.channels = c1
a1.sinks = k1
a1.sinks.k1.type = hdfs
a1.sinks.k1.channel = c1
a1.sinks.k1.hdfs.path = /flume/events/%Y-%m-%d/%H%M/%S
a1.sinks.k1.hdfs.filePrefix = events-
a1.sinks.k1.hdfs.round = true
a1.sinks.k1.hdfs.roundValue = 10
a1.sinks.k1.hdfs.roundUnit = minute
```

**Hive Sink**：このSinkは、テキスト又はJSONデータを含む事件を受け取り、それらを直接Hiveテーブル又は区分（partition）に転送できる。事件はHiveトランザクションを使用して書き込まれる。一旦事件をHiveに提出すると、直ぐにHiveに検索してきてある。Flumeは区分を事前に作成することも、区分存在しない場合に作成することもできる。入力の事件データからのフィールドは、Hiveテーブルの対応する列に分配される。

```
create table weblogs ( id int , msg string )
    partitioned by (continent string, country string, time string)
    clustered by (id) into 5 buckets
    stored as orc;

a1.channels = c1
a1.channels.c1.type = memory
a1.sinks = k1
a1.sinks.k1.type = hive
a1.sinks.k1.channel = c1
a1.sinks.k1.hive.metastore = thrift://127.0.0.1:9083
a1.sinks.k1.hive.database = logsdb
a1.sinks.k1.hive.table = weblogs
a1.sinks.k1.hive.partition = asia,%{country},%Y-%m-%d-%H-%M
a1.sinks.k1.useLocalTimeStamp = false
a1.sinks.k1.round = true
a1.sinks.k1.roundValue = 10
a1.sinks.k1.roundUnit = minute
a1.sinks.k1.serializer = DELIMITED
a1.sinks.k1.serializer.delimiter = "\t"
a1.sinks.k1.serializer.serdeSeparator = '\t'
a1.sinks.k1.serializer.fieldnames =id,,msg
```

**logger sink**：直接に事件情報を画面に出力させる。通常、これはテストやデバッグに使用される。唯一の例外で原稿データの部分で説明されている追加の設定を必要としない。
