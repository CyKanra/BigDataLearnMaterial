# Hiveデータ倉庫工具ー8

## 第８節　HQL操作--DML命令

　　データ操作言語DML（Data Manipulation Language）、主に３類形式あり：挿入（INSERT）、消除（DELETE）、更新（UPDATE）。

　　トランザクション（transaction）はデータを処理する際、互いに関連、依存の複数の処理を纏め、一体不可分の処理単位として扱うという意味です。

　　ACIDはトランザクションの４つの要素えす。それぞれは’原子性（Atomicity）、一貫性（Consistency）、隔離性（Isolation）、永続性（Durability）。

- 原子性：トランザクションは分割できない作業単位であり、トランザクション内のすべての操作はすべて実行されるか、またはすべて実行されないかのいずれかです。
- 一貫性：トランザクションの一貫性は、トランザクションの実行がデータベースのデータの完全性と整合性を壊すことができないことを意味します。トランザクションは実行する前と実行した後、データベースの状態は一致です。
- 隔離性。並行環境で、並行して実行されるトランザクションは互いに隔離されており、1つのトランザクションの実行が他のトランザクションに干渉されないことを意味します。
- 永続性：トランザクションがコミットされると、データベース内のデータの変更は永続的であるべきです。

公式文献：[Hive Transactions - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/Hive+Transactions)

### 第１項　Hiveのトランザクション

　　隔離級別：直列化（SERIALIZABLE）＞繰り返し読込（REPEATABLE READ）＞コミットされた読込（READ COMMITTED）＞コミットされない読込（READ UNCOMMITTED）、隔離性が強さから弱さまで区切られた。　

　　HiveのトランザクションはACIDの４つ要素が備わっているが、Hiveの隔離級別はスナップショット級別が支持されて、而も行のみに効いてある。つまり、繰り返し読込が保証できない、多テーブルのデータを更新する中に他のスレッドが入って、一部分まだ更新されないデータが読み込まれる可能性がある、又はデータが控えを取ってる時、バックアップのデータが不一致となる問題も発生してある。不同点は前者が事実上データ錯誤がない、後者の錯誤が永久的です。

**トランザクションの制限**

- Hiveのスナップショット機能は行単位に対して、多行を更新される必要の場合、読み込まれた行の間に関連性が保証できない。
- BEGIN、COMMIT、及びROLLBACKはまだ支持されない。すべての言語操作は自動にコミットです。
- 最初のリリースではORCファイル形式のみが支持されている。
- 黙認状況にトランザクションは無効に設定されている。
- 内部テーブルと桶テーブル条件が必要です。
- ACIDテーブルから非ACID読み書き模式は許可されてない。後は、Hiveトランザクション管理をorg.apache.hadoop.hive.ql.lockmgr.DbTxnManagerに設定する必要がある。
- 現時点ではスナップショット級別の分離のみが支持されている。
- LOAD DATA...を通じてのトランザクション操作は支持されない。
- 直接にデータファイルを改修することは支持されない。

### 第２項　Hiveのトランザクション案例

```
#Hive会話画面に設置させて今度会話のみが効いてあり、hive-site.xml中設置しては永久です
SET hive.support.concurrency = true;
SET hive.enforce.bucketing = true;
SET hive.exec.dynamic.partition.mode = nonstrict;
SET hive.txn.manager = org.apache.hadoop.hive.ql.lockmgr.DbTxnManager;
```

![image-20230823164226612](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230823164226612.png)

```
#データ準備
1,kimura,08033537653,1993-11-28
2,suzuki,09012345678,1985-06-15
3,tanaka,07098765432,1990-02-03
4,yamamoto,08011112222,1998-09-20
5,ito,09044445555,1982-12-10
6,sato,08077778888,1991-04-22
7,takahashi,09099990000,1995-08-03
8,okada,07055556666,1988-11-17
9,nakamura,08022223333,2000-03-08
10,kobayashi,09066667777,1987-09-12
11,ishikawa,08088889999,1994-01-25
12,abe,09022221111,1999-07-30
13,watanabe,07077774444,1992-06-05
14,hayashi,08044445555,1986-12-15
15,kato,09033334444,1997-10-18

#トランザクションテーブルを作成（内部テーブル、桶、ORC格式メモリー条件）
create table transTable(
id int,
name string,
phone string,
birth date
)
clustered by(id) into 3 buckets
stored as orc
tblproperties('transactional'='true');

#臨時テーブル、データをtransTableに導入するため。
create table temp(
id int,
name string,
phone string,
birth date
)
row format delimited
fields terminated by ",";

#データ輸入
load data local inpath '/usr/hadoop/data/transTable.dat' overwrite into table temp;
insert into table transTable select * from temp;
```

![image-20230823173659454](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230823173659454.png)

![image-20230823173704843](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230823173704843.png)

　　ここ普通のテーブルnoTransTableも作成するとトランザクションテーブルに比較して、目録の結構が不同と見える。

![image-20230824105913671](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230824105913671.png)

![image-20230824105922002](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230824105922002.png)

　　両者のファイル名称が違うだけではない、delta_0000001_0000001_0000名の目録が余分一つとあり、それは毎トランザクションコミット過程の中で類似の目録を追加しており、当の目録下のファイルが今度の改修内容を保存してある。具体的な説明は次の項に紹介してあげる。

```
delete from transTable where id in (3,12,10,2);
```

![image-20230825070402462](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230825070402462.png)

```
insert into transTable values (3, "abe", "09022221111", "1999-07-30");
```

![image-20230825071336290](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230825071336290.png)

```
insert into transTable values 
(10, "tanaka", "07098765432", "1990-02-03"), 
(12, "kobayashi", "09066667777", "1987-09-12");
```

![image-20230825072412181](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230825072412181.png)

```
#桶フィールドを改修することはできない
update transTable set name="suzuki", phone="09012345678", birth="1985-06-15" where id=12;
update transTable set name="sasaki", phone="08012345678", birth="1996-05-20" where id=1;
```

![image-20230825072838881](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230825072838881.png)

　　トランザクションのテーブルにはLOAD DATA語句が支持されない

```
16,sasaki,08012345678,1996-05-20
17,okamoto,09087654321,1989-02-14
18,mori,07011112222,1992-11-07
19,shimizu,08099998888,1997-09-28
20,fujita,09055556666,1984-08-09

LOAD DATA LOCAL INPATH '/usr/hadoop/data/addTransTable.dat' INTO TABLE transTable;
```

![image-20230825194058270](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230825194058270.png)

```
set hive.strict.checks.bucketing = false;
LOAD DATA LOCAL INPATH '/usr/hadoop/data/addTransTable.dat' INTO TABLE transTable;
```

![image-20230825194720392](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230825194720392.png)

　　Hiveはトランザクションが支持できるんですが、伝統のデータベースに比べて自由度が高くない、中々小さい制限があり、例えば、NULL値の入力が許さない。具体的な紹介は下の公式文献から了解できる。

公式文献：[LanguageManual DML - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+DML)

### 第３項　Hiveのトランザクション基本設計

