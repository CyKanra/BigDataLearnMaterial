# Hiveデータ倉庫工具ー6

## 第６節　HQL操作--DQL命令

DQL -- Data Query Language データ検索語句

### 第１項　select 検索語句

```
SELECT [ALL | DISTINCT] select_expr, select_expr, ...
FROM table_reference
[WHERE where_condition]
[GROUP BY col_list]
[ORDER BY col_list]
[CLUSTER BY col_list | [DISTRIBUTE BY col_list] [SORT BY
col_list]]
[LIMIT [offset,] rows]
```

データ準備

```
#データ
1,Haruki Nakamura,CLERK,20,2010-12-17,448800
2,Aiko Tanaka,SALESMAN,30,2011-02-20,311600
3,Yuya Suzuki,SALESMAN,20,2011-02-22,411250
4,Emi Yamamoto,MANAGER,10,2011-04-02,352975
5,Kazuki Sato,SALESMAN,30,2011-09-28,361250
6,Mei Kobayashi,MANAGER,20,2011-05-01,332850
7,Hiroshi Takahashi,MANAGER,30,2011-06-09,322450
8,Rina Nakajima,ANALYST,40,2017-07-13,363000
9,Shinji Aoki,PRESIDENT,30,2011-11-07,355000
10,Mio Hayashi,SALESMAN,20,2011-09-08,381500
11,Yukihiro Kimura,CLERK,40,2017-07-13,341100
12,Ayaka Nakamura,CLERK,40,2011-12-03,373950
13,Miku Sato,ANALYST,10,2011-12-03,323000
14,Masaki Nakajima,CLERK,20,2012-01-23,351300

#employeeTable作成
CREATE TABLE employeeTable (
id int,
name string,
job string,
department int,
hiredate DATE,
sal int
)row format delimited fields terminated by ",";

#データ輸入
LOAD DATA LOCAL INPATH '/usr/hadoop/data/employeeTable.dat' INTO TABLE employeeTable;

#データ
10,develop
20,financial
30,market
50,administrative

#departmentTable作成
CREATE TABLE departmentTable (
id int,
name string
)row format delimited fields terminated by ",";

#データ輸入
LOAD DATA LOCAL INPATH '/usr/hadoop/data/departmentTable.dat' INTO TABLE departmentTable;
```

**基本検索**

SQLと基本的に一致です

```
#fromを略して検索
select 100;
select current_date;

#別名を使う
select 100 number;
select current_date as currdate;

#全ての検査
select * from employeeTable;

#指定フィールドを検査
select name, sal from employeeTable;

#函数使う、NULL値が含まれない
select count(*) from employeeTable;
select sum(sal) from employeeTable;
select max(sal) from employeeTable;
select avg(sal) from employeeTable;

#返る結果を制限
select * from employeeTable limit 3;

#where条件を付け
select * from employeeTable where id=2;
```

### 第２項　where 条件検索語句

```
#where条件を付け
select * from employeeTable where id=2;
```

**比較演算子**

