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

　：検査結果から見えてHBase格納方式と伝統のデータベース差別が大きい、key-value形式データを一つ一つで積んでくれ、そのお陰様で空値に対して空間を分配してあげることが必要ない、大幅にストレージ空間の使用率が上がる。あと、同じバケーションデータが一つの区域（cell）に属して、更にデータを細分して検索を効率的にする。

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

#### 基本概念

　　データ検索を紹介する前に幾つかHBase shell概念を説明し、主に演算子やキーワードの使いことに関わり、詳しい内容が公式サイト([Apache HBase ™ Reference Guide](https://hbase.apache.org/book.html#thrift.filter_language))から了解できる。

**二項演算子**

- AND：AND両方の濾過条件を満たしたデータを表す。

- OR：OR両方少なくとも1つの濾過条件を満たしたデータを表す。

**単項演算子**

- SKIP：特定の行を除いて、他の濾過条件を満たしたデータを表す。

**比較演算子**

- <、<=、=、!=、>、>=比較演算子が使える。

**比較器**

- BinaryComparator - binary：データに完全一致に比較したデータを篩い分ける。

- BinaryPrefixComparator - binaryprefix：データの接頭辞を比較した一致のデータを篩い分ける。
- SubStringComparator - substring：曖昧検索する場合に使い、比較値が含まれるデータを篩い分ける。
- RegexStringComparator - regexstring：正規表現を使って比較を行う。

**濾過器**

- RowFilter：RowKeyについて、該当のRowKeyに対して行を返す。
- ValueFilter：RowKeyを除いてkey-valueのvalueにおけて、該当の値や対応のRowKeyを一緒に返す。列の範囲を指定できる。
- SingleColumnValueFilter：ValueFilterに似て、該当の行を全て返す。
- QualifierFilter：列名について、修飾された列の下にある全ての値を返す。
- PrefixFilter：RowKeyについて、接頭語の比較し、該当の行を返す。

```
scan 'table_name', {COLUMNS => 'column_family:column_name', FILTER => "FilterName (argument, argument,... , 'argument')"}
```

　　大体の検索命令の書き方が上記の様子、不同の検索条件によって「FilterName(argument)」書き方の差別があり、具体の使い方が次に例を挙げて説明する。ところで、命令の最外層の括弧{}を省けることができるが、本節まだ公式に沿って書くつもりです。

　　後は、濾過器は濾過の範囲や返すデータの形を決め、具体の濾過条件を比較器で決める。例えば、「RowFilter(=,'substring:k3')」中に「substring:k3」の意味は「k3」で曖昧検索し、濾過器「RowFilter」を使ったと検索の範囲をRowKey列に限り、且つ、条件を満たした値に対応の全ての行を取得して表す。

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

**複数の行**

```
# 全体のデータを取得
scan 'studentInfo'

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
scan 'studentInfo',{TIMERANGE => [1703630815262,1703630815431]}
```

![image-20231229113000993](D:\OneDrive\picture\Typora\image-20231229113000993.png)

**完成一致検索**

```
# 大文字と小文字が区別することある
scan 'studentInfo', {COLUMNS => 'base_info:name', FILTER => "ValueFilter(=,'binary:Jane Smith')"}

# 列を指定しなくても検索でき、ただ検索の対象はRowKeyが含まれない
scan 'studentInfo', {FILTER => "ValueFilter(=,'binary:Jane Smith')"}

# 指定された値を除いてデータを表す
scan 'studentInfo', {FILTER => "ValueFilter(!=,'binary:Jane Smith')"}
```

![image-20240104101922526](D:\OneDrive\picture\Typora\image-20240104101922526.png)

```
# 返却列を指定されない場合に適当の値に対応の行を表す
scan 'studentInfo', {FILTER => "SingleColumnValueFilter('base_info', 'name', =,'binary:Jane Smith')"}
# つまり、キーワードの間に一定の優先順位がある
scan 'studentInfo', {COLUMNS => 'base_info:name', FILTER => "SingleColumnValueFilter('base_info', 'name', =,'binary:Jane Smith')"}
```

![image-20240115153231372](D:\OneDrive\picture\Typora\image-20240115153231372.png)

```
# Rowkeyについての一致検索
scan 'studentInfo',{FILTER=>"RowFilter(=,'binary:rk3')"}
```

![image-20240108143113288](D:\OneDrive\picture\Typora\image-20240108143113288.png)

**曖昧検索**

　　比較器「substring」に差し替えて曖昧検索の働きを果たすのもです。

```
# 適当のフィールド値のみを表す
scan 'studentInfo', {COLUMNS => 'base_info:name', FILTER => "ValueFilter(=,'substring:a')"}

# 曖昧検索に「!=」も使える
scan 'studentInfo', {COLUMNS => 'base_info:name', FILTER => "ValueFilter(!=,'substring:a')"}
```

![image-20240103111905946](D:\OneDrive\picture\Typora\image-20240103111905946.png)

```
# 整体の行データを返却
scan 'studentInfo', {FILTER => "SingleColumnValueFilter('base_info','name',=,'substring:a')"}
```

![image-20240115163007146](D:\OneDrive\picture\Typora\image-20240115163007146.png)

```
# RowKeyにの曖昧検索
scan 'studentInfo',{FILTER=>"RowFilter(=,'substring:k3')"}
```

![image-20240115165613974](D:\OneDrive\picture\Typora\image-20240115165613974.png)

　　PrefixFilterはRowKeyのみについて接頭語を比較して特別の濾過器です、比較器substringなどを添えない、濾過や比較の働きを兼ねる。このAPIを提供されたことから見えて、事実の業務によってRowKeyの形を設計しておき、検索を効率的にするのができる。

```
scan 'studentInfo',{FILTER=>"PrefixFilter('rk3')"}
```

![image-20240115170919128](D:\OneDrive\picture\Typora\image-20240115170919128.png)

**単一の行**

　　RowKeyを指定して単一の行を検査し出すとget命令を使える。Get命令は、一つの行のデータを取得するために一種の特殊なScan操作と考えることができ、特定の行を検索する必要がある場合、Get命令を使用するとScan命令よりも効率的です。

　　Scan命令は、複数の行をスキャンして表すためのAPIです。これは一つ又は複数の範囲から複数の行のデータを取得し、データの濾過と並び替えに役に立つ。

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

　　フィールド名が存在しなくてもエラーメッセージ、空値など返すのがない、何も表れないんです。あと、HBaseのストレージ結構はkey-vlue形式で格納し、一行の複数の列に同時に完全一致検索や曖昧検索も行える。

```
get 'studentInfo', 'rk2', {FILTER => "ValueFilter(=,'substring:a')"}
```

![image-20240115211127378](D:\OneDrive\picture\Typora\image-20240115211127378.png)

```
get 'studentInfo', 'rk2', {FILTER => "SingleColumnValueFilter('base_info','name', =,'substring:a')"}
```

![image-20240116150838135](D:\OneDrive\picture\Typora\image-20240116150838135.png)

**数字の比較**

```
scan 'studentInfo',{FILTER=>"SingleColumnValueFilter('extra_info','math_grade', <,'binary:90')"}

# 絶対大文字で書き、そしないとエラーが発生になる。
scan 'studentInfo',{FILTER=>"(SingleColumnValueFilter('extra_info','math_grade', <,'binary:80')) AND (SingleColumnValueFilter('extra_info','math_grade', >,'binary:50'))"}
```

![image-20240116221023445](D:\OneDrive\picture\Typora\image-20240116221023445.png)

```
scan 'studentInfo',{COLUMNS => 'extra_info:math_grade', FILTER=>"ValueFilter(<,'binary:90')"}
scan 'studentInfo',{FILTER=>"ValueFilter(<,'binary:78')"}
```

![image-20240116162226149](D:\OneDrive\picture\Typora\image-20240116162226149.png)

**列名について検索**

　　HBaseに列名について検索することができ、一般のデータベースがこの特性がない。ただ、指定されたフィールド値を表れる必要だと「COLUMNS => 'base_info:name'」を添えるなら、列名「name」が比較条件「binary:math_grade」の濾過結果に合わないため、結局何も返却しない。

```
scan 'studentInfo', {FILTER => "QualifierFilter(=,'binary:math_grade')"}
scan 'studentInfo', {FILTER => "QualifierFilter(=,'substring:grad')"}
```

![image-20240116153541144](D:\OneDrive\picture\Typora\image-20240116153541144.png)

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
