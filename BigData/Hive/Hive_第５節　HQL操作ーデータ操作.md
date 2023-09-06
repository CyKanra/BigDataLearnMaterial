# Hiveデータ倉庫工具ー5

## 第５節　HQL操作ーデータ操作

### 第１項　データの輸入

```
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)]
 
LOAD DATA [LOCAL] INPATH 'filepath' [OVERWRITE] INTO TABLE tablename [PARTITION (partcol1=val1, partcol2=val2 ...)] [INPUTFORMAT 'inputformat' SERDE 'serde'] (3.0 or later)
```

- LOCAL：

  LOAD DATA LOCAL…とは本地サービスからHive表にロード。

  LOAD DATA…とはHDFSファイル系統からHive表にロード。

- INPATH：ロードのデータファイルの経路。
- OVERWRITE：既にあるデータを書き直す。さもなければ、データを追加する操作です。
- PARTITION：データをテーブルの指定される仕切りにロードする。

```
load data local inpath "/home/hadoop/data/t1.dat" into table t3 partition(dt="2020-06-01");
```

**データロード（Load）**

```
#表を作成
CREATE TABLE loadTable (
id int,
name string,
area string
) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;

#データファイルを準備
1,fish1,SZ
2,fish2,SH
3,fish3,HZ
4,fish4,QD
5,fish5,SR

#HDFSにコピー
hdfs dfs -mkdir -p /data/Hive
hdfs dfs -put loadTable.txt /data/Hive
```

![image-20230703145850364](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230703145850364.png)

```
#本地からHiveにロードして、元ファイル保留されておく。
LOAD DATA LOCAL INPATH '/usr/hadoop/data/loadTable.txt' INTO TABLE loadTable;
```

![image-20230703151747956](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230703151747956.png)

```
#データを消除し、外部テーブルにtruncate命令が使えない
truncate table loadTable;

#HDFSからデータをHiveにロードする。HDFS上の元ファイルが消える。
LOAD DATA INPATH '/data/Hive/loadTable.txt' INTO TABLE loadExternalTable;
```

![image-20230703152906733](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230703152906733.png)

![image-20230703152918480](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230703152918480.png)

　　元にHDFS上のデータファイルが消えて、hiveの専門的な元データ管理目録の '/user/hive/warehouse/mydb.db/loadtable' にloadTable.txtを移動する。

![image-20230703153921952](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230703153921952.png)

　　でも、本地の操作はoadTable.txtをこの専門目録にコピーして、元データファイルが依然保留されておく。　　

![image-20230703162943709](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230703162943709.png)

　　上述の違う点は外部テーブルとか内部テーブルとか関わらず、テーブルとファイル名が違いかどうかも構わない、Hiveデータ目録の下に同じ名称のファ イルを新しく作成されてある。どこから輸入だけで決まってる。

**OVERWRITE使う**

```
#元データ上で追加
LOAD DATA LOCAL INPATH '/usr/hadoop/data/loadTable.txt' INTO TABLE loadTable;

#元データを書き直す
LOAD DATA LOCAL INPATH '/usr/hadoop/data/loadTable.txt' OVERWRITE INTO TABLE loadTable;
```

![image-20230703203205453](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230703203205453.png)

**仕切りを追加**

```
LOAD DATA LOCAL INPATH '/usr/hadoop/data/loadTable.txt' INTO TABLE loadTable PARTITION (month = '06');
```

**テーブル作成時にデータを追加**

```
#テーブルを作成する同時にデータをロードする
CREATE TABLE loadTable1 (
id INT
,name string
,area string
) ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' Location '/user/hive/tabB';
```

[LOCATION hdfs_path]から見えて、LOCATION 後の経路はHDFSサービスのみでき、LOCALキーワード属性がない。

### 第２項　データの挿入

