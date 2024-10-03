# 分散大規模データ処理システム -- Hadoop-6

# 第４章　MapReduce計算フレームワーク-2/2

　第４章には、主にMapReduceの運行原理に関する紹介をします。

## 第７節　MapReduceの原理分析

　全体のMapReduce流れがMapTaskとReduceTaskに分けておきて紹介を進めます。

### 7.1　MapTask運行仕組み

![Screenshot 2024-09-03 214128](C:\Users\Izaya\Desktop\Screenshot 2024-09-03 214128.png)

![img](D:\OneDrive\picture\Typora\BigData\Hadoop\a9b2a382aae117feefb7706a65771940.png)

**Read階段**

- まず、目標ファイルを実際に読み込む前にHDFSにアップロードしてあり、ブロック（block）に切り分けて各節点に割り当てます。
- データを読み込む時にgetSplits() メソッドを使用してブロックを切片（splits）に切り分けます。そのの切り分けが理論的で、ブロックになる物理的ではなりません。デフォルト場合でsplitsの大小が128Mで、ブロックのデフォルト大小と同じになります。つまり、デフォルト場合ではsplitとblockが一対一関係です。
- splitsの数量に応じて同じのMapTask数が起動されます。splitとblockの対応関係は、デフォルトでは1対1で、切り分けの大小がデフォルト128Mです。
- InputFormatクラスを継承するFileInputFormatが、getSplits()メソッドを実装してファイルの切り分けをします。この切り分けは論理的（ロジック）で、物理的な切り分けではありません。

![image-20240923114610607](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240923114610607.png)

- 

入力ファイルをsplitsに分割した後、RecordReaderオブジェクト（デフォルトではLineRecordReader）が\nを区切りとしてデータを読み込み、1行分のデータを<key, value>として返します。Keyは各行の先頭文字のオフセット値、valueはその行のテキスト内容を表します。

splitを読み込んで<key, value>を返した後、ユーザーが継承したMapperクラスに入り、ユーザーがオーバーライドしたmap関数を実行します。RecordReaderが1行読み取るごとに、この処理が1回呼び出されます。

mapのロジックが完了した後、mapの各結果は`context.write`を通じてデータ収集されます。このcollectの処理では、まずパーティショニングが行われ、デフォルトではHashPartitionerが使用されます。