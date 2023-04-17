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

Data

```
#data1.datデータ
1;suzuki;book,TV;tokyo:sinbuyakun
2;yamada;Music,game,novel;kanagawaken:kawasaki,tokyo:itabasikun
3;kimura;skiing,animation;chibaken:sakurasi
```

SQL

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

　　ある時、特定の条件に一致する部分のデータを走査するだけで、仕切り観念を引っ込み、不同的な目録に相同的な条件のデータを保存させて、毎目録が一つの仕切りに対応する。仕切りの内部に詮索するのは、全てのデータの走査が必要ない、処理の効率が高まる。

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

**仕切り検査**

```
show partitions data2;
```

![image-20230417215112458](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230417215112458.png)

![image-20230417215125655](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230417215125655.png)

HDFSにデータの保存形式からして、データの仕切りは不同的なファイルに分割して保存し、仕切りの相互に重複のデータが存在することは許す。
