# ビッグデータ高速計算エンジンSpark-3

# 第３章　RDDコーディング

## 第１節　RDDとは

　RDDはSparkの基盤となる仕組みで、Sparkのデータ処理における中心的な概念。

　RDD（Resilient Distributed Dataset：耐障害性分散データセット）は、簡単に言うと、複数のパーティションに分割され、並列処理できる分散データの集合を表す。

　RDD自体は変更不可で、一度作成したRDDの内容を直接変更することはできない。処理を加える場合は、新しいRDDが生成される。

#### RDDの5つの特徴

**1. パーティション（Partition）の集合**

　RDDは、複数のパーティションに分割されたデータセット。

　各パーティションはそれぞれTaskによって処理されるため、パーティション数が並列処理の粒度に関係する。RDDを作成するときにパーティション数を指定でき、指定しない場合はデフォルト値が使用される。

**2. パーティションごとに計算する函数（compute）**

　SparkのRDDは、パーティション単位で計算を行う。

　各RDDには `compute` 函数があり、Iteratorを使って各パーティションのデータを処理する。計算結果を毎回保存しておく必要はなく、必要になったときに計算できる。簡単と言えば、「そのRDDの1つのPartitionをどう計算するか」をまとめた処理です。

```
RDD
├─ Partition 0
├─ Partition 1
└─ Partition 2

Task 0 → compute(Partition 0)
Task 1 → compute(Partition 1)
Task 2 → compute(Partition 2)
```

**3. RDD同士の依存関係（Lineage）**

　RDDに変換処理を行うと、新しいRDDが作られ、RDD同士に前後の依存関係（Lineage）ができる。

　一部のパーティションデータが失われた場合でも、この依存関係をたどって失われた部分だけを再計算できる。それはなんでRDD自体は変更不可わけです。これがRDDの耐障害性を支える重要な仕組み。

**4. Partitioner**

　Key-Value形式のRDDでは、Partitionerを持つことがある。

　Sparkでは代表的に次の2種類がある。

- `HashPartitioner`：Keyのハッシュ値を使って振り分ける
- `RangePartitioner`：Keyの範囲によって振り分ける

　基本的にPartitionerはKey-Value形式のRDDで利用される。PartitionerはRDDのパーティション構成や、Shuffle時のデータの振り分けに関係する。

**5. 優先実行場所（Preferred Location）**

　RDDは、各パーティションをどのノードで処理するのが効率的かという情報を持つことができる。

　例えば、HDFS上のデータなら、データブロックが保存されているノードが優先される。「データを計算場所へ移動する」のではなく、「計算をデータのある場所へ移動する」というデータローカリティ（Data Locality）の考え方。

　つまりSparkは、可能な限りデータが存在するノードの近くでTaskを実行し、ネットワーク転送を減らすようにスケジューリングする。

## 第２節　RDDの特徴（詳細）

#### **パーティション（Partition）**

　RDDは論理的に複数のパーティションに分割されている。各パーティションのデータは実体として保持されているのではなく、必要になったときに `compute()` 関数を使って取得・計算される。

　`compute()` の処理内容は、RDDがどのように作られたかによって異なる。

- **ファイルシステムから作成したRDD**

  　`compute` が指定されたファイルからデータを読み込む。

- **別のRDDから変換して作成したRDD**

  　`compute` が元のRDDに対して変換処理を行い、必要なデータを計算する。

![image-20260921200757664](D:\OneDrive\picture\Typora\BigData\Spark\image-20260921200757664.png)

#### 読み取り専用（Immutable）

　RDDは一度作成すると、中身を直接変更できない。RDDのデータを変更したい場合は、元のRDDを書き換えるのではなく、既存のRDDをもとに新しいRDDを作成する。

　RDDから別のRDDへの変換には、さまざまな演算子が用意されている。`map`、`filter`、`union`、`join`、`reduceByKey` など。MapReduceのように、基本的に `map` と `reduce` だけで処理を組み立てる必要がなく、より柔軟にデータ処理を記述できる。

RDDの操作は大きく2種類：

- **Transformation（変換）**
   　RDDから新しいRDDを作る処理、`map` や `filter` などが該当する。Transformationを呼び出しただけでは実際の計算はまだ行われない。
- **Action（アクション）**
   　実際の計算を開始する処理、計算結果を取得したり、ファイルシステムに保存したりする。`collect`、`count`、`saveAsTextFile` などが該当する。

　つまり、RDD → Transformation → 新しいRDD → Transformation → 新しいRDD → Action → 実際に計算開始というイメージです。

