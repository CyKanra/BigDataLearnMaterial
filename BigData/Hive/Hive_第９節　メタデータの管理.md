# Hiveデータ倉庫工具ー9

## 第９節　メタデータの管理

　　Hiveを具体的に使用する際、最初に直面する問題はテーブルの構造情報をどのように定義し、データファイルと構造化されたデータを対応できる。ここで言う「対応」とは、対応のある関係を指す。Hiveでは、テーブルとファイルの間の対応関係や、列とフィールドの間の関係などを明確に記述する必要がある。これらの対応関係を記述するデータをHiveのメタデータと呼ぶ。このデータは非常に重要であり、SQLの記述と実際の操作の関係がメタデータを通じて確定できる。

　　**Metadata**はメタデータ、Hiveで作成されたデータベース、テーブル、列などの要素を含んでの記述類型のファイル、データ自体ではない。このメタデータは関連型データベースに格納され、内蔵のDerbyやMySQLなどの第三者データベースを使用して保存される。

　　**Metastore**はメタストア、Hiveがメタデータを管理するためのサービスです。メタストアの存在により、上位のサービスは生のデータファイルに訪問する必要がなくなる。代わりに、構造化されたデータベース情報に基づいて、Hive がメタストアを通じて計算してデータを画面に表してくれる。

公式文献：[AdminManual Metastore Administration - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/AdminManual+Metastore+Administration)

### 第１項　メタストアの模式

**組み込み模式（Embedded）**

　　組み込み模式は、組み込みのDerbyデータベースを使用してメタデータを保存し、個別にMetastoreサービスを追加するのは必要ない。つまり、インストールパッケージを解凍するやDerbyデータベースを初期化にして全ての流れが完成だ。でも、1回に1つのクライアント接続しか処理できず、テスト環境に適しているが、本番環境では適してない。

```
#データベース初期化
schematool -dbType derby -initSchema
```

**本地模式（load）**

　　本地模式は、外部データベースを使用してメタデータを保存するの模式です。それも本篇の文章に主に紹介したの模式、幾つか配置情報を添加しておくのは必要です。現在支持されているデータベースはMySQL、Postgres、Oracle、MS SQL Serverです。

　　本地模式では、個別にMetastoreサービスを起動する必要はない。Hiveと同じプロセス内で動作する。つまり、Hiveサービスを起動するとその内部で自動的にMetastoreサービスも起動される。Hiveはhive.metastore.uris変数の値に基づいて判定し。この変数が空であれば、本地模式が採用られる。

**リモート模式（Remote）**

　　リモート模式では、Metastoreサービスを個別に起動する必要があり、各クライアントは設定ファイルでこのMetastoreサービスへの接続情報を設定する。リモートのMetastoreサービスとHiveは不同なプロセスで実行される。そのため、手動で起動する必要がある。本番環境では、Hive Metastoreを設定する際にリモートモードを使用することをお勧めです。

　　この模式下のは、他のHiveに依頼するソフトウェアもMetastoreを通じてHiveに訪問できる。この場合、hive.metastore.uris変数を配置し、MetastoreサービスのIPアドレスとポートを添加しておく必要です。Metastoreサービスは複数の節点に配置でき、利用可能な節点を自動的に選択し、その節点が故障になったらHiveの利用不能を引き起こすことも回避できる。

| 節点    | metastore | client |
| ------- | --------- | ------ |
| centos1 | √         |        |
| centos2 | √         |        |
| centos3 | √         |        |
| centos4 |           | √      |

```
#centos2上のHiveを他のサービスにコーヒーして
cd $HIVE_HOME
cd ../
scp -r hive-2.3.7/ centos1:$PWD
scp -r hive-2.3.7/ centos3:$PWD
scp -r hive-2.3.7/ centos4:$PWD
```

![image-20230901155339474](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230901155339474.png)

```
#環境変数ファイルも他のサービスを分配
cd /etc
scp profile centos1:$PWD
scp profile centos3:$PWD
scp profile centos4:$PWD

#各サービスのファイルを利かす
source /etc/profile

#成功かどうかを検証
cd $HIVE_HOME
```

![image-20230901160751906](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230901160751906.png)

```
#centos1~centos3のmetastoreサービスを起動
nohup hive --service metastore &

#metastoreサービスのポートを検査、ちょっと起動の時間が掛かる
lsof -i:9083

#若しツールなければ、yum取り付け
yum install lsof
```

![image-20230901163352311](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230901163352311.png)

centos4のhive-site.xmlを改修：MySQLに関する配置情報を消除して、metastoreの接続情報を添加する

![image-20230901171458887](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230901171458887.png)

```
#需要の配置情報
<!-- hive metastore サービスアドレス -->
<property>
	<name>hive.metastore.uris</name>
	<value>thrift://centos1:9083,thrift://centos2:9083,thrift://centos3:9083</value>
</property>
```