```
INSERT OVERWRITE TABLE tablename1 [PARTITION (partcol1=val1, partcol2=val2 ...) [IF NOT EXISTS]] select_statement1 FROM from_statement;
INSERT INTO TABLE tablename1 [PARTITION (partcol1=val1, partcol2=val2 ...)] select_statement1 FROM from_statement;
 

FROM from_statement
INSERT OVERWRITE TABLE tablename1 [PARTITION (partcol1=val1, partcol2=val2 ...) [IF NOT EXISTS]] select_statement1
[INSERT OVERWRITE TABLE tablename2 [PARTITION ... [IF NOT EXISTS]] select_statement2]
[INSERT INTO TABLE tablename2 [PARTITION ...] select_statement2] ...;
FROM from_statement
INSERT INTO TABLE tablename1 [PARTITION (partcol1=val1, partcol2=val2 ...)] select_statement1
[INSERT INTO TABLE tablename2 [PARTITION ...] select_statement2]
[INSERT OVERWRITE TABLE tablename2 [PARTITION ... [IF NOT EXISTS]] select_statement2] ...;

INSERT OVERWRITE TABLE tablename PARTITION (partcol1[=val1], partcol2[=val2] ...) select_statement FROM from_statement;
INSERT INTO TABLE tablename PARTITION (partcol1[=val1], partcol2[=val2] ...) select_statement FROM from_statement;
```

**データ挿入（Insert）**

```
#テーブル作成
CREATE TABLE InsertTable (
id int,
name string,
area string
) 
partitioned by (month string) 
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;

#データ挿入
insert into table InsertTable partition(month = '06') values (10, 'kimura', 'tokyo'), (11, 'tamada', 'chibakei');

#検索結果のデータを挿入
insert into table InsertTable partition(month = '07') select id, name, area from loadTable;
```

![image-20230704130708010](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230704130708010.png)

![image-20230705151614045](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230705151614045.png)

　　挿入（insert）操作を扱って保存されるファイル形式と輸入（load）のが違う。前者は前のテーブル作成を紹介された保存形式とが同じ、仕切り名を目録名として000000_0データファイルが保存される。後者は元データファイルを直接に指定経路にコピー又は移動して、

```
#複数の仕切りデータを同時に挿入
from InsertTable 
insert overwrite table InsertTable partition(month='07') select id, name, area where month='06' 
insert overwrite table InsertTable partition(month='06') select id, name, area where month='07';
```

![image-20230704131925290](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230704131925290.png)

```
#テーブル作成時に検索結果を挿入する
create table if not exists selectTable as select * from InsertTable;
```

![image-20230704134827314](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230704134827314.png)

### 第３項　データの輸出

```
INSERT OVERWRITE [LOCAL] DIRECTORY directory1
  [ROW FORMAT row_format] [STORED AS file_format] (Note: Only available starting with Hive 0.11.0)
  SELECT ... FROM ...
 
Hive extension (multiple inserts):
FROM from_statement
INSERT OVERWRITE [LOCAL] DIRECTORY directory1 select_statement1
[INSERT OVERWRITE [LOCAL] DIRECTORY directory2 select_statement2] ...
 
  
row_format
  : DELIMITED [FIELDS TERMINATED BY char [ESCAPED BY char]] [COLLECTION ITEMS TERMINATED BY char]
        [MAP KEYS TERMINATED BY char] [LINES TERMINATED BY char]
        [NULL DEFINED AS char] (Note: Only available starting with Hive 0.13)
```

**データの輸出**

```
#検索結果を本地サービスに輸出
insert overwrite local directory '/usr/hadoop/data/InsertTable' select * from InsertTable;
```

![image-20230705151303933](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230705151303933.png)

![image-20230705151334912](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230705151334912.png)

　　先ず輸入の過程はLoadみたいキーワードがありません、データの輸入といっても逆に挿入することです。而も、仕切りテーブルのデータを輸出してHiveの保存される目録結構ような作成しません、全てのデータを一つのファイルに輸出した。仕切りフィールドの値があるけれど、他の仕切り情報がありません。

　　値の間に区切り文字は黙認符号、読みが不便です。

![image-20230707114017629](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230707114017629.png)