![image-20260921202650911](D:\OneDrive\picture\Typora\BigData\Spark\image-20260921202650911.png)

#### 依存関係（Dependency）

　RDDにTransformationを行うと、新しいRDDが生成される。新しいRDDには、「どのRDDから、どのような処理を経て作られたか」という情報が記録される。このRDD同士のつながりを Lineage（系統・依存関係） と呼ぶ。

RDDの依存関係は、大きく2種類に分けられる。

- **狭い依存関係（Narrow Dependency）**
   　親RDDと子RDDのパーティションの関係が比較的単純で、基本的に 1対1、または複数対1（n:1） になる。例：`map`、`filter`
- **広い依存関係（Wide Dependency）**
   　子RDDの1つのパーティションを作るために、親RDDの複数のパーティションからデータを集める必要がある。このとき Shuffle（シャッフル）が発生する。例：`reduceByKey`、`groupByKey`

　簡単に区別するはShuffle（シャッフル）が発生するかどうか。

![image-20260921204704867](D:\OneDrive\picture\Typora\BigData\Spark\image-20260921204704867.png)

#### キャッシュ（Cache）

　RDDは、計算したデータをメモリやディスクにキャッシュして再利用できる。同じRDDをアプリケーション内で何度も使う場合、そのRDDをキャッシュしておくと効率がよい。最初にRDDを計算するときは、依存関係（Lineage）を辿って各パーティションのデータを計算する。一度キャッシュされると、次回以降は依存関係を辿って再計算する必要がなく、キャッシュから直接データを取得できる。

　つまり、初回にLineageをたどる → 計算 → キャッシュに保存し、2回目以降はキャッシュから直接取得できる。そうなるため、同じRDDを繰り返し利用する処理を高速化できる。

![image-20260921205625655](D:\OneDrive\picture\Typora\BigData\Spark\image-20260921205625655.png)

#### Checkpoint（チェックポイント）

　RDDは依存関係を持っているため、一部のパーティションが失われても、依存関係を辿って失われたデータを再計算できる。ただし、処理を何度も繰り返すようなアプリケーションでは、RDD同士のLineageがどんどん長くなる。この状態で途中のデータが失われると、長い依存関係を遡って再計算する必要があり、処理に時間がかかる。そこで使うのがチェックポイント（Checkpoint）。

　Checkpointを設定すると、RDDのデータをHDFSなどの永続ストレージに保存し、それ以前の依存関係を切り離すことができる。その後は元の親RDDまで遡る必要がなく、Checkpointに保存されたデータを起点として処理を再開できる。

　つまり、以前は「親RDD → 親RDD → 親RDD → 現在のRDD」、今は「Checkpoint → 現在のRDD」となり、依存関係が長くなりすぎた場合の再計算コストを抑えられる、という仕組みです。

## 第３節 Sparkのプログラミングモデル

　Sparkでは、DB・ファイルシステム・HDFSなどのさまざまなデータソースからデータを読み込み、RDDとして処理する。基本的な流れは「データソース → SparkContext → RDD → Transformation / Action → 結果出力」となる。

![image-20260923112626416](D:\OneDrive\picture\Typora\BigData\Spark\image-20260923112626416.png)

- 先ずSparkContextは、Sparkプログラミングの入り口にあたる。例えば、Javaの方はDBをアクセスしくてJDBCに関するクラスを新規する必要があり、SparkContextはSparkを利用して最初の連接クラスです。
- RDDは処理対象となるデータを表して複数のパーティションに分かれます。
- RDDを変換する処理を Transformation と呼ぶ。さまざまなメソッドを呼び出し、データを変換・処理する
- Transformationは遅延評価（Lazy Evaluation）で、呼び出しただけでは実際の計算は行われない。Actionが呼び出されたタイミングで、必要なTransformationがまとめて実行される。
- 最後に計算結果を画面に表示したり、外部ストレージへ保存したりする。
- 大体「RDDはデータ」「Transformationは計算方法の定義」「Actionは実際の計算を開始させるもの」 の3つを中心に理解してオッケーです。

#### DriverとWorkerの役割

Sparkアプリケーションを実行するには、Driverプログラムを作成し、クラスターに投入する。

- Driver側では、RDDを作成し、`map` や `filter` などの処理内容を定義する。

- Worker側では、実際にRDDの各パーティションに対する定義した計算処理を実行する。

![image-20260923112835699](D:\OneDrive\picture\Typora\BigData\Spark\image-20260923112835699.png)