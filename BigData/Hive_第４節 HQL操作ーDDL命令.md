# Hiveデータ倉庫工具ー4

## 第４節　HQL操作ーDDL命令

DDL（data definition language）：データ定義言語の略であり、主な命令はCREATE、ALTER、DROP等。データベースの結構と類型を定義し、変更する為に使用される。

参考：[LanguageManual DDL - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DDL)

### 第１項　データベースの操作

- Hiveには、黙認データベースという（default）としてあり、若し特に指定してなければ、HQL操作する場合で「default」が使用される。つまり、一般的にデータベースを明示的に指定するはずです。
- Hiveのデータベースとテーブル名称が大文字と小文字の区別がないという特徴がある。
- 名称の先頭に数字を使用。
- データベース名やテーブル名の定義にキーワードが使用することはできません。

**データベース作成の語法**

```
CREATE (DATABASE|SCHEMA) [IF NOT EXISTS] database_name
[COMMENT database_comment]
[LOCATION hdfs_path]
[MANAGEDLOCATION hdfs_path]
[WITH DBPROPERTIES (property_name=property_value, ...)];
```

```
#データベース作成，HDFS上にアドレス： /user/hive/warehouse/*.db
hive (default)> create database mydb;
hive (default)> dfs -ls /user/hive/warehouse;

#「if not exists」使って、存在したデータベースを作成するエラーが避ける
hive (default)> create database if not exists mydb;

#備考が添加し，データ保存する位置が指定する
hive (default)> create database if not exists mydb2
comment 'this is mydb2'
location '/user/hive/mydb2.db';
```

![image-20230405161539482](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230405161539482.png)

**データベースの顕示**

```
#全てのデータベース顕示
show database;

#データベースの情報が顕示する
desc database mydb2;
desc database extended mydb2;
describe database extended mydb2;
```

![image-20230405162848165](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230405162848165.png)

**データベース使う**

```
use mydb;
```

**データベース削除**

```
#空データベース削除
drop database databasename;

#空データベースじゃなければ、cascade使って強制的に削除する
drop database databasename cascade;
```

![image-20230405164706783](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230405164706783.png)

### 第２項　テーブル作成語法

```
create [EXTERNAL] table [IF NOT EXISTS] table_name
    [(colName colType [comment 'comment'], ...)]
    [comment table_comment]
    [partition by (colName colType [comment col_comment], ...)]
    [clustered by (colName, colName, ...)
    [sorted by (col_name [ASC|DESC], ...)] into num_bucketsbuckets]
    [
        [row format row_format]
        [stored as file_format]
    ]
    [LOCATION hdfs_path]
    [TBLPROPERTIES (property_name=property_value, ...)]
    [AS select_statement];

CREATE [TEMPORARY] [EXTERNAL] TABLE [IF NOT EXISTS]
    [db_name.]table_name
    LIKE existing_table_or_view_name
    [LOCATION hdfs_path];
```

1. create table：定義された名称によってテーブル作成する。若し同じ名称のテーブルが存在すれば、エラー異常が触発する。[IF NOT EXISTS]使ってエラーが回避できる。

   ```
   CREATE TABLE IF NOT EXISTS my_table (
     column1 INT,
     column2 STRING
   );
   ```

2. EXTERNAL：キーワードが添加して当のテーブルは外部テーブル（External Tables）限り、他のは管理テーブル（内部テーブル、Managed /Internal）です。

3. comment：テーブルの備考。

4. partition by：データを仕切り（partition）に分割して仕切りのフィールドを指定してください。一般的に日付や地域別に分割することがよくある。

   ```
   CREATE TABLE my_table (
     column1 INT,
     column2 STRING
   )
   PARTITIONED BY (date STRING);
   ```

5. clustered by：パーティションされるデータを桶（bucket）に分割して、桶のフィールドを指定してください。

6. **sorted by**：桶内のデータを並び替える

