# 分散大規模データ処理システム -- Hadoop-6

# 第４章　MapReduce計算フレームワーク-2/2

　第４章の第二目部分は、主にMapReduceの運行原理に関する紹介をします。

## 第７節　MapReduceの原理分析

### 7.1　MapTask運行仕組み

![Screenshot 2024-09-03 214128](C:\Users\Izaya\Desktop\Screenshot 2024-09-03 214128.png)

- まず、データを読み取るコンポーネントであるInputFormat（デフォルトではTextInputFormat）が、`getSplits` メソッドを使用して入力ディレクトリ内のファイルを論理的に切り分けてsplitsを取得します。splitsの数に応じて、同じ数のMapTaskが起動されます。splitとblockの対応関係は、デフォルトでは1対1です。

入力ファイルをsplitsに分割した後、RecordReaderオブジェクト（デフォルトではLineRecordReader）が\nを区切りとしてデータを読み込み、1行分のデータを<key, value>として返します。Keyは各行の先頭文字のオフセット値、valueはその行のテキスト内容を表します。

splitを読み込んで<key, value>を返した後、ユーザーが継承したMapperクラスに入り、ユーザーがオーバーライドしたmap関数を実行します。RecordReaderが1行読み取るごとに、この処理が1回呼び出されます。

mapのロジックが完了した後、mapの各結果は`context.write`を通じてデータ収集されます。このcollectの処理では、まずパーティショニングが行われ、デフォルトではHashPartitionerが使用されます。