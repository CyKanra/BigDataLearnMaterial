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

4. partition by：データを仕切り（partition）に分割してパーティションのフィールドを指定してください。一般的に日付や地域別に分割することがよくある。

   ```
   CREATE TABLE my_table (
     column1 INT,
     column2 STRING
   )
   PARTITIONED BY (date STRING);
   ```

5. clustered by：パーティションされるデータを桶（bucket）に分割して、桶のフィールドを指定してください。

6. sorted by：桶内のデータを並び替える