　　若し輸出したデータを再利用し、仕切りフィールドの値が含んでデータファイルには中々不便です。仕切りにデータをロードする際に、仕切り値はSql語句によって決まって、データファイルに元にある値が採用してない。却ってデータを導入できるため、仕切り値を消除するのを工夫する。そもそもデータの完全性を保して、ここで余計なものになる。

```
#格式化にファイルを輸出
insert overwrite local directory '/usr/hadoop/data/InsertTable' row format delimited fields terminated by ' ' select id, name, area from InsertTable;
```

![image-20230707113712442](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230707113712442.png)

```
#検索結果をHDFSに輸出
insert overwrite directory '/data/Hive/InsertTable' row format delimited fields terminated by ' ' select id, name, area from InsertTable;
```

![image-20230707145220338](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230707145220338.png)

　　Hive本質はファイル解析機、データファイルに対してどこでも本体が変わりません、直接にデータファイルをコピーしていい。

```
#dfs命令を使って本地までにコピー
hadoop fs -get /user/hive/warehouse/mydb.db/inserttable/month=06 /usr/hadoop/data/inserttable

dfs -get /user/hive/warehouse/mydb.db/inserttable/month=06 /usr/hadoop/data/inserttable
```

![image-20230707164404762](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230707164404762.png)

**Linuxのリダイレクト輸出**

Linux自分のリダイレクト機能を使って、目標ファイルにデータを輸出。

```
hive -e "select * from mydb.insertTable" > insertTable.log
```

![image-20230712141912444](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230712141912444.png)

### 第４項　Import導入／Export導出

```
IMPORT [[EXTERNAL] TABLE new_or_original_tablename [PARTITION (part_column="value"[, ...])]] 
FROM 'source_path' 
[LOCATION 'import_target_path']

EXPORT TABLE tablename [PARTITION (part_column="value"[, ...])] TO 'export_target_path' [ FOR replication('eventid') ]
```

データ準備

```
#新しいテーブルを作成
CREATE TABLE exportTable (
id int,
name string,
area string
) 
partitioned by (month string) 
ROW FORMAT DELIMITED FIELDS TERMINATED BY ',' ;

#データ
1,student1,SZ
2,student2,SH
3,student3,HZ
4,student4,QD
5,student5,SR
6,student6,SB

#load輸入
LOAD DATA LOCAL INPATH '/usr/hadoop/data/exportTable.txt' INTO TABLE exportTable PARTITION (month = '06');
LOAD DATA LOCAL INPATH '/usr/hadoop/data/exportTable1.txt' INTO TABLE exportTable PARTITION (month = '07');
```

![image-20230711154832155](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711154832155.png)

![image-20230711160738022](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711160738022.png)

**Export導出**

```
export table exportTable to '/data/Hive/export/exportTable';
```

![image-20230711180300214](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711180300214.png)

![image-20230711180153127](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711180153127.png)

![image-20230711180321340](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711180321340.png)

　　ここ見えて、export方法を使って導出し、導出されたファイル結構と元Hiveのテーブルの結構が同じです。まだ、もう一つメーターデータのファイルが作成された。その機能は前に紹介したの挿入（insert）など方法が実現できない。

　　Export導出語句は　[PARTITION (part_column="value"[, ...])]　変数が含まれて、個別の仕切りを抽出できる。

```
export table exportTable partition (month = "06") to '/data/Hive/export/exportTable';
```

　　メーターデータを開けて、ファイルにJson格式で色々なテーブル情報が保存されたと見える。

```
hdfs dfs -cat /data/Hive/export/_metadata
```

![image-20230711164622397](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711164622397.png)

**Import導入**

```
#相同的なテーブルを作成（この手筈が略すれる）
create table importTable like exportTable;

#import操作
import table importTable from '/data/Hive/export/exportTable';
```

![image-20230711191317236](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711191317236.png)

![image-20230711192212363](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230711192212363.png)

　　ExportとImport命令は完全なテーブルをコピーできる利点に対して、データ倉庫の間にデータを伝送する場合で適当に扱う。でも、一般的にビッグデータの伝送には、特別にデータを手動に引導するつもり限り、専門の工具を使用してお勧めです。