![image-20230901171742617](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230901171742617.png)

　　下図ようにcentos4サービスにHive命令を使って会話に入ったら、リモート模式の配置が成功です。

![image-20230901172249204](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230901172249204.png)

　　一つのmetastoreサービスを保留して、他のmetastoreサービスが全部閉じる。会話に再入って、そのままで運行状態のmetastoreサービスを別のサービスに変更して、当面Hiveの訪問に影響がない。それはHiveの高可用性（High Availability）です。

### 第２項　HiveServer2

　　HiveServer2を紹介する前にHiveServerは何のことをちょっと説明しておく。

　　HiveServerは、リモートのクライアントがHiveServerを通じてHiveに訪問できる内蔵サービスです。Apache ThriftTM技術の上に構築されているため、時々Thrift Serverとも呼ばれた。新しいサービスであるHiveServer2もThriftの上に構築されているからです。HiveServer2が導入されて以降、HiveServerはHiveServer1とも呼ばれている。今HiveServer1 サービスもう廃棄された。

　　HiveServer2導入は以下の不足点を完備するためHiveServerサービスを書き直す

- 1回1つのクライアントが連接させるのが許可する。
- 身分検証が支持しない。
- 訪問流れの会話管理が支持しない。

**HiveServer2概説**

　　HiveServer2（HS2）は、クライアントがHiveに訪問が実行できるサービスです。HiveServer2はHiveServer1の後継バージョンです。HS2は複数のクライアントの同時実行と認証を支持し、JDBC、ODBCなどの常用APIクライアントに対してより良く支持を提供することを目的としている。

- クライアントにリモートでHiveのMetastoreを訪問するサービスを提供してあげる。
- thrift協議を基づいて、クロスプラットフォームやクロスプログラム言語が実現した。
- 複数の訪問と身分検証が支持できる。

**HiveServer2架構**

　　HiveServer2は、Thrifを基づいて新しいRPCインターフェースを実装し、このインターフェースはクライアントの並行リクエストを処理できる。現在のバージョンでは、Kerberos、LDAP、及び挿入可能の特製（Pluggable Custom）身分認証を支持している。

![image-20230906160402024](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230906160402024.png)

　　上図から見て、HiveServer2はHiveの実行エンジンとして容器（docker）です。各クライアント接続に対して、新しい実行コンテキストが作成され、クライアントからのHive SQLリクエストを処理する。新しいRPCインターフェースは、サーバーの実行コンテキストと外部のリクエストを関連付けてメタデータの訪問が実現してある。

**HiveServer2依存**

- Metastore：埋め込み模式（HS2と同じプロセス内）又はリモート模式（Thriftのサービス）として構成できる。HS2とMetastoreを接続してMetastoreに必要なリクエスト情報を提供してあげる。

- Hadoop：HS2はさまざまな実行エンジン（MapReduce/Tez/Spark）の物理実行計画を準備し、Hadoopに実行任務を引き渡す。

![image-20230906160445316](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230906160445316.png)

**HiveServer2身分検証**

　　HiveServer2は、匿名（検証なし）、SASLあり及びSASLなし、Kerberos（GSSAPI）、LDAPを通じて挿入可能な特製認証（Pluggable Custom Authentication）、及び挿入可能な認証モジュール（Pluggable Authentication Modules）を支持している。

**HiveServer2クライアント**

　　不同なHiveServer2クライアントの方式をちょっと説明する。

- Beeline：Hiveの新しく引用されたのコマンドライン形式でSQLLine CLIを基づいてのJDBCクライアント、SQLLine CLIを基づいて以後Hiveクライアントが完全にBeelineを代わってる。
- JDBC：その名詞は中々了解してあり、HiveServer2 はJDBCドライバあり、MySQLのJDBCみたいHiveServer2サービスを連接していい。
- Python ClientとRuby Client：PythonとRuby言語でHiveServer2サービスの連接方法。

**HiveServer2配置**

| 節点    | HiveServer2 | client |
| ------- | ----------- | ------ |
| centos1 |             |        |
| centos2 | √           |        |
| centos3 |             |        |
| centos4 |             | √      |

　　「$HADOOP_HOME/etc/hadoop/」目録下のcore-site.xmlに以下の内容を追加。この二つの配置の目的はrootユーザーに全ての訪問権限を分配してあげる。配置なければ、外部訪問する時三つ目図のようなエラー情報が返却される。

```
<!-- rootユーザーが全てのユーザーに代わってる -->
<property>
	<name>hadoop.proxyuser.root.hosts</name>
	<value>*</value>
</property>
<property>
	<name>hadoop.proxyuser.root.groups</name>
	<value>*</value>
</property>
```

![image-20230906202514054](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230906202514054.png)

![image-20230908161043485](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230908161043485.png)