7. **ROW FORMAT**：Hiveの第３項に書いて表され、データが一定的コーディングによって保存されて、データの毎部分の中間に特別な符号で区切ります。

   ```
   ROW FORMAT DELIMITED
   　[FIELDS TERMINATED BY char]
   　[COLLECTION ITEMS TERMINATED BY char]
   　[MAP KEYS TERMINATED BY char]
   　[LINES TERMINATED BY char] | SERDE serde_name
   　[WITH SERDEPROPERTIES　(property_name=property_value,property_name=property_value, ...)]
   ```

   ```
   CREATE TABLE page_view(viewTime INT, userid BIGINT,
        page_url STRING, referrer_url STRING,
        ip STRING COMMENT 'IP Address of the User')
    COMMENT 'This is the page view table'
    PARTITIONED BY(dt STRING, country STRING)
    CLUSTERED BY(userid) SORTED BY(viewTime) INTO 32 BUCKETS
    ROW FORMAT DELIMITED
      FIELDS TERMINATED BY '\001'
      COLLECTION ITEMS TERMINATED BY '\002'
      MAP KEYS TERMINATED BY '\003'
      LINES TERMINATED BY '\r\n'
    STORED AS SEQUENCEFILE;
   ```

   

8. stored as SEQUENCEFILE|TEXTFILE|RCFILE：メモリー格式が指定する。普通のテキスト格式でメモリされてTEXTFILEを使える。若しデータを圧縮にするつもり、SEQUENCEFILE（二進法）使える。

9. LOCATION：HDFSの保存位置を指定する。

10. TBLPROPERTIES：テーブル属性の定義。

11. AS：後ろに詮索語句を付いて、詮索結果よってテーブルを作成する。

12. LIKE：like テーブル名、既にあるテーブルをコーピして、でもテーブルのデータがコーピしてない。

### 第３項　内部テーブルと外部テーブル

　　テーブルを作成する際には、テーブル類型が指定されることができる。その類型は内部テーブル（Managed /Internal Tables）と外部テーブル（External Tables）の2つがあります。

- 特に指定がない場合は内部テーブルが作成される。一方、キーワードexternal使って外部テーブルを作成できる。
- 内部テーブルを削除すると、関連するメタデータとテーブルのデータ同時に削除される。
- 外部テーブルを削除して関連なメタデータ削除されるだけ、データ自体が保留される。
- 現行環境では外部テーブルがよく使われる。

**内部テーブル**

**Data**

```
#data1.datデータ
1;suzuki;book,TV;tokyo:sinbuyakun
2;yamada;Music,game,novel;kanagawaken:kawasaki,tokyo:itabasikun
3;kimura;skiing,animation;chibaken:sakurasi
```

**SQL**

```
# 内部テーブル作成
create table data1(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>
)
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";
 
# 情報の顕示
desc data1;

# より詳しい情報を顕示し、格式が規範になる
desc formatted data1;
```

![image-20230413193032161](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230413193032161.png)

```
-- データをアップロード
load data local inpath '/usr/hadoop/data/data1.dat' into table data1;

-- データを詮索
select * from data1;

-- データファイルを顕示
dfs -ls /user/hive/warehouse/mydb.db/data1;
```

![image-20230413195612202](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230413195612202.png)

![image-20230413195912465](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230413195912465.png)

```
-- テーブルを削除し、データも削除される。
drop table data1;
```

![image-20230413200056362](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230413200056362.png)

**外部テーブル**

SQL

```
# 外部テーブル作成
create external table data1(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>
)
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";
 
# 情報の顕示
desc data1;

-- データをアップロード
load data local inpath '/usr/hadoop/data/data1.dat' into table data1;

-- データを詮索
select * from data1;
```

![image-20230415163156332](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230415163156332.png)

```
-- テーブルを削除し、データも削除される。
drop table data1;

-- データファイルを顕示
dfs -ls /user/hive/warehouse/mydb.db/data1;
```

![image-20230415163252735](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230415163252735.png)

保留されたデータを再利用する場合は、同じ名称テーブルを作成してデータを再度利用することができます。

![image-20230415171912262](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230415171912262.png)

**内部テーブルと外部テーブル変換**

```
create table data1(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>
)
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";

load data local inpath '/usr/hadoop/data/data1.dat' into table data1;

# 外部テーブルに変換する
alter table data1 set tblproperties('EXTERNAL'='TRUE');

# 内部テーブルに変換する
alter table data1 set tblproperties('EXTERNAL'='TRUE');

desc formatted data1;
```

