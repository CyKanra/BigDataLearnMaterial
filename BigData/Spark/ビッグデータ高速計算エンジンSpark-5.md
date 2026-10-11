# ビッグデータ高速計算エンジンSpark-4

# 第３章　RDDコーディング

## 第４節　RDDの作成

#### SparkContextとは

　SparkContextは、Sparkアプリケーションの入り口となるクラスで、Sparkクラスターとの接続を管理し、クラスター全体とのやり取りを担当する。Sparkサービスを使う限り、SparkContextは最初の作るクラスです。

　主な役割は次のとおり。

- クラスターとの接続：Sparkクラスターに接続し、リソース管理システムと連携する。
- RDDの作成：分散処理に使用するRDDを生成する。
- Accumulator（アキュムレータ）：複数のTaskから値を集計するための共有変数を作成する。
- Broadcast Variable（ブロードキャスト変数）：共通データを各Executorに効率よく配布する。

　最後の二点はプログラム内部の処理で、開発者が何もしなくていい。SparkContextクラスの作成とRDDの定義を開発者がするとは必要です。

　あと、SparkContextの注意点いくつかある。

- 1つのJVM内で同時に有効にできるSparkContextは、原則として1つだけ。
- 新しいSparkContextを作成する場合は、既存のSparkContextを `stop()` で停止する必要がある。
- SparkContextの`SparkConf` 設定ファイルを使って、改めてアプリケーションの設定を指定できる。

#### SparkContext作成

　本節は主にRDDの様々函数を紹介する。開発を簡単にするため、今回はSparkに標準搭載されているSpark Shellを使ってその上にコードを書いている。Sparkの運行モードもクラスタモード--Standaloneに替え、Yarnのサービスが要らなくなる。

**spark-defaults.conf**

　Yarnの設定はコメントにする。

![image-20261010164346893](D:\OneDrive\picture\Typora\BigData\Spark\image-20261010164346893.png)

　変更の設定ファイルを各ノードに配布するのを忘れないように。

![image-20261010164811804](D:\OneDrive\picture\Typora\BigData\Spark\image-20261010164811804.png)

　先ず、SparkのログはHDFSに格納されているため、Spark起動する前にHDFSサービスが要る。その後、Sparkのstart-all.shファイルを実行してStandaloneモードSparkサービスを起動する。

![image-20261010170114346](D:\OneDrive\picture\Typora\BigData\Spark\image-20261010170114346.png)

SparkサービスURL：http://192.168.31.135:8080/

![image-20261010170330690](D:\OneDrive\picture\Typora\BigData\Spark\image-20261010170330690.png)

　spark-shellを起動すると、SparkContextも自動的に作成されるため、自分で初期化する必要がなくてその画面にコードを書いて直接に実行できる。

![image-20261010171727154](D:\OneDrive\picture\Typora\BigData\Spark\image-20261010171727154.png)
