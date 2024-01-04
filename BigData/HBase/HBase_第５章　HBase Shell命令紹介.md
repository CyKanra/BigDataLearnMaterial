# HBase-分散型大規模非関係データベース-5

## 第５章　HBase Shell命令紹介

　　関係データベースみたいの実行語句には、HBaseデータベースでこの語句がShell命令と呼ばれ、検索語句ではない。その設定の上に、複合の語句で検索することが支持しない、Jasonみたい書き方も理解しにくいなど特徴がある。

```
# 任意の節点に登録でき
hbase shell
# ヘルプ
help
# テーブル検査
list
```

![image-20231222074643934](D:\OneDrive\picture\Typora\image-20231222074643934.png)

#### テーブル作成

```
create 'studentInfo', 'base_info', 'extra_info'

# VERSIONSイコール3意味は最近三つのバケーションデータを保留
create 'studentInfo', {NAME => 'base_info', VERSIONS => '3'},{NAME => 'extra_info',VERSIONS => '3'}
```

![image-20231225065123273](D:\OneDrive\picture\Typora\image-20231225065123273.png)

#### データ操作

　　HBaseは非関係データベースですが、その内に庫の概念がない。データの操作を実行する前にどっちの庫で指定することが必要ない。あと、データを挿入するのは一つ一つのフィールド値で添加するだけ、HBaseがテーブル概念があるけれど。

```
# 添加
put 'studentInfo', 'rk1', 'base_info:name', 'John Doe'
# 複数のフィールド値を共に入力するのが駄目です
put 'studentInfo', 'rk1', 'base_info:name', 'John Doe', 'base_info:sex', '1'

# フィールド添加、一つ一つのフィールドでデータ添加だけ
put 'studentInfo', 'rk1', 'base_info:sex', '1'
put 'studentInfo', 'rk1', 'extra_info:math_grade', '85'
put 'studentInfo', 'rk1', 'extra_info:history_grade', '75'

# データ検査
get 'studentInfo', 'rk1'
```

![image-20231225072149913](D:\OneDrive\picture\Typora\image-20231225072149913.png)

　　検査結果から見えてHBase格納方式と伝統のデータベース差別が大きい、key-value形式データを一つ一つで積んでくれ、そのお陰様で空値に対して空間を分配してあげることが必要ない、大幅にストレージ空間の使用率が上がる。あと、同じバケーションデータが一つの区域（cell）に属して、更にデータを細分して検索を効率的にする。

```
# データ改修
put 'studentInfo', 'rk1', 'extra_info:history_grade', '94'
```

![image-20231225072647526](D:\OneDrive\picture\Typora\image-20231225072647526.png)

　　データ改修と添加の語句が同じ、存在したデータあれば上書きしてくれ、存在しないデータが添加になる。

```
# データ消除
delete 'lagou', 'rk1', 'base_info:age'

# テーブルデータ全部消除
truncate 'lagou'
```

#### データ検索

```
# データ添加
put 'studentInfo', 'rk1', 'base_info:name', 'John Doe'
put 'studentInfo', 'rk1', 'base_info:sex', '1'
put 'studentInfo', 'rk1', 'base_info:birthday', '1990/05/15'
put 'studentInfo', 'rk1', 'extra_info:math_grade', '85'
put 'studentInfo', 'rk1', 'extra_info:history_grade', '78'

put 'studentInfo', 'rk2', 'base_info:name', 'Jane Smith'
put 'studentInfo', 'rk2', 'base_info:sex', '0'
put 'studentInfo', 'rk2', 'base_info:birthday', '1988/11/22'
put 'studentInfo', 'rk2', 'extra_info:math_grade', '92'
put 'studentInfo', 'rk2', 'extra_info:history_grade', '88'

put 'studentInfo', 'rk3', 'base_info:name', 'Mike Johnson'
put 'studentInfo', 'rk3', 'base_info:sex', '1'
put 'studentInfo', 'rk3', 'base_info:birthday', '1995/07/10'
put 'studentInfo', 'rk3', 'extra_info:math_grade', '78'
put 'studentInfo', 'rk3', 'extra_info:history_grade', '65'

put 'studentInfo', 'rk4', 'base_info:name', 'Emily Davis'
put 'studentInfo', 'rk4', 'base_info:sex', '0'
put 'studentInfo', 'rk4', 'base_info:birthday', '1992/03/03'
put 'studentInfo', 'rk4', 'extra_info:math_grade', '95'
put 'studentInfo', 'rk4', 'extra_info:history_grade', '91'

put 'studentInfo', 'rk5', 'base_info:name', 'Chris Brown'
put 'studentInfo', 'rk5', 'base_info:sex', '1'
put 'studentInfo', 'rk5', 'base_info:birthday', '1987/09/28'
put 'studentInfo', 'rk5', 'extra_info:math_grade', '89'
put 'studentInfo', 'rk5', 'extra_info:history_grade', '75'
```

**単一の行**

　　RowKeyを指定して単一の行を検査し出すとget命令を使う。

