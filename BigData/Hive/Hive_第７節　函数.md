# Hiveデータ倉庫工具ー7

## 第７節　関数

　　Hiveの関数を使うとSQLのが類似です。当の節は常に使用する函数を選べて紹介する。もっと了解して欲しいなら、最後に公式文章のリンクが提供されたおき、公式サービスに全ての関数を詳しく紹介してある。

公式文献：[LanguageManual UDF - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+UDF#LanguageManualUDF-Built-inFunctions)

### 第１項　システム内部関数

**システム関数検査**

```
#システム自分の関数を検査
show functions;

#単独の関数の使い方を検査（重要）
desc function upper;
desc function extended upper;
```

**期日関数**

```
#今の期日又は時刻
select current_date;
select unix_timestamp();
select current_timestamp();

#タイムスタンプを期日に転換
select from_unixtime(1690324419);
select from_unixtime(1690324419, 'yyyyMMdd');
select from_unixtime(1690324419, 'yyyy-MM-dd HH:mm:ss');

#期日をタイムスタンプに転換
select unix_timestamp('2023-07-26 14:23:00');

#期日差分
select datediff('2020-04-18','2023-07-26');
select datediff('2023-07-26', '2020-04-18');

#期日は本月に何日
select dayofmonth(current_date);

#月末の期日:
select last_day(current_date);

#当月の第１日:
select date_sub(current_date, dayofmonth(current_date)-1);

#来月の第２日:
select add_months(date_sub(current_date, dayofmonth(current_date)-1), 1);

#文字型を時刻に転換（格式：yyyy-MM-dd格式）
select to_date('2023-07-26');
select to_date('2023-07-26 12:12:12');

#期日、タイムスタンプ、文字型格式を標準の時間格式に転換
select date_format(current_timestamp(), 'yyyy-MM-dd HH:mm:ss');
select date_format(current_date(), 'yyyyMMdd');
select date_format('2023-07-26', 'yyyy-MM-dd HH:mm:ss');
```

**文字列関数**

```
#小文字に転換　lower
select lower(job), job from employeetable;

#大文字に転換　upper
select lower(name), name from employeetable;

#文字列の長さ　length
select length(name), name from employeetable;

#文字列の合弁　concat / ||
select name || " " ||job nameJob from employeetable;
select concat(name, " " ,job) nameJob from employeetable;

#分離符が指定できる concat_ws(separator, [string | array(string)]+)
SELECT concat_ws('.', 'www', array('bing', 'com'));
SELECT concat_ws("+++", name, job) from employeetable;

#文字列の抽出、符号位置が含まれる　substr
SELECT substr('www.bing.com', 4);
SELECT substr('www.bing.com', -4);
SELECT substr('www.bing.com', 4, 4);

#文字列の分割、’.' 前にエスケープ文字を追加のが必要です
select split("www.lagou.com", "\\.");
```

**数字関数**

```
#四捨五入　round
select round(314.15926);
select round(314.15926, 2);
select round(314.15926, -2);

#上へ整数を取り　ceil
select ceil(3.1415926);

#下へ整数を取り　floor
select floor(3.1415926);
```

**集計関数**

```
#総和
select sum(sal) from employeetable;

#平均値
select department, avg(sal) from employeetable group by department;

#データ数
select department, count(id) from employeetable group by department;

#最大値と最小値
select department, max(sal) from employeetable group by department;
select mix(sal) from employeetable;
```

　　集計関数の使い方がMysqlのSQLに似るんですが、事実は中々違うんです。下の図は公式文章に集計函数の説明の最後に「values of the column in the group」と述べられた。簡単に解釈してきて、実際の扱いには他の輸出されるカラムが必ず並べ替え語句に修飾されたのカラムです。SQLみたい任意のカラム一緒に輸出されることはできない。

![image-20230730213436883](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230730213436883.png)

　　余りのカラムが並べ替えの副問合せにない場合は実行が通ってない。

```
select name, min(sal) from employeetable group by department;
select name, min(sal) from employeetable;
```

![image-20230731085455625](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230731085455625.png)

　　下記の副問合せを添加して書き方が結果も得られない。Hiveは＝、!=など後ろに副問合せを追加して埋め込みの方法が使えない。

```
select a.* from employeetable a where a.sal=(select min(b.sal) from employeetable b);
```

![image-20230731114434629](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230731114434629.png)

　　このような結果が得られる。ここから感じられて、複雑の検索語句を書くはHive全ての勉強に難点です。

```
select * from employeetable order by sal asc limit 1;
```

**条件関数**

```
#if (boolean 条件, T valueTrue, T valueFalse Or Null)
select sal, if (sal<300000, 3, if (sal>350000, 1, 2)) sallevel from employeetable;

#CASE WHEN 条件１ THEN b [WHEN 条件２ THEN d]* [ELSE e] END
select sal, case when sal<300000 then 3 when sal>350000 then 1 else 2 end sallevel from employeetable;

#一定の場合、下記のような書き方も使える
select department, case department when 20 then 1 when 40 then 2 else 3 end departmentNum from employeetable;
```

### 第２項　UDTF関数

　　UDFT：User Defined Table-Generating Functions。テーブル生成の関数、専門的に一行のArray又はMap型カラムに対して関数を使って多行データに輸出できる。

**explode**

```
#１行データ中にArray又はMapような複雑の結構を多行データに分割する
select explode(array('A','B','C')) as col;
select explode(map('a', 8, 'b', 88, 'c', 888));
```

![image-20230726211147239](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230726211147239.png)

　　一行データにArray型カラム又はMap型カラム中のデータを抽出して、多行に輸出する同時に他のカラムが保留して欲しいなら、ただexplode関数を使って満足できない。

```
select 1, explode(array('A','B','C'));
```

![image-20230727214427524](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230727214427524.png)

**lateral View**

```
LATERAL VIEW udtf(expression) tableAlias AS columnAlias (',' columnAlias)*
fromClause: FROM baseTable (lateralView)*
```

　　上記の語法を通訳して「LATERAL VIEW explode(Array&Mapカラム)  テーブル別名 as カラム別名」とういう意味。

```
#Array使い方
with t1 as (
 select 'hello' colA, split('www.bing.com', '\\.') colB
)
select colA, colC from t1 lateral view explode(colB) t2 as colC;
```

![image-20230727164339410](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230727164339410.png)

```
#Map使い方
with t1 as (
 select 'hello' colA, map('a', 1, 'b', 2, 'c', 3) colB
)
select colA, colC, colD from t1 lateral view explode(colB) t2 as colC, colD;
```

![image-20230728055807716](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230728055807716.png)

　　若しArray又はMapのデータがありません（array()、map()）、輸出結果が何もない。でも、依然結果があってほしいなら、OUTERキーワードを付いて欠けたカラムをnullで補う。因みに、Array()とArray(null)の輸出結果が違う

```
with t1 as (
 select 'hello' colA, map() colB
)
select colA, colC, colD from t1 lateral view explode(colB) t2 as colC, colD;

with t1 as (
 select 'hello' colA, map() colB
)
select colA, colC, colD from t1 lateral view outer explode(colB) t2 as colC, colD;
```

![image-20230728070032493](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230728070032493.png)

### 第３項　窓口関数

　　窓口関数（Windowing and Analytics Functions）は複雑の統計に関する計算に対して重要の存在、色々な場合に使用してある。でも、集計函数所紹介された、単独の函数にHive Sqlの使いが不便、何と言ってもHiveはデータベースじゃない、MySQLよりそんな強い機能がない。だから、ここでoverキーワードを引用してきて、関数と組み合わせて使用する。

公式文献：[LanguageManual WindowingAndAnalytics - Apache Hive - Apache Software Foundation](https://cwiki.apache.org/confluence/display/Hive/LanguageManual+WindowingAndAnalytics)

**窓口関数over()**

　　常にCOUNT、SUM、MIN、MAX、AVG一緒に使う。

```
#毎職員の給料が総額を占める百分比のを計算
#給料の総額
select sum(sal) from employeetable;

#下の書き方が語法の錯誤あり、組み分け語句が欠かす。
select ename, sal, sum(sal) salsum from employeetable;

#単純に一つのselect文を使って百分比の計算を実現するつもりなら、over()が役に立てる
select name, sal, sum(sal) over() salSum, concat(round(sal/sum(sal) over()*100, 1) || '%') ratiosal from employeetable;
```

![image-20230728120237822](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230728120237822.png)

　　over()の作用はsum()に当のテーブルデータを提供してあげて、前の計算関数がover()からデータを取得して計算を進行し、fromの後ろの部分ではない。over()の具体的な返却値が括弧の条件に決まった、その範囲も窓口の大小として知られる。例えば、over(partition by department)、 over(order by sal)など。

**partition byと結合**

```
#職員の名称、給料、部門の給料総和
select name, department, sal, sum(sal) over(partition by department) salSum from employeetable;
```

![image-20230801194224360](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230801194224360.png)

**order byと結合**

```
#order by追加
select name, department, sal, sum(sal) over(partition by department order by sal) salsum from employeetable;
```

![image-20230801194258767](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230801194258767.png)

　　ここのsum(sal)が一行目から当の行目にかけての総和です。下の書き方の結果と同じ、黙認状況にbetween ... andが欠かしてorder by後ろに「rows between unbounded preceding and current row」条件を追加する。

```
select name, department, sal, sum(sal) over(partition by department order by sal rows between unbounded preceding and current row) salsum from employeetable;
```

**between ... and ...と結合**

```
(ROWS | RANGE) BETWEEN (UNBOUNDED | [num]) PRECEDING AND ([num] PRECEDING | CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
(ROWS | RANGE) BETWEEN CURRENT ROW AND (CURRENT ROW | (UNBOUNDED | [num]) FOLLOWING)
(ROWS | RANGE) BETWEEN [num] FOLLOWING AND (UNBOUNDED | [num]) FOLLOWING
```

- unbounded preceding：組内の最初の行のデータ
- n preceding：組内の現在の行の前にあるN行のデータ
- current row：現在の行のデータ
- n following：組内の現在の行の後ろにあるN行のデータ
- unbounded following：組内の最後の行のデータ

```
#一行目から当の行まで総和
select name, sal, department, sum(sal) over(partition by department order by name rows between unbounded preceding and current row) from employeetable;

#前の一行、当の行、次の行の総和
select name, sal, department, sum(sal) over(partition by department order by name rows between 1 preceding and 1 following) from employeetable;
```

### 第４項　分析関数

　　分析函数（Analytics Functions）は複雑の統計によく使用する。

```
#データを準備
1001,Jervis,Roll,Director of Sales,Sales,30000
1002,Gordon,Mattster,Marketing Manager,Sales,25000
1003,Gracie,Fronllen,Assistant Manager,Sales,25000
1004,Joelly,Wellback,Account Coordinator,Account,15000
1005,Bob,Havock,Accountant II,Account,20000
1006,Carmiae,Courage,Account Coordinator,Account,15000
1007,Cellie,Trevaskiss,Assistant Manager,Sales,25000
1008,Gally,Johnson,Manager,Account,28000
1009,Richard,Grill,Account Coordinator,Account,12000
1010,Sofia,Ketty,Sales Coordinator,Sales,20000

#テーブル作成
CREATE TABLE analyticsTable (
ID int,
FIRST_NAME string,
LAST_NAME string,
DESIGNATION string,
DEPARTMENT string,
SALARY int
)row format delimited fields terminated by ",";

#データ輸入
LOAD DATA LOCAL INPATH '/usr/hadoop/data/analyticsTable.dat' INTO TABLE analyticsTable;
```

![image-20230804195246744](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230804195246744.png)

**順位関数**

- ROW_NUMBER()：一組内に各行を一意の値を分配される。
- RANK()：順位を割り当て、同じ値を持つ行は同じ順位が割り当てられ、次の順位が抜かされる。
- DENSE_RANK()：RANK()函数に反して、同じ値が出現するなら、次の順位が保留される。over()から割り当てられるデータの範囲内に順位が連続だ。

```
SELECT department as dept, salary as sal, 
ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) as row_num,
RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rnk,
DENSE_RANK() OVER(PARTITION BY department ORDER BY salary DESC) as dns_rnk 
FROM analyticsTable;
```

![image-20230804195327842](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230804195327842.png)

- CUME_DIST()：この関数は累積分布を表す。グループ内の列の値の相対的な位置を計算します。ここでは、すべての仕切り間で累積分布を計算できる。

　　累積分布値= 指定されたフィールドの値で以下または等しい値を持つ行の総数 / 結果の総行数

```
SELECT id, department, salary, CUME_DIST() OVER (ORDER BY salary) as cum_dist FROM analyticsTable;
```

![image-20230815143925349](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230815143925349.png)

　　IDは1002を例として、25000に等しい又は以下の行数が８、総数10を除するとして0.8結果が得る。

- PERCENT_RANK()：これはCUME_DIST関数と類似している。でも、結果を百分比の順序で表れ、及びすべでのデータに対するではない、仕切り範囲内に計算される。最初の行は百分比0を持ち、戻り値は倍精度浮動小数点型です。

　　百分比 = (当の行数 - 1) / (当の行目含まれて残りの行数)

```
SELECT department, salary, RANK() OVER (PARTITION BY department ORDER BY salary DESC) as rnk, PERCENT_RANK() OVER (PARTITION BY department ORDER BY salary DESC) as perc_rnk FROM analyticsTable;
```

![image-20230815153158162](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230815153158162.png)

　　部門Accountを例として、給与2000以上の行数が1、ならば部門Accountに残り行数が４、両者除すってもらって0.25結果が得る。

- NTILE()：関数には特定の数を付けされた、仕切り内の行数をできるだけ均等に分割する。

```
SELECT id, department, salary, 
NTILE(3) OVER (PARTITION BY department ORDER BY salary DESC) as ntile3, 
NTILE(4) OVER (PARTITION BY department ORDER BY salary DESC) as ntile4, 
NTILE(6) OVER (PARTITION BY department ORDER BY salary DESC) as ntile6 
FROM analyticsTable;
```

![image-20230816152433177](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230816152433177.png)

　　明確な数式ない、でも検索結果によって内部にどんな流れで得ることは推測できる。仮定にNTILE()に入力の値がN、仕切り行数がCです。若しN>=C、ntile6結果みたい毎の行が唯一値が’得られる。若しN<C、例えばN=3場合、5%3=2、余りの2つは順調で1からNまでで全部分配されておく。つまり、1と2が余り1行が分配された、他の行よりもっと１がある。ntile3の結果[1,1,2,2,3,4]通り、1から3まで2、2、1ように5行データを分割された。