![image-20230415185250107](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230415185250107.png)

![image-20230415185443811](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230415185443811.png)

### 第４項　パーティションテーブル

　　Hiveは詮索語句を実行する際に、全てのデータが走査するため、データ量が大きい場合は時間がかかり、効率も低くなる。

　　ある時特定の条件に一致する部分のデータを走査するだけで、仕切り観念を引っ込み、不同的な目録に相同的な条件のデータを保存させて、毎目録が一つの仕切りに対応する。仕切りの内部に詮索するのは、全てのデータの走査が必要ない、処理の効率が高まる。

　　通常の情況に時間、アドレスを仕切りとして使用する。

```
create external table data2(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>
)
partitioned by (dt string)
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";
 
# データ添加
load data local inpath '/usr/hadoop/data/data1.dat' into table data2 partition(dt="2023-04-17");

load data local inpath '/usr/hadoop/data/data1.dat' into table data2 partition(dt="2023-04-18");
```

![image-20230417205518706](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230417205518706.png)

　　仕切りのフィールドがテーブル自体のフィールドじゃないが、列にとして処理できる。

![image-20230417211209822](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230417211209822.png)

　　若しテーブル自体のフィールドを仕切りのフィールドとして指定すると、エラーを発生することがある。

![image-20230417214501596](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230417214501596.png)

　　先述の内容は静態仕切りであり、仕切りフィールドの値が既に決まった。若し一定数の仕切りを設定することが必要なら、例えば市名など、データを仕切りに一つ一つで分割して作業量が大きくなる。開発者にとって動態的処理する方法があることを望む、ここで動態仕切りを使用して問題が解決できる。

　　先ず変数を改修して、動態仕切りの機能が効く。

```
# 仕切り模式，黙認：strict（厳格状態）
set hive.exec.dynamic.partition.mode=nonstrict

# 動態仕切りを起動、黙認：true
set hive.exec.dynamic.partition=true

# 最大仕切り数、黙認：1000
set hive.exec.max.dynamic.partitions=1000
```

![image-20230510164831225](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230510164831225.png)

　　事前に仕切りフィールドの値が決まってないために、元のデータファイル中に仕切りフィールドがありません、本地サービスから直接にデータを仕切りに導入するのができない、エラーが現れる。

![image-20230510184053969](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230510184053969.png)

```
# 新しいテーブルを作成
create external table data2Partition1(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>
)
partitioned by (dt string)
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";

# data2から詮索してdata2Partition1に挿入
insert into table data2Partition1 partition (dt) select * from data2;

# 元のデータを書き換えるように導入
insert overwrite table data2Partition1 partition (dt) select * from data2;

# 仕切りフィールドを順序で最後に置く
insert overwrite table data2Partition1 partition (dt) select id, name, hobby, addr, dt from data2;
```

![image-20230510191524043](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230510191524043.png)

![image-20230510192158123](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230510192158123.png)

　　最後に指定されたフィールドによって動態的に不同的な仕切りに分配する。

　　ここで幾つか注意点が説明する。

- 仕切りフィールドを順序で最後に置くはずです。同じフィールド名によって自動的にデータを挿入できない。

```
insert overwrite table data2Partition1 partition (dt) select id, name, hobby, addr, dt from data2;
```

![image-20230511164446398](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230511164446398.png)

- 複数の仕切りフィールドが許す。

```
# insert overwrite table data2Partition1 partition (year, month) select id, name, hobby, addr, year, month from data2;
```

- 上記の過程から明らかなように、どんな仕切りにデータを挿入しても、データが重複排除の処理などがありません、最後に表れる結果が完全に挿入されたデータに決められる、つまり、挿入の前にデータの正当性を特に確認することが必要です。

**仕切り検査**

```
show partitions data2;
```

![image-20230417215112458](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230417215112458.png)

![image-20230417215125655](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230417215125655.png)

　　HDFSにデータの仕切りは複数のファイルに分割されて保存し、仕切りに重複のデータが存在することは許し、実は、各仕切りの中間に関係ないということです。仕切りファイルが新しい仕切り作成される際にも作成されてる。

**仕切りを添加とデータを追加**