| 演算子                   | 類型 | 描写                                                         |
| ------------------------ | ---- | ------------------------------------------------------------ |
| A = B                    | all  | AがBと等しい場合はTRUE、それ以外の場合はFALSEを返す          |
| A == B                   | all  | = と同じ                                                     |
| A <=> B                  | all  | 非NULLのオペランドに対してはEQUAL(=)演算子と同じ結果を返しますが、両方がNULLの場合はTRUEを返し、いずれかがNULLの場合はFALSEを返す（バージョン0.9.0以降） |
| A <> B                   | all  | AまたはBがNULLの場合はNULLを返し、AがBと等しくない場合はTRUEを返し、それ以外の場合はFALSEを返す |
| A != B                   | all  | <> と同じ                                                    |
| A < B                    | all  | AまたはBがNULLの場合はNULLを返し、AがBより小さい場合はTRUEを返し、それ以外の場合はFALSEを返す |
| A <= B                   | all  | AまたはBがNULLの場合はNULLを返し、AがB以下の場合はTRUEを返し、それ以外の場合はFALSEを返す |
| A > B                    | all  | AまたはBがNULLの場合はNULLを返し、AがBより大きい場合はTRUEを返し、それ以外の場合はFALSEを返す |
| A >= B                   | all  | AまたはBがNULLの場合はNULLを返し、AがB以上の場合はTRUEを返し、それ以外の場合はFALSEを返す |
| A [NOT] BETWEEN B AND C  |      | AまたはBまたはCがNULLの場合はNULLを返し、AがB以上かつC以下の場合はTRUEを返し、それ以外の場合はFALSEを返します。NOTキーワードを使用すると、結果を反転させることができます（バージョン0.9.0以降） |
| A IS NULL                |      | AがNULLの場合はTRUEを返し、それ以外の場合はFALSEを返す       |
| A IS NOT NULL            |      | AがNULLの場合はFALSEを返し、それ以外の場合はTRUEを返す       |
| A IS [NOT] (TRUE\|FALSE) |      | Aが条件を満たす場合にのみTRUEと評価される（バージョン3.0.0以降） |
| A [NOT] LIKE B           |      | AまたはBがNULLの場合はNULLを返し、文字列AがSQLのシンプルな正規表現Bに一致する場合はTRUEを返し、それ以外の場合はFALSEを返します。比較は文字ごとに行われます。Bの文字はAの任意の文字に一致します、Bの%文字はAの任意の数の文字に一致します。たとえば、'foobar' like 'foo'はFALSEを評価し、'foobar' like 'foo_ _ _'はTRUEを評価します。また、'foobar' like 'foo%'もTRUEと評価されます |
| A RLIKE B                |      | AまたはBがNULLの場合はNULLを返し、Aの任意の（空でも可能な）部分文字列がJavaの正規表現Bに一致する場合はTRUEを返し、それ以外の場合はFALSEを返します。たとえば、'foobar' RLIKE 'foo'はTRUEを評価し、'foobar' RLIKE '^f.*r$'もTRUEを評価します |
| A REGEXP B               |      | RLIKE と同じ                                                 |

備考：普通にNULLは運算中に含まれ、戻り値がNULLです（NULL<=>NULL結果がtrue）

```
#like使う
select name, sal from employeeTable where name like '%ka%';

#rlike使って、正規表現と取り組み、Ka又はHaを始めとする名前で検査
select name, sal from employeeTable where name rlike '^(Ka|Ha).*';
```

![image-20230717143921668](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230717143921668.png)

**論理演算子**

| 演算子                     | 類型    | 描写                                                         |
| -------------------------- | ------- | ------------------------------------------------------------ |
| A AND B                    | Boolean | AとBが両方ともTRUEの場合はTRUEを返し、それ以外の場合はFALSEを返す。AまたはBがNULLの場合はNULLを返す |
| A OR B                     | Boolean | AまたはBまたは両方がTRUEの場合はTRUEを返し、FALSEまたはNULLはNULLを返し、それ以外の場合はFALSEを返す |
| NOT A                      | Boolean | AがFALSEまたはNULLの場合はTRUEを返し、それ以外の場合はFALSEを返します。AがNULLの場合はNULLを返す |
| ! A                        | Boolean | NOT と同じ                                                   |
| A IN (val1, val2, ...)     | Boolean | Aが値のいずれかに等しい場合はTRUEを返す。Hive 0.13以降、サブクエリが支持されている |
| A NOT IN (val1, val2, ...) | Boolean | Aが値のいずれとも等しくない場合はTRUEを返す。Hive 0.13以降、サブクエリが支持されている |
| [NOT] EXISTS (subquery)    |         | サブクエリが少なくとも1行を返す場合はTRUEを返します。Hive 0.13以降でサポートされている |

```
#EXISTS 使う
select * from employeeTable a 
where exists (
    select * 
    from departmentTable b 
    where b.id = a.department
);
```

![image-20230717162128107](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230717162128107.png)

　　この語句はemployeeTableテーブルを検索する結果がdepartmentTableテーブルに関連して存在するデータのみが表すという意味です。上の図が見えて、departmentTable.idイコール'40'のデータがないために、employeeTableにdepartment値イコール'40'のデータをが表れない。