```
# rowkeyイコールrk1データを検索
get 'studentInfo', 'rk1'

# rk1下に指定された列族を検索
get 'studentInfo', 'rk1', 'base_info'
# 複数の列族を指定
get 'studentInfo', 'rk1', 'base_info', 'extra_info'
get 'studentInfo', 'rk1', {COLUMN => ['base_info', 'extra_info']}

# 指定されたフィールドを検索
get 'studentInfo', 'rk1', 'base_info:name', 'base_info:sex'
get 'studentInfo', 'rk1', {COLUMN => ['base_info:name', 'extra_info:math_grade']}
```

![image-20231227112034965](D:\OneDrive\picture\Typora\image-20231227112034965.png)

　　フィールド名が違ってもエラーメッセージ、空値。ど返してのがない、何も表れないだけです。

**複数の行**

　　Get命令は、一つの行のデータを取得するためのもので、一種の特殊なScan操作と考えることができる。ただし、Getは一つの行のデータのみを必ずRowKey変数をついて取得し、Scanは複数の行のデータを取得する。特定の行を検索する必要がある場合、Get命令を使用するとScan命令よりも効率的です。

　　Scan命令は、複数の行をスキャンして表すためのAPIです。これは一つ又は複数の範囲から複数の行のデータを取得し、データの濾過と並び替えに役に立つ。

```
# 全体のデータを取得
scan 'studentInfo'
```

**列族やフィールド**

```
# base_info列族検査
scan 'studentInfo', {COLUMNS => 'base_info'}
scan 'studentInfo', {COLUMNS => ['base_info']}
# 具体的なフィールド値検査
# 複数の列族又はフィールドの検索は「,」で分割して[]に包まれる、
scan 'studentInfo', {COLUMNS => ['base_info', 'extra_info']}
scan 'studentInfo', {COLUMNS => ['base_info:name', 'extra_info:history_grade']}

# VERSIONSバケーション番号を指定でき、例えば、全て保留できるバケーション数が3、「VERSIONS => 3」が最大のバケーション値を表す
scan 'studentInfo', {COLUMNS => ['base_info:name', 'extra_info:history_grade'],  VERSIONS => 1}
```

![image-20231229101418006](D:\OneDrive\picture\Typora\image-20231229101418006.png)

**RowKey範囲**

　　rowkeyについて一定範囲内の検査、STOPROWに対応のrowkeyが含まれない。

```
scan 'studentInfo', {STARTROW=>'rk2',STOPROW=>'rk5'}
scan 'studentInfo', {COLUMNS => 'base_info', STARTROW => 'rk1', ENDROW => 'rk3'}
```

![image-20231229183417983](D:\OneDrive\picture\Typora\image-20231229183417983.png)

**時間範囲**

　　時間範囲の内にデータを検査し、範囲は前値t1以上かつ後値t2未満と「t1<=time<t2」ようです。ここの時間が実際のデータを挿入する時刻、バケーションではない。

```
# 1703630815431に等しいデータが表れない
# []の意味が列族又はフィールドの検索と違う
scan 'studentInfo',{TIMERANGE=>[1703630815262,1703630815431]}
```

![image-20231229113000993](D:\OneDrive\picture\Typora\image-20231229113000993.png)

**完成一致検索**

```
# 大文字と小文字が区別することある
scan 'studentInfo', {COLUMNS => 'base_info:name', FILTER => "(ValueFilter(=,'binary:Jane Smith'))"}

# 列を指定しなくても検索でき、ただRowKeyが含まれない
scan 'studentInfo', {FILTER => "(ValueFilter(=,'binary:Jane Smith'))"}
```

![image-20240104101922526](D:\OneDrive\picture\Typora\image-20240104101922526.png)

```
# 適当なデータが全てのフィールド値を表す
scan 'studentInfo', {FILTER => "(SingleColumnValueFilter('base_info', 'name', =,'binary:Jane Smith'))"}
```

![image-20240104115350527](D:\OneDrive\picture\Typora\image-20240104115350527.png)

**曖昧検索**

```
scan 'table_name', {COLUMNS => 'column_family:column_name', FILTER => "filter_string"}
```

　　大体の格式が上記の様です。ただ、不同の検索条件によって「filter_string」書き方の差別が大きい、ここで例を挙げて説明しよう。

```
scan 'studentInfo', {COLUMNS => 'base_info:name', FILTER => "(ValueFilter(=,'substring:a'))"}
```

![image-20240103111905946](D:\OneDrive\picture\Typora\image-20240103111905946.png)



以下是一些常用的 HBase shell 命令：

- `create <table>, {NAME => <family>, VERSIONS => <VERSIONS>}`：创建表。
- `list`：列出所有已创建的表。
- `describe <table>`：显示表相关的详细信息。
- `put <table>,<rowkey>,<family:column>,<value>`：向指定表单元添加值。
- `get <table>,<rowkey>`：获取行的值。
- `delete <table>,<rowkey>,<family:column>`：删除指定对象的值。
- `deleteall <table>,<rowkey>`：删除指定行的所有元素值。
- `scan <table>`：通过对表的扫描来获取对应的值。
- `count <table>`：统计表中行的数量。
- `disable <table>`：使表无效。
- `enable <table>`：使表有效。
- `drop <table>`：删除表。
- `exit`：退出 HBase shell。