　　hdfs-site.xmlに以下の内容を追加（不要可）、せめて私の環境はこの変数が需要じゃない。

```
<!--webhdfsサービスを起動 -->
<property>
	<name>dfs.webhdfs.enabled</name>
	<value>true</value>
</property>
```

![image-20230906203626226](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230906203626226.png)

```
#他のサーバに分配する
scp -r core-site.xml centos2:$PWD
scp -r hdfs-site.xml centos2:$PWD
```

![image-20230906203910560](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230906203910560.png)

```
#再Hadoopサービスを起動、jps命令使って起動が成功かどうかを検査してる
start-dfs.sh
start-yarn.sh
mr-jobhistory-daemon.sh start historyserver

#centos2サーバがhiveserver2を起動
nohup hiveserver2 &

#hiveserver2のポイント号を検査
lsof -i:10000
```

![image-20230907154101159](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230907154101159.png)

　　ブラウザにhiveserver2の存在サーバIPアドレスと10002ポイント号（centos2:10002/）を入力し、順調なら下図が現れる。

![image-20230907154351161](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230907154351161.png)

　　centos4サーバでhiveserver2のBeelineクライアントを起動

```
#$HIVE_HOME/bin目録beeline起動ファイル
./beeline
!connect jdbc:hive2://centos2:10000 scott tiger
show databases;
use mydb;
select * from temp;
!help
!quit
```

![image-20230907161310108](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230907161310108.png)![image-20230907162729004](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230907162729004.png)

![image-20230907162820317](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230907162820317.png)

　　Beelineが成功にhiveserver2を連接した時、Hiveserver2の会話管理Active Sessionsが1のsessionsになる。具体的な実行流れのSQL語句も表せる。

　　上図から見えて、 「select * from mydb.transtable」命令が失敗した。トランザクションのテーブルに対して新たな会話に入る前に幾つか変数が設定する必要です。

![image-20230907163744146](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230907163744146.png)

```
SET hive.support.concurrency = true;
SET hive.enforce.bucketing = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
SET hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager;

#若し「NullPointerException」みたいのエラー情報が表し、以下の変数を入力してみる
SET hiveconf:tez.am.container.reuse.enabled=false;
```

![image-20230908195107170](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230908195107170.png)

### 第３項　HCatalog

　　HCatalogは、Hadoop向けのテーブルとメタデータの格納の管理層（又は抽象層）、さまざまなデータ処理ツール（Pig、MapReduce、Sparkなど）を使用してあるため、ユーザーがHadoopクラスタでのデータファイルを読み書きする際の手続きを簡略化することを目的としている。HCatalogはユーザーに抽象テーブルを提供させて使って、具体的なデータの格納形式と訪問方法が注目する必要ない。

![image-20230911155821362](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230911155821362.png)

　　簡単の説明は、Hiveの命令画面やHivesever2ような方式ではない、Haddoop分散ファイルシステムを訪問して、直接にファイル中のデータが操作を行うことです。

```
#新たな配置が必要ない
cd $HIVE_HOME/hcatalog/bin

#案例
./hcat -e "use mydb; show tables"
./hcat -e "desc mydb.emp"
./hcat -e "create table default.test1(id string, name string, age int)"
```

![image-20230911161638547](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230911161638547.png)

### 第４項　データの格納形式

　　Hive支持できる格納形式：TEXTFILE（黙認形式）、SEQUENCEFILE、RCFILE、ORCFILE、PARQUET。

　　TEXTFILEは黙認形式、テーブルを作成して形式が指定しなったら、TEXTFILEに使って、直接本地サーバからテーブルにテータを導入できる。でも、他の形式が定義されたのテーブルが必ずTEXTFILE形式のテーブルに導入しておき、後でそのテーブルからinsert命令でsequencefile、rcfile、orcfileテーブルに挿入する。

**Row-basedとcolumn-based**

　　行指向型（Row-based）の格納方法は1つ1つの行を一塊のデータとして扱う。データの追加も行単位で実行するし、検索結果も複数の行として返ってきるし、削除や更新の際には特定の行に対して更新や削除と操作が行われる。一般的なデータベースは行指向であり、例えばMySQL、Oracle、DB2、SQL Serverなどです。

　　列指向型（column-based）の格納方法は列方向にデータを纏めて扱う。そのため、特定の列の値を纏めて処理することに非常に長けている。例えば、あるテーブルから商品名と価格だけを抜き出す処理などは効率的です。逆に、全てのフィールドに操作が扱う場合、データを抜き出し、又は更新や削除は苦手です。別の利点は列格納で空文字が空間を占めない、つまり、同じ大小データが列指向で格納してより小さいメモリーが需要です。

![image-20230912162809196](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230912162809196.png)

　　TEXTFILE、SEQUENCEFILEは行指向型の格納形式、ORCやPARQUETは列指向型に基づいて格納する。