### 第３項　group by 組み分け検索語句

　　group by語句は常に函数と一緒に使用し、単数又は複数のフィールドで組み分けて、毎組が合併に計算する。

```
#毎部門の料金の平均値
select department, avg(sal) from employeeTable group by department;

#毎部門毎職務に一番高い料金値
select department, job, max(sal) from employeeTable group by department, job;
```

![image-20230718154144235](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230718154144235.png)

![image-20230718154111366](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230718154111366.png)

**Havingキーワード**

```
-- 求每个部门的平均薪水大于2000的部门
select department, avg(sal) from employeeTable group by department having avg(sal) > 2000;
```

![image-20230718161823402](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230718161823402.png)

whereとhavingの区別

- where語句はテーブルデータに対して制限を付ける。havingは検索結果に対して制限を付ける。
- where語句は函数を使えない、havingのができる。
- havingはgroup by組み分け語句の後ろに使えるだけ。

### 第４項　テーブル結合

　　Hiveは SQL Join 機能が支持し、でも、黙認状況に等値結合が支持するだけ、不等値結合ができない。

1. 内部結合：[inner] jion

   両テーブルで指定したカラムの値が一致するデータのみを抽出する

2. 外部結合 (outer jion)

   - 左外部結合：left [outer] join 

     左のテーブルのデータが全て表れる。右の存在しないデータがnullで表れる。

   - 右外部結合：right [outer] join 

     右のテーブルのデータが全て表れる。左の存在しないデータがnullで表れる。

   - 全外部結合：full [outer] join

     両テーブルのデータが全て表れる。存在しないデータがnullで表れる。

![image-20230718193043565](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230718193043565.png)

```
#左結合、departmentTableに存在ないデータ（b.id=40）がNULLで表れる
select a.id, a.department, a.name, b.name from employeeTable a left join departmentTable b on a.department=b.id;
```

![image-20230719142556578](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230719142556578.png)

```
#右結合、employeeTableに存在ないデータ（a.department=50）がNULLで表れる
select a.id, a.department, a.name, b.name from employeeTable a right join departmentTable b on a.department=b.id;
```

![image-20230719142741292](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230719142741292.png)

```
#内結合、データ本体はNULL値がなければ、検索結果が全部存在だ。空値を除いて結果に類似する
select a.id, a.department, a.name, b.name from employeeTable a join departmentTable b on a.department=b.id;

#外結合、結果データ数が両テーブル数の積
select a.id, a.department, a.name, b.name from employeeTable a full join departmentTable b on a.department=b.id;
```

![image-20230719144810233](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230719144810233.png)

![image-20230719144858558](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230719144858558.png)

**複数テーブル結合**

```
#データ
1,CLERK
2,SALESMAN
3,MANAGER
4,ANALYST
5,PRESIDENT

#jobTable作成
CREATE TABLE jobTable (
id int,
name string
)row format delimited fields terminated by ",";

#データ輸入
LOAD DATA LOCAL INPATH '/usr/hadoop/data/jobTable.dat' INTO TABLE jobTable;

#三つのテーブルを結合して検索し
select a.id, a.name, a.department, b.name, a.job from employeeTable a left join departmentTable b on a.department=b.id left join jobTable c on a.job=c.name;
```

![image-20230719162657083](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230719162657083.png)

**デカルト積**

　　デカルト積又は直積、黙認状況にHiveはデカルト積が支持しない。以下の条件はデカルト積の発生が触発する。

- 関連条件ない
- 関連条件が無効
- 全てのデータが関連に検索できる

```
select * from employeeTable, departmentTable;
```

![image-20230719194320593](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230719194320593.png)

```
#関する変数を検査してtureが設置された
#若しデカルト積を支持しよう、falseに変更していい
set hive.strict.checks.cartesian.product=false;
```

![image-20230719195105939](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230719195105939.png)

　　一時に56条データが検査されて出て、大部分のは必要ない。サービスの性能も占めすぎ、当の変数がfalseを保ってお勧めです。言わば、ある錯誤をするために過量データが発生されたのが避けて保護仕組みの存在です。

