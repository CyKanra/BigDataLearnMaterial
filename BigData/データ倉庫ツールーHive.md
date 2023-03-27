# Hiveデータ倉庫工具ー２

## 第２節　Hive取り付けと配置

### 第１項　Hiveの取り付け

　　第１節に紹介した、Hiveは唯SQLを MapReduceの任務に変換する工具、データベースじゃない。なら、HDFSにデータを読み込んでMR任務に変換する時に、必ず何かに基づいて変換できる、それはメタデータ（MateData）。

　　Hiveの取り付けは二部分に分ける、MateDataとHive。

　　MateDataのメモリーはここでRDBMSデータベースMySQLを選び、関係性データベースに選んで保存することは必要です。Hiveの黙認データベースはDerby、Java言語開発為占用される資源が少ない、でも単独のプロセスとユーザー、唯自分の学習に合う、一般に実際の環境はMySQLが採用される、本節もMySQLを使って紹介する。

　　サービスの分配：

| ソフト | Centos1 | Centos2 | Centos3 | Centos4 |
| ------ | ------- | ------- | ------- | ------- |
| Hadoop | √       | √       | √       | √       |
| Hive   |         | √       |         |         |
| MySQL  |         | √       |         |         |

　　必要なパッケージ：

```
#hive取り付けパッケージ
#バージョン：2.3.7
apache-hive-2.3.7-bin.tar.gz

#Mysql取り付けパッケージ
#バージョン：5.7.26
mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar

#MysqlのJDBCドライブ
#バージョン：5.1.46
mysql-connector-java-5.1.46.jar

#全ての取り付けの流れを説明
１．MySQL取り付け
２．Hive取り付け
３．Hiveの常用配置を追加
```

＃備考：出来るだけ文章に説明されるバージョンによって使って取り付け、色々意外が発生すると避けられる。

