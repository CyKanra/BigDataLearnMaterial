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
- データを読み込む時にgetSplits() メソッドを使用してブロックを切片（splits）に切り分けます。そのの切り分けが理論的で、ブロックになる物理的ではなりません。デフォルト場合でsplitsの大小が128Mで、ブロックのデフォルト大小と同じになります。つまり、デフォルト場合ではsplitとblock関係が一対一です。あと、splitsの数量に応じて次の階段の同じなMapTask数を起動されてあります。
- InputFormatクラスを継承するFileInputFormatが、getSplits()メソッドを実装してファイルの切り分けをします。この切り分けは論理的（ロジック）で、物理的な切り分けではありません。

![image-20240923114610607](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240923114610607.png)

- 切り分けされたのsplit情報、実行するコードを含むjarファイル、そしてジョブ（Job）実行に必要な設定情報（XML設定ファイルなど）を作成します。この一連の情報がまとめられ、YARNに提出されます。
- YARNは、この提出されたジョブに対して必要なリソース（メモリ、CPUなど）を割り当ててジョブを実行します。YARNはまず MrAppMaster（MapReduce Application Master）を起動します。MrAppMasterは、ジョブの周期を管理し、各タスクの実行順序を制定します。後のMapTaskとReduceTask、MapReduceジョブ全体がMrAppMasterの制御で完成されます。

**Map階段**

- Map階段に実行するロジックは書き直されたのmap()メソッドです。
- 入力の[key/value]値が分別ドキュメントの行数keyと当の行の文字です。出力の[key/value]値が一つ単語textとその単語の計数1です。旧の解析されて出たkey/value値を輸出したいkey/value値に転換する過程です。

![image-20241014162847885](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20241014162847885.png)

入力ファイルをsplitsに分割した後、RecordReaderオブジェクト（デフォルトではLineRecordReader）が\nを区切りとしてデータを読み込み、1行分のデータを<key, value>として返します。Keyは各行の先頭文字のオフセット値、valueはその行のテキスト内容を表します。

splitを読み込んで<key, value>を返した後、ユーザーが継承したMapperクラスに入り、ユーザーがオーバーライドしたmap関数を実行します。RecordReaderが1行読み取るごとに、この処理が1回呼び出されます。

mapのロジックが完了した後、mapの各結果は`context.write`を通じてデータ収集されます。このcollectの処理では、まずパーティショニングが行われ、デフォルトではHashPartitionerが使用されます。