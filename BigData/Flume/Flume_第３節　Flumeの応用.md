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
