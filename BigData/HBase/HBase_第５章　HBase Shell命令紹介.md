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

```
# 添加
put 'studentInfo', 'rk1', 'base_info:name', 'John Doe'
# フィールド添加、一つ一つのフィールドでデータ添加だけ
put 'studentInfo', 'rk1', 'base_info:sex', '1'
put 'studentInfo', 'rk1', 'extra_info:math_grade', '85'
put 'studentInfo', 'rk1', 'extra_info:history_grade', '75'
# データ検査
get 'studentInfo', 'rk1'
```

![image-20231225072149913](D:\OneDrive\picture\Typora\image-20231225072149913.png)

```
# データ改修
put 'studentInfo', 'rk1', 'extra_info:history_grade', '94'
```

![image-20231225072647526](D:\OneDrive\picture\Typora\image-20231225072647526.png)