### 第５項　並べ替え語句

**全局並べ替え(order by)**

　　order byは最終の結果に並べ替えを進行し、テーブルデータ本体ではない。一般的に全体の検索語句の最後に置く。黙認状況の下に昇順(ASC)、必要なら、カラムの後ろにDESCを付いて降順が表す。

```
#普通の並べ替え
select * from employeeTable order by department;

#多カラム場合
select * from employeeTable order by department, sal desc;

#別名使う
select name, job, sal salcomm, department from employeeTable order by salcomm desc;
```

　　並べ替えカラムが必ず検査の結果に存在し、以下の語句が扱えない、select語句にdepartmentカラムが欠かす。

```
select name, job, sal from employeeTable order by department;
```

![image-20230721153520795](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230721153520795.png)

**MR任務内部並べ替え(sort by)**

　　SORT BYを使ってデータの並べ替えがMR任務処理する前にも完成しておく。具体的な並べ替え方式はカラム類型に決まって、数字なら数字の順序で並べ替え、文字型なら英語字母の順序で並べ替える。

```
#orderTable作成
CREATE TABLE orderTable (
id int,
name string,
job string,
department int,
hiredate DATE,
sal int
) 
partitioned by (jobId int)
row format delimited fields terminated by ",";

#データ挿入
insert into table orderTable partition(jobId=1) select * from employeeTable where job='CLERK';
insert into table orderTable partition(jobId=2) select * from employeeTable where job='SALESMAN';
insert into table orderTable partition(jobId=3) select * from employeeTable where job='MANAGER';
insert into table orderTable partition(jobId=4) select * from employeeTable where job='ANALYST';
```

![image-20230723162657224](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230723162657224.png)

```
#order by
select * from orderTable order by id;
```

![image-20230723162805265](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230723162805265.png)

```
#reduceの個数を設置
set mapreduce.job.reduces=2;

# sort by
select * from orderTable sort by id;
```

![image-20230723164527199](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230723164527199.png)

　　ここ見えてorder byと検査結果が違う。今結果データを本地サービスに輸出して見る。

```
insert overwrite local directory '/usr/hadoop/data/orderTable' select * from orderTable sort by id;
```

![image-20230723171134859](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230723171134859.png)

　　輸出ファイルが二つに分けて、毎部分にIdの昇順で並べ替える。その効果の前提はreduceの個数が複数です（一つreduceなら、結果が同じ）。それはなぜMR任務内部の並べ替えと言われるということです。簡単にorder byとsort byの区別は前者が全てのデータについて並べ替え、後者がただ単数のreduce内部にの並べ替えが保証していい。

**桶割り（distribute by）**

　　distribute byには並べ替えの機能がありません、具体的な執行効果が桶テーブル（clustered by）に似て、区別は執行の流れが違う。distribute byが検査語句を執行する時に発効して、clustered byはテーブルの作成した時に使うことがあり、導入のデータに一定的な規則で桶割りを扱う。

```
#reduceの個数を設置
set mapreduce.job.reduces=3;

#直接に本地サービスに輸出
insert overwrite local directory '/usr/hadoop/data/distribute' select * from orderTable distribute by id;
```

![image-20230724171643038](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230724171643038.png)

![image-20230724171758873](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230724171758873.png)

　　ただdistribute byを使って並べ替えのことはありません、sort byと組み合わせて毎reduceのデータを並べ替えられる。

```
#最後にsort byを追加し、毎reduceに部門の降順で並べ替える
insert overwrite local directory '/usr/hadoop/data/distribute' select * from orderTable distribute by id sort by department desc;
```

![image-20230724183935062](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230724183935062.png)

**桶割りの並べ替え（cluster by）**

　　distribute byとsort byの修飾されるカラムが一つなら、cluster by略書き方が使える。でも、cluster byは並べ替えの順序が指定されない。

```
#分けて書き方とは等価です
insert overwrite local directory '/usr/hadoop/data/distribute' select * from orderTable cluster by id;
```

![image-20230724190937719](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230724190937719.png)