```
# 一つの仕切り添加
alter table data2 add partition(dt='2023-04-19');

# 複数の仕切り添加
alter table data2 add partition(dt='2023-04-20') partition(dt='2023-04-21');
```

![image-20230419151255863](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230419151255863.png)

```
# データを追加しておく
dfs -cp /user/hive/warehouse/mydb.db/data2/dt=2023-04-17　/user/hive/warehouse/mydb.db/data2/dt=2023-04-22

# 後で新しい仕切りを作成する
alter table data2 add partition(dt='2023-04-22');
```

![image-20230419170348753](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230419170348753.png)

**？？-----------仕切りのHDFSアドレスを改修**

```
alter table data2 partition(dt='2023-04-21') set location '/user/hive/warehouse/data2/dt=2023-04-23';
```

![image-20230424153513600](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230424153513600.png)

　　正直言って、どんな状況でHDFSアドレスを改修することが分かんない。

![image-20230419221352147](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230419221352147.png)

![image-20230419222720401](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230419222720401.png)

　　若し改修の目標仕切り（dt=2023-04-23）存在しません、又は目標仕切りのデータがありません、改修されたデータが失う。そのため、改修前に目標仕切りと具体的な情況を確認することが必要です。

**ーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーーー**

**仕切りの削除**

　　仕切りの削除とテーブルの削除は、全てデータ自体に対して削除するという操作、ただ範囲が違う。また、外部テーブルが使用したら、削除する際にデータファイルが保留される。

```
alter table data2 drop partition(dt='2023-04-21'), partition(dt='2020-04-22');
```

![image-20230424155426438](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230424155426438.png)

![image-20230424155441268](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230424155441268.png)

### 第６項　バケツテーブル

　　データが単一の仕切りに保存するが、データ量が大きすぎ、仕切りの方法で細分できない。そのため、桶に分割する技術を使えてデータをより細かく切る。この技術は指定されたフィールドに沿ってデータが分割され、ファイルの形式で保存される。

　　桶分割の原理はHadoopのMR任務に分割する方式に似てる。

- Hive中：桶フィールド.hashCode % 桶の数量
- MaoReduce中：key.hashCode % reduceTask

**Data**

```
#data3.datデータ
1 java 90
1 c 78
1 python 91
1 hadoop 80
2 java 75
2 c 76
2 python 80
2 hadoop 93
3 java 98
3 c 74
3 python 89
3 hadoop 91
5 java 93
6 c 76
7 python 87
8 hadoop 88
```

**SQL**

```
# 桶テーブルを作成
create table course(
 id int,
 name string,
 score int
)
clustered by (id) into 3 buckets
row format delimited fields terminated by "\t";
```

![image-20230427204657726](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230427204657726.png)

　　本地サービスから直接に仕切りにデータを挿入するように桶に挿入するなら、エラーが出現した。insert... select語句を通じて別のテーブルからデータを桶テーブルに挿入する。又は、Hiveの属性を改修してデータが挿入できる。でも、安全性が考えてお勧めしない。

```
# load data local inpath '/home/hadoop/data/data3.dat' into table course;
```

![image-20230427210249245](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230427210249245.png)

```
# 普通なテーブルを作成
create table data3(
 id int,
 name string,
 score int
)
row format delimited fields terminated by " ";

# データ添加
load data local inpath '/usr/hadoop/data/data3.dat' into table data3;
```

![image-20230427221147985](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230427221147985.png)

```
# データを桶テーブルに挿入
insert into table course select * from data3;
```

![image-20230427222736756](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230427222736756.png)

　　ここでログの内容について少し説明する。

　　例の話通りHiveはデータベースじゃない、Hadoopに基づいて転換器だけ。HDFSファイルをHiveの終端表される内容に転換し、一方データを処理する操作とMR任務の間の転換を行い。後者にはMap-Reduceの概念が理解することが必要です。

　　Hive-Map：一つのジョブが複数の任務に分割される過程です。どのように分割させりると幾つのMap任務が需要だ等過程が考えることが必要ない。

　　Hive-Reduce：複数のMapの処理結果を合併する過程です。合併結果のReduce数量と最初のジョブ数とMap任務数関係ない。

