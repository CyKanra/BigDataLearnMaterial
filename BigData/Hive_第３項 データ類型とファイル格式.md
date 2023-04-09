# Hiveデータ倉庫工具ー3

## 第３項　データ類型とファイル格式

　　Hiveは関係性データベースに似て大多数基本的なデータ類型が支持し、また４つの集合データ類型も支持できる。

### 第１節　基本的なデータ類型の変換

　　HiveはJavaに似て、様々な整数型と浮動小数点型が支持し、また論理型、文字列型及び日付時刻型等も支持できる。具体的な分類が以下の表に表示される。

| 類型                                       | 具体的な類型                                                 |
| ------------------------------------------ | ------------------------------------------------------------ |
| Integers(整数型)                           | TINYINT -- 1バイトの有符号整数型<br />SMALLINT -- 2バイトの有符号整数型<br />INT -- 4バイトの有符号整数型<br />BIGINT -- 8バイトの有符号整数型 |
| Floating point numbers<br />(浮動小数点型) | FLOAT -- 単精度数点型<br />DOUBLE -- 双精度数点型            |
| Fixed point numbers<br />(固定小数点型)    | DECIMAL -- 17バイト、任意の精度数字                          |
| String(文字列型)                           | STRING -- 可指定、文字集合、不固定長文字列型<br />VARCHAR -- 1-65535不固定長文字列型<br />CHAR -- 1-255固定長文字列型 |
| Datetime(日付時刻型)                       | STRTIMESTAMP -- タイムスタンプ <br />DATE -- 日付時刻型      |
| Boolean(論理型)                            | BOOLEAN -- TRUE / FALSE                                      |
| Binary types(バイナリ型)                   | BINARY -- バイト序列                                         |

- MySQLを例として挙げると、BooleanとBinaryような類型がありません、どのデータ型にも該当しない。

- 一定の範囲内に数字データがINT型で扱われる。若し別の類型を扱いたい、例えばTINYINT 、SMALLINT或いはBIGINTに変換し、 末尾にY、S、 Lを追加できるんです。

  ```
  100Y – TINYINT, 100S – SMALLINT, 100L – BIGINT
  ```

- VARCHAR に対してデータ長さを決まっておく、使用しないバイトがリリースされる。CHARが駄目です。

- 関連性データベースと同じ、以上のキーワードはHQLを編纂する際にフィールドとして使えない。

Hive本質にJavaで開発され、その為両方のデータ類型が’基本的な一致です。

| Hiveデータ類型 | Javaデータ類型 | 長さ                  |
| -------------- | -------------- | --------------------- |
| TINYINT        | byte           | 1バイトの有符号整数型 |
| SMALLINT       | short          | 2バイトの有符号整数型 |
| INT            | int            | 4バイトの有符号整数型 |
| BIGINT         | long           | 8バイトの有符号整数型 |
| BOOLEAN        | boolean        | 論理型                |
| FLOAT          | float          | 単精度数点型          |
| DOUBLE         | double         | 双精度数点型          |
| STRING         | string         | 文字列型              |
| TIMESTAMP      |                | 日付時刻型            |
| BINARY         |                | 配列型                |

**データ類型の暗黙変換（Implicit Conversion）**

　　Hiveのデータ類型は暗黙変換が進行でき、Javaデータ変換に似る。例えば、ユーザーは検査中に整数型と小数点類型一緒に運算してる場合、自動的により大きいサイズへ変換される。Hiveには、以下の階層結構に因って、より下位の派生型（subtype）から上位の基本型（supertype）に変換する。

```
- Primitive Type
    - Number
     - DOUBLE
        - FLOAT
          - BIGINT
            - INT
              - SMALLINT
                - TINYINT
        - DECIMAL (Can be converted to String, varchar only)
        - STRING (Can be converted to Varchar, Double, Decimal)
        - VARCHAR (Can be converted to String, Double, Decimal)
          - DATE (Converted to String, varchar)
          - TIMESTAMP (same as date)
    - BOOLEAN
    - BINARY
```

規律纏め：

- より大きい類型へ変換できる。
- 全ての数字類型、String(数字)も、Doubleに変換できる。
- BOOLEANとBINARY類型は他の類型へ変換できない。

**データ類型の明示変換（Explicit Conversion）**

　　cast函数使うと、強制的に目標類型に変換できる。但し、強制的な変換はデータの精度の損失を引き起こす可能性がある為、注意が必要です。

```
select cast('1111' as int);
select cast('adsa' as int);
```

若し明示変換失敗したら、NULL返す。

### 第２節　集合データ類型

Hiveはarray、map、struct、unionを含み、集合データ類型も支持できると沢山関係性データベースにとってできない。

| 類型   | 描写                                     | 例                                                           |
| ------ | ---------------------------------------- | ------------------------------------------------------------ |
| ARRAY  | 秩序的、相同的な類型の集合               | array(1,2)                                                   |
| MAP    | key：必ず基本類型です<br />value：不限   | map('a', 1, 'b',2)                                           |
| STRUCT | 不同基本類型集合                         | struct('1',1,1.0)<br />named_struct('col1', '1', 'col2', 1, 'clo3', 1.0) |
| UNION  | 不同類型元素が一つのフィールドに保存する | create_union(1, 'a', 63, array(1,2))                         |

- 基本類型と同様にフィールドとして使えない。

- STRUCTがC言語のstructに似て、不同的な基本類型が集まる。
- UNIONがC言語のunionsに似て、どんな類型データも含まれられる。

### 第３節　データのテキストコーディング

　　Hiveテーブルのデータは実はテキスト形式でHDFSに保存され、特別な格式で格納されてある。Hiveの黙認コーディングがあるが、ユーザーの自分の格式を制定することも支持できる。一般に珍しい文字列を使って元のデリミタを置き換える。

**黙認デリミタ**

| デリミタ | 名称       | 説明                                                         |
| -------- | ---------- | ------------------------------------------------------------ |
| \n       | 改行文字列 | 毎行のデータ中間に分割                                       |
| ^A       | < Ctrl >+A | フィールドの中間に分割<br />SQL中に８進数 "\001" が表示する  |
| ^B       | < Ctrl >+B | ARRAY、MAP、STRUCTとUNIONの元素中間に分割<br />SQL中に８進数 ”\002” が表示する |
| ^C       | < Ctrl >+C | MAPのkey、value中間に分割<br />SQL中に８進数 \003 が表示する |

　　Hiveでは、データ格式が特別に規定されない、ユーザーに沿って指定することができる。データ格式を定義するには、列デリミタ（欠字、"\t"、"\x001"）、行デリミタ（”\n"”）及びデータの読み込み方式3つの属性が指定することが必要です。

　　データを読み込む過程に、データの自体何も変更されない、HDFS目録に単純ににコーピするんだけ。

　　Hiveから本地にダウンロードして黙認デリミタ（^A、^B、^C）が使用する場合。moreやcat命令でデリミタがテキスト中に表れません。ただ以下の命令が使用すればデリミタが顕示できる。

```
cat -A XXXX.dat
```

### 第４節　読み模式（schema on read）

　　通常のデータベースに、テーブルに合わないデータの読み込みが拒否される。データを書き込む際に検査を行い、符合かどうかを確認する。それは書き模式（schema on write）と呼ぶ。

　　一方、Hiveは読み模式が採用し、データを書き込んで検査しない、読み込む際に不符合的なデータがNULLを表示される。それは読み模式（schema on read）と呼ぶ。
