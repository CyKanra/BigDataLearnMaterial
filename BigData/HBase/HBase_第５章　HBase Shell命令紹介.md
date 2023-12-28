# HBase-分散型大規模非関係データベース-5

## 第５章　HBase Shell命令紹介

Hbaseクライアント画面に入って、クラスター中に任意の節点を選べられる。

```
hbase shell
```

![image-20231222074643934](D:\OneDrive\picture\Typora\image-20231222074643934.png)

**ヘルプファイル**

```
help
```

**テーブル検査**

```
list
```

**テーブル作成**

```
create 'studentInfo', 'base_info', 'extra_info'
# 又は
# VERSIONSイコール3意味は最近三つのバケーションデータを保留
create 'studentInfo', {NAME => 'base_info', VERSIONS => '3'},{NAME => 'extra_info',VERSIONS => '3'}
```

![image-20231225065123273](D:\OneDrive\picture\Typora\image-20231225065123273.png)

**データ操作**

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

**データ検索**

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

　　フィールド名が違ってもエラーメッセージ、空値など返してのがない、何も表れないだけです。

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

```
# 全体のデータを取得
scan 'studentInfo'
```

　　Get命令は、一つの行のデータを取得するためのもので、一種の特殊なScan操作と考えることができる。ただし、Getは一つの行のデータのみを必ずRowKey変数をついて取得し、Scanは複数の行のデータを取得する。特定の行を検索する必要がある場合、Get命令を使用するとScan命令よりも効率的です。

　　Scan命令は、複数の行をスキャンするためのAPIです。これは一つ又は複数の範囲から複数の行のデータを取得し、データの濾過と並び替えにするために使用できる。







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