```
# １のジョブの入力から３のReduce任務が出力し、Reduce任務数が桶数です。
Total jobs = 1
Launching Job 1 out of 1 Number of reduce tasks determined at compile time: 3

# 色々な変数が設定される
In order to change the average load for a reducer (in bytes):
 set hive.exec.reducers.bytes.per.reducer=<number>
In order to limit the maximum number of reducers:
 set hive.exec.reducers.max=<number>
In order to set a constant number of reducers:
 set mapreduce.job.reduces=<number>
 
# 最後のReduce数が規定値を超えるため、本地模式が使用できない
Cannot run job locally: Number of reducers (= 3) is more than 1

# 下記は処理過程の情報、何の程度まで進めるのははっきり了解できる。
Hadoop job information for Stage-1: number of mappers: 1; number of reducers: 3
2023-04-27 21:26:52,346 Stage-1 map=0%, reduce = 0%
2023-04-27 21:26:57,621 Stage-1 map = 100%, reduce = 0%, Cumulative CPU 1.32 sec
2023-04-27 21:27:02,811 Stage-1 map = 100%, reduce = 33%, Cumulative CPU 3.1 sec
2023-04-27 21:27:03.843 Stage-1 map = 100%. reduce = 100%. Cumulative CPU 5.64 sec
MapReduce Total cumulative CPu time: 5 seconds 640 msec
```

　　データをcourseテーブルに挿入された。でも特別な物がない。

![image-20230428184358675](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230428184358675.png)

　　データファイルを直接に検査して、３のファイルで保存されており、毎桶に対応している。

![image-20230428194855448](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230428194855448.png)

![image-20230428195620347](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230428195620347.png)

　　区分規則：桶フィールドの値 % 桶数で同じ残りのデータを一つの桶に込める。

```
000000_0: ID % 3 --> 0
000001_0: ID % 3 --> 1
000001_0: ID % 3 --> 2
```

　　SORTED BY語句は桶のデータを並び替えられるが、全体のデータの順序に影響がない。つまり、SORTED BYの作用範囲が一つの桶テーブルだけ、あとは

```
create table courseSort(
 id int,
 name string,
 score int
)
clustered by (id) sorted by (score DESC) into 3 buckets
row format delimited fields terminated by "\t";
```

![image-20230501161429518](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230501161429518.png)

　　実は clustered と sorted に修飾されるフィールドが非数値型だとも許せる。

```
clustered by (name) sorted by (score DESC) into 3 buckets
```

![image-20230501171707455](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230501171707455.png)

　　更にMapとArray集合データを桶フィールドとして使用できる。

```
# 普通のテーブル
create table data4(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>
)
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";

load data local inpath '/usr/hadoop/data/data1.dat' into table data4;

# Mapフィールドに分割
create table data4Bucket1(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>
)
clustered by (addr) sorted by (id DESC) into 3 buckets
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";

insert into table data4Bucket1 select * from data4;
```

![image-20230505174223219](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230505174223219.png)

![image-20230505174228135](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230505174228135.png)

　　実際情況の下に非数値型又は集合に基づいて分割する場合が少ない。一般的には、数値型のフィールドが使用される。桶の分割の目的は大きすぎデータを切り、処理効率が向上させる。しかし、この様な分割方法で得られる計算結果は往々平均ではない。また、開発者にとって複雑な切り方も理解しにくい、データ挿入する際に操作難しさが増えるになる。

　　次は仕切りテーブルに基づいて桶テーブルを作成してみる。

```
create table data2Bucket1(
 id int,
 name string,
 hobby array<string>,
 addr map<string, string>,
 dt string
)
clustered by (id) sorted by (id DESC) into 3 buckets
row format delimited
 fields terminated by ";"
 collection items terminated by ","
 map keys terminated by ":";
 
insert into table data2Bucket1 select * from data2;
```

![image-20230509162144407](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230509162144407.png)

![image-20230509162151226](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230509162151226.png)

　　若し桶テーブルを作成する際に仕切りテーブルの仕切りフィールドがありません、挿入語句を実行してエラーが出現することがある。

![image-20230509171354364](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230509171354364.png)

### 第７項　パーティションテーブルとバケツテーブル区別

　　
