# データ採集工具 -- Flume

## 第１節　Flume概説

　　Hadoopはファイル管理システムとして自身がデータを作り出せない、Hiveにはデータ倉庫工具としてもこの状況です。Flumeは大量データを転送することが実現するため設計された技術です。

### 第１項　Flumeの定義

　　Flumeは、Cloudera社によって開発された、分散型、高信頼性、高可用性、大規模なログデータの収集、集約の転送システムです。 Flumeにはデータの収集を行うため、様々な種類のデータ送信者が対応できる機能を提供されてある。 而も、データを簡単に処理でき、様々なデータ受信者に書き込む能力を提供する。 簡単に言えば、Flumeは実時間でログデータを収集するのデータ採集エンジンです。他の種類データ採集工具がdataX、kettle、Logstash、Scribe、sqoop。

**Flume特性**：

- 分散式：Flumeは分散クラスターとして展開でき、拡張性が高いです。
- 高信頼性：節点の障害が発生した場合でも、ログデータは他のノードに転送され、データの損失がない。
- 利便性: Flumeの取り付けが易いですが、使用する時の配置はちょっと複雑で、一定な専門知識が必要です。
- 実時間収集: Flumeはデータの実時間収集を支持している。

### 第２項　Flumeの結構

![image-20230919160148593](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230919160148593.png)

**Flume結構部品**：

- Agent（エージェント／代理）: Flume代理は、データの収集、転送、及び処理を行うプロセスです。AgentはJVMプロセスであり、外部のログ生成者から事件データを収集し、目的地又は別のAgentに転送する。Agentには通常、Source、Channel、及びSinkの3つの主要なコンポーネントが含まれている。

- Source（ソース）: Sourceはデータの入力部品を表す。さまざまな種類のログデータ（例: avro、exec、spooldir、netcatなど）を受け取る。

- Channel（チャネル）: ChannelはSourceとSinkの間にあるバッファリング（buffering）区です。Channelは同時に複数のプロセスを処理でき、各プロセスの間に独立、安全です。Channelは異なる速度でSourceとSinkが運行する時、データの一時的な保存と転送を管理する。通常のChannelが二種ある：
  - Memory Channel（メモリーチャネル）：メモリ内にデータを一時的に保存するChannelです。Memory Channelはデータの一時的な損失を許容できる場合に適している。Flumeなど異常終了したら、再起動した場合でもメモリ内のデータが失われることがある。
  - File Channel（ファイルチャネル）：全てのデータをディスク（disk）に書き込むChannelです。そのため、全体サーバを閉じるにしても、再起動してデータが失われる ことはない。

- Sink（シンク／水槽）: Sinkはデータの出力先を表す。SinkはChannelからデータを取得し、バッチ処理して指定された目的地、又は別のFlumeに送信する。Sinkは完全にトランザクショナルであり、データが正常に書き込まれたことを確認した後、Channelからデータを削除し、その流れを絶えず繰り返す。

- Event（イベント／事件）: EventはFlumeがデータストリームを扱う最小単位です。Eventにはデータ本体とそのメタデータが含まれており、これらがSourceからSinkまで転送される。


### 第３項　Flumeのトポロジー結構

**直列接続**

　　複数のFlumeを順次接続し、データを初めのSourceから最終的なSinkまで連続して転送する方法です。接続の所が必ずAvro方式を使用して転送する。

![image-20230920151503666](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230920151503666.png)

**コピー模式**

　　Eventデータを複数の目的地に向けて流させる。Sourceからのデータが複数のChannelにコピーされて同じデータが転送する。Sinkが不同な目的地にデータを送信することができる。

![image-20230920153111863](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230920153111863.png)

**重合模式**

　　この模式は実用性高い、通常な場合に数百台のサーバーが分散配置されている。各サーバーには生成されるログを収集して中央Agentに集めさせる、大規模な場合にはこの組み合わせ方式が非常に役立つ。

![image-20230920165416309](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230920165416309.png)

**負荷分散**

　　一つEventデータを複数のSinkに分けて不同なサーバのAgentに送信する。最後に各Agentのデータを一つの対象に重合される。

![image-20230920165040160](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230920165040160.png)

### 第４項　Flume内部原理

![image-20230921153806278](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230921153806278.png)

**流れ説明**：

**Source（ソース）**：Sourceは事件を受信し、それらをChannel に渡す役割を果たす。

**Interceptor（インターセプター）**：InterceptorはSourceから受け取った事件を濾過するものです。

**Channel Selector（チャネルセレクター／選別器）**：Channel Selectorは、Interceptorで処理された事件をどのチャネルに書き込むかを決める。2つの一般的な類型の選別機がある。

- **Replicating Channel Selector（複製型選別器）**：この選別器は、Sourceからの事件をすべてのチャネルに複製する。これは、データの冗長性と可用性を確保するために複数のチャネルを使用する場合に役立つ。
- **Multiplexing Channel Selector（多重化型選別器）**：この選別器は、事件の特定のheader値に基づいて、不同なチャネルに事件を分散させる。headerによってチャネルへの振り分けが行われる。

**Sink（シンク）**：Sinkはチャネルから事件を受け取り、それらを最終的な目的地に書き込む役割を果たす。