**TEXTFILE**

　　これがデフォルトの保存形式です。テーブルを作成する際にファイルの形式が指定されていない場合、HiveはTEXTFILE形式を使用します。データを導入する際、データファイルはHDFSに直接コピーされ、余りの処理は行わない。

```
Create table textfile_table(
column_specs)
stored as textfile;
```

**SEQUENCEFILE**

　　SequenceFileは、Hadoop APIが提供する二進数ファイル形式で、使用が簡単で分割、圧縮も可能な特性を持っている。SequenceFileは3つの圧縮方式を支持しており、それは「none」、 「record」、 「block」です。「record」の圧縮率が低いため、通常は「block」圧縮をお勧め。

**RCFILE**

　　RCFile（Record Columnar File）は、SequenceFileに類似したキーと値のデータファイルです。先ず、データを行単位で水平に分割し、これにより1行のデータは同じクラスタ節点（node）に保存されることが保証できる。次に、行を垂直に分割して列指向型形式で保存される。行指向型と列指向型の利点と欠点を組み合わせて混合格納形式です。

　　Hiveには索引功能にしてのコストが高い、列指向型の格納形式を引用したら、ある列を読み取りに対して索引検索に等しい効果です。同時に行指向型の幾つか特性が保留されてあり、全フィールドの読み取り、全表を検索することが避けられる。

![image-20230913153222111](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230913153222111.png)

**ORCFILE**

　　ORC File（Optimized Row Columnar File）は、RCFileを最適化したもので、Hive 0.11で引用されたファイル形式です。このファイル形式は、Hiveデータを効率的に格納する方法を提供し、他の形式の不足を克服することを目指して設計されている。ORC Fileを使用すると、Hiveのデータの読み取り、書き込み、および圧縮の性能を向上させることができる。

ORCファイルの構造は、次の3つの主要な部分から成り立っている：

1. ストライプ（Stripe）：実際のデータを格納する場所で、通常は1つのストライプのサイズは250MBです。ストライプ内のデータは列指向型の格納、複数のストリームに分割されて格納される。
   - 索引データ（Index Data）：ORCファイルには、データの高速の訪問を支持するための軽量な索引が含まれている。通常、1万行毎に索引が作成される。これには、ストライプ内のデータの統計情報や、ストライプ内で位置情報が含まれる。
   - 行データ（Rows Data）：実際のデータが格納される場所です。行データを取得して列に書き込む。各列は符号化にして、複数のストリーム形式に分割して格納される。これにより、データの圧縮と効率的な格納が実現される。
   - ストライプフッター（Stripe Footer）：各ストライプのメタデータ情報が含まれる。
2. ファイルフッター（File Footer）：ファイル内のストライプのリスト、各ストライプの行数、及び各列のデータ型などが含まれる。また、各列の最大値、最小値、行数、合計などの情報も含まれている。
3. Postscript：圧縮引数と圧縮サイズに関する情報が含まれる。

![image-20230913163204970](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230913163204970.png)

　　ORCFileは、3つの異なる級別の索引情報を提供している：ファイル級別、ストライプ級別、行グループ級別。これらの索引情報を利用することで、データの検索と読み取りの効率を向上させ、条件を満たさないクエリ条件のデータ塊がほとんど回避できる。ORCを使用することで、整体の運行負荷が効率的になれるんです。

**PARQUET**

　　Apache Parquetは、Hadoop生態内で使用される新しい列指向のデータ格納形式であり、Hadoop生態内の殆どの計算フレームワーク（MapReduce、Sparkなど）と互換性があり、様々な検索エンジン（Hive、Impala、Drillなど）に対応している。また、言語やプラットフォームに依存しない。汎用性はORCFileに比べて一つの優勢です。

　　Parquetファイルは二進数形式で保存され、直接読み取ることはできない。ファイルには実際のデータとメタデータが含まれており、Parquet形式のファイルは自己解釈可能です。

![image-20230915193946712](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230915193946712.png)

Row group（行組）:

- データを書き込む時の最大格納単位。
- MapReduce任務の最小並列単位。
- 通常、50MBから1GBのサイズです。

Column chunk（列塊）:

- 現在のRow group内の特定の列データを格納する。
- 最小のIO並列処理単位です。

Page（ページ）:

- 圧縮及びデータの読み取りの最小単位。
- 単一のデータを取得する際の最小読み取り単位。
- 通常、8KBから1MBのサイズで、サイズが大きいほど圧縮効率が高くなる。

Footer（フッター）:

- データに相関情報。
- 各Row groupの元情報：オフセット（offset）、サイズ。
- 各Column chunkの元情報：列毎のエンコーディング（encoding）形式、最初のページのオフセット、最初の索引ページのオフセット、個数、サイズなどの情報を含る。