Hive公式：[[Apache Hive](https://hive.apache.org/)](https://hive.apache.org/)

ダウンロードURL：[Index of /dist/hive (apache.org)](http://archive.apache.org/dist/hive/)

資料URL：[LanguageManual - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual)

#### 2.1.1　MySQL取り付け

MySQL5.7.26取り付け流れ

```
１．環境整備
２．MySQL取り付け
３．Rootパスワード改修
４．MySQLにHiveユーザー作成
```

**１．関連のパッケージをアップロード**

rz命令を行う。

![image-20230310143153488](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310143153488.png)

**２．MariaDB削除**

Centos7.6自体のMariaDB（MariaDBはMySQLの分脈、もデータベースです）とMySQLは衝突があり、ここでCentos7.6のMariaDBを選んで削除する。

```
#　MariaDB存在するかどうかを確認する
rpm -aq | grep mariadb

#　MariaDBを削除する
#　-e　指定されるキットを削除する
#　--nodeps　キットの関連性が不検証です
rpm -e --nodeps mariadb-libs
```

![image-20230310154322003](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310154322003.png)

**３．依頼の取り付け**

```
yum install perl -y
yum install net-tools -y
```

![image-20230310155228521](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310155228521.png)

![image-20230310155234811](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310155234811.png)

**４．MySQL取り付け**

```
#　パッケージ解凍
tar xvf mysql-5.7.26-1.el7.x86_64.rpm-bundle.tar

#　序に命令を行う
rpm -ivh mysql-community-common-5.7.26-1.el7.x86_64.rpm
rpm -ivh mysql-community-libs-5.7.26-1.el7.x86_64.rpm
rpm -ivh mysql-community-client-5.7.26-1.el7.x86_64.rpm
rpm -ivh mysql-community-server-5.7.26-1.el7.x86_64.rpm
```

![image-20230310161253164](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310161253164.png)

**５．Mysql起動**

```
#　status：状態検査、start：起動、stop：停止
systemctl start mysqld
```

![image-20230310164036379](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310164036379.png)

**６．rootパスワードを探す**

パスワードをコーピしてMySQLの操作画面に進入する。

```
# パスワード探す
grep password /var/log/mysqld.log

#　MySQL進入，前に探したパスワードを入力
mysql -u root -p
```

![image-20230310164647266](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310164647266.png)

**７．rootのパスワード改修**

```
#　パスワードの強度を改修する
set global validate_password_policy=0;

#　rootパスワードを12345678に変更する
set password for 'root'@'localhost' =password('12345678');

#　設置を刷新する
flush privileges;
```

![image-20230310170911138](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310170911138.png)

**８．hiveユーザー作成**

```
#　ユーザーの作成、授権、刷新
CREATE USER 'hive'@'%' IDENTIFIED BY '12345678';
GRANT ALL ON *.* TO 'hive'@'%';
FLUSH PRIVILEGES;
```

![image-20230310171901918](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230310171901918.png)

以上の説明に因って全部完成したら、ここまでMySQLの取り付けが終わった。

#### 2.1.2　Hive取り付け

Hive取り付けの流れ

```
１．サービスにアップロード、解凍
２．環境変数の改修
３．hive配置ファイル改修
４．JDBCドライブのアップロード
５．メタデータベースの初期化
```

**１．取り付けパッケージ解凍**

rz命令を行って取り付けパッケージをアップロードし、後解凍して、ファイル名が長い為ここで短くなって改修する、でもバージョン番号保留が必要だ。

```
tar zxvf apache-hive-2.3.7-bin.tar.gz
#　若し取り付けパッケージが他の目録に居れば、下の命令が使える
tar zxvf apache-hive-2.3.7-bin.tar.gz -C ../servers/

#　名を改修する
mv apache-hive-2.3.7-bin hive-2.3.7
```

![image-20230321092253755](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321092253755.png)

**２．環境変数改修**

```
#　/etc/profile ファイルを編輯して、最後に下の２行を追加する
vim /etc/profile

#　最後に下の２行を追加する
export HIVE_HOME=/opt/bigdata/servers/hive-2.3.7
export PATH=$PATH:$HIVE_HOME/bin

# 执行并生效
source /etc/profile

# 改修の成功かどうかを確認する
cd $HIVE_HOME/
```

![image-20230321100759324](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321100759324.png)

![image-20230321100923293](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321100923293.png)



**３．Hive配置ファイル改修**

$HIVE_HOME/conf 目録に hive-site.xml ファイルを創建し、以下の内容をファイルに追加する。

```
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
	<!-- hiveメタデータのメモリー位置-->
	<property>
		<name>javax.jdo.option.ConnectionURL</name>
		<value>jdbc:mysql://centos2:3306/hivemetadata?createDatabaseIfNotExist=true&amp;useSSL=false</value>
		<description>JDBC connect string for a JDBCmetastore</description>
	</property>
	<!-- 指定するドライバー -->
	<property>
		<name>javax.jdo.option.ConnectionDriverName</name>
		<value>com.mysql.jdbc.Driver</value>
		<description>Driver class name for a JDBCmetastore</description>
	</property>
	<!-- データベースを連絡するユーザー名 -->
	<property>
		<name>javax.jdo.option.ConnectionUserName</name>
		<value>hive</value>
		<description>username to use against metastoredatabase</description>
	</property>
	<!-- データベースを連絡するパスワード -->
	<property>
		<name>javax.jdo.option.ConnectionPassword</name>
		<value>12345678</value>
		<description>password to use against metastoredatabase</description>
	</property>
</configuration>
```

![image-20230321100659486](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321100659486.png)

![image-20230321100705071](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321100705071.png)

＃備考：

１．若しhive-site.xml ファイルなければ新しく創建していい。

２．&amp;useSSL=false 大量警告が消除できる。

**４．JDBCドライブのコーピ**

JDBCドライブ mysql-connector-java-5.1.46.jar を $HIVE_HOME/lib にコーピする

![image-20230321120601443](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321120601443.png)

**５．メタデータを初期化にする**

$HIVE_HOME/lib 目録に進入し、下の命令を行う。

```
schematool -dbType mysql -initSchema
```

![image-20230321122330240](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321122330240.png)

若し一切順調に進めば、MySQLに新しいデータベース hivemetadata を作成すした。

![image-20230321122351342](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321122351342.png)

**６．Hive起動**

![image-20230321123059872](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321123059872.png)

＃備考：Hive起動前に必ずHadoopサービスが運行する状態です。

以上の説明に因って全部完成したら、ここまで【Hiveの取り付け】内容が終わった。

#### 2.1.3	Hive属性配置

この部分の内容は幾つか属性配置を追加してもっと便利にHiveを使用できる。

**データの保存位置**

データの保存位置はHDFSです。配置の改修が必要ない、でもデータはどこに保存することが了解すべきです。

```
<property>
	<!-- データの黙認保存位置(HDFS) -->
	<name>hive.metastore.warehouse.dir</name>
	<value>/user/hive/warehouse</value>
	<description>location of default database for thewarehouse</description>
</property>
```

**目下のデータベース名を表す**

```
<property>
	<!-- コマンド画面操作の下，目下のデータベースを表す -->
	<name>hive.cli.print.current.db</name>
	<value>true</value>
	<description>Whether to include the current database in theHive prompt.</description>
</property>
```

**列名を表す**

```
<property>
	<!-- コマンド画面操作の下に，データ列名を表す -->
	<name>hive.cli.print.header</name>
	<value>true</value>
</property>
```

**本地模式**

```
<property>
	<!-- 小規模データの下に本地サービス行う、効率上げる -->
	<name>hive.exec.mode.local.auto</name>
	<value>true</value>
	<description>Let Hive determine whether to run in localmode automatically</description>
</property>
```

＃備考：Hiveに入力するデータ量が中々小さい時は、本地模式を通じて単節点に全部の任務を処理する。この状態で掛かる時間が明らかに減れる。

以下は本地模式を触発する条件：

- 任務のデータ量は変数より小さい

  hive.exec.mode.local.auto.inputbytes.max (黙認128MB)

- 任務のmap数量は変数より小さい

  hive.exec.mode.local.auto.tasks.max (黙認4)

- 任務のreduce数量は0又は1

**Hiveログファイル**

conf 目録に hive-exec-log4j2.properties.template ファイルあって、Hiveログの保存位置とファイル名が記録する。

![image-20230321171003834](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321171003834.png)

![image-20230321180429725](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321180429725.png)

若し他の位置に改修すれば、下の手順に因って行う。

```
# 新しいファイル創建
vim $HIVE_HOME/conf/hive-log4j2.properties

# 以下の内容を追加
property.hive.log.dir = /opt/bigdata/servers/hive-2.3.7/logs
```

ここまで【Hiveの属性配置】内容が終わった。

#### 2.1.4　変数を配置する方法

変数を配置するのは何ですか、例えば前に説明したの本地模式、配置ファイルに対応の変数は true 設置され、その流れは変数の配置、も一つの方法です。

```
# Hive操作画面に進入
# 全部を調べる
set;

# 具体的な変数を調べる
set hive.exec.mode.local.auto;
```

![image-20230321193107093](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321193107093.png)

変数配置の三つの方法

**１．配置ファイル方法**

黙認配置ファイル：hive-default.xml

カスタマイズの配置ファイル：hive-site.xml

配置の優先度：hive-site.xml > hive-default.xml

全てのHiveプロセスが有効する。

**２．Hive起動する際に変数を追加する**

キーワード -hiveconf を通じて具体的な変数を追加し、只今回の起動が有効する。

```
# 起動際に命令を行う
hive -hiveconf hive.exec.mode.local.auto=false
```

![image-20230321200459838](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321200459838.png)

**３．命令で変数を改修する**

Hive操作画面にキーワード set を通じて具体的な変数を改修し、只今回の起動が有効する

```
# 運行際に命令を行う
set hive.exec.mode.local.auto=false;
```

![image-20230321202021451](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230321202021451.png)

三つの方法の優先度：

set > -hiveconf > hive-site.xml > hive-default.xml

ここまで【変数を配置する方法】内容が終わった。

### 第２項　Hive命令

本項は入門的な命令を紹介し、主にどうHiveを操作する命令、具体的なデータ操作が及ばない。

**１．Hive**

```
usage: hive
 -d,--define <key=value>          Variable substitution to apply to Hive
                                  commands. e.g. -d A=B or --define A=B
    --database <databasename>     Specify the database to use
 -e <quoted-query-string>         SQL from command line
 -f <filename>                    SQL from files
 -H,--help                        Print help information
    --hiveconf <property=value>   Use value for given property
    --hivevar <key=value>         Variable substitution to apply to Hive
                                  commands. e.g. --hivevar A=B
 -i <filename>                    Initialization SQL file
 -S,--silent                      Silent mode in interactive shell
 -v,--verbose                     Verbose mode (echo executed SQL to the
                                  console)
```

以上の命令は全部Linuxコマンドに操作し、幾つか例を引って解説する。

```
# 進入前にデータベースを指定する
hive --database test

# 直接にsqlを行う
hive -e "select * from users"

# sqlファイルも行える
hive -f hiveSql.sql

# 行い終える結果は指定されるファイルに書き込む
hive -f hiveSql.sql >> result.log

```

**２．Hiveコマンドに sell / dfs 命令を行う**

```
! ls;
! clear;
dfs -ls / ;
```

**３．Hive退出**

```
exit;
quit;
```

もっとHive命令に関する資料：

[LanguageManual Commands - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+Commands)

ここまで本節終わった、お役立てれば良かったんです。
