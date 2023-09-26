# データ採集工具 -- Flume-3

## 第３節　Flumeの応用

　　Flumeは使える場合が中々多い、その性質に代わって相関の配置がちょっと複雑です。先ず基本的知識を紹介し、後は案例に基づいて深く了解していく。



Avroポートでリッスンし、外部のAvroクライアントストリームからイベントを受信します。別の（前のホップ）Flumeエージェントの内蔵Avro Sinkとペアにすると、階層型のコレクショントポロジを作成できます。

**Flume Source**

Avro Source：Avroポートで監聴し、外部のAvroクライアントから序列化の事件を受信する。別のFlumeエージェントの内蔵Avro Sinkと一対にすると階層型の集合を作成できる。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = avro
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 4141
```

Thrift Source：Thriftポートで監聴し、外部のThriftクライアントから事件を受信する。Thrift Sourceは安全模式にしてKerberosと言う安全認証システムを使用できる。

```
a1.sources = r1
a1.channels = c1
a1.sources.r1.type = thrift
a1.sources.r1.channels = c1
a1.sources.r1.bind = 0.0.0.0
a1.sources.r1.port = 4141
```

