# 分散大規模データ処理システム -- Hadoop-5

# 第４章　MapReduce計算フレームワーク

　始める前にちょっとMapReduce使用現状を話します。Hadoop中にMapReduce計算フレームワークにおける知識が一番多いです。だが、実際の運用中にMapReduce登場する場合が少ない。Spark、Finkなどもっと優秀の計算フレームワークに代わられます。一方で、MapReduceコーディングは確かに複雑です。特にビッグデータに関わる計算には、MapReduceコーディングもう満足しません。MapReduceコーディングしなくても構わなく読み取れるとはオーケーです。従って、本章はコードに対して講釈を先頭にを置き、その後でまだMapReduce内部仕組みを講釈します。

## 第１節　MapReduceの紹介

　MapReduceはHadoop計算フレームワークとして日常タスクを処理します。どうやって大規模計算を実行するには、MapReduceの思想の核心は「分割統治」というもので、並列処理の利点を十分に活用しています。

MapReduceのタスクプロセスは、2つの処理段階に分かれています：

- Map段階：この段階の主な役割は「分割」です。つまり、複雑なタスクをいくつかの単独なタスクに分解して並列処理します。Map段階のこれらのタスクは並行して計算でき、互いに依存関係がない。

- Reduce段階：この段階の主な役割は「統合」です。Map段階の結果を全体的に集約する。

![Hadoop - Architecture - GeeksforGeeks](D:\OneDrive\picture\Typora\BigData\Hadoop\mapreduce-workflow.png)

## 第２節　WordCount案例の分析

　公式の案例で、文字の単語出現回数を数えて統計する働きです。`hadoop-mapreduce-examples-2.9.2.jar`パッケージが`/opt/bigdata/servers/hadoop-2.9.2/share/hadoop/mapreduce`ディレクトリにいます。

![image-20240808152733378](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808152733378.png)

　HadoopサービスからダウンロードしてIdeaに導入します。WordCountクラスは目標ファイルです。

![image-20240808153229676](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808153229676.png)

　WordCountコードは三つの部分に分けます。

- Map：分割流れに対応して、Mapper親クラスを継承するのTokenizerMapperクラスで、親クラスのmapメソッドを書き直す必要です。Map階段のロジックがここに書きます。

![image-20240808160200736](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808160200736.png)

　入力値keyが一行文字のオフセット（offset）で、一応行数を理解していい、特に使われていません。入力値valueが一行文字の内容です。大体のロジックは、mapメソッドが一行文字を受けて空白で仕切りを行います。複数の単語を含める結果値`itr`がループ処理をします。一つ一つ単語を「<単語, 出現回数1>」形式で`context`に書き込みます。

- Reduce：統合流れに対して、Reducer親クラスを継承するのIntSumReducerクラスで、親クラスのreduceメソッドを書き直す必要です。Reduce階段のロジックがここに書きます。

![image-20240811153315099](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240811153315099.png)

　入力値keyがMap階段から出力のkey値で、つまり具体的な単語です。

