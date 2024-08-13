# 分散大規模データ処理システム -- Hadoop-5

# 第４章　MapReduce計算フレームワーク

　始める前にちょっとMapReduce使用現状を話します。Hadoop中にMapReduce計算フレームワークにおける知識が一番多いです。だが、実際の運用中にMapReduce登場する場合が少ない。Spark、Finkなどもっと優秀の計算フレームワークに代わられます。一方で、MapReduceコーディングは確かに複雑です。特にビッグデータに関わる計算には、MapReduceコーディングもう満足しません。MapReduceコーディングしなくても構わなく読み取れるとはオーケーです。従って、本章はコードに対して講釈を先頭にを置き、その後でMapReduce内部仕組みを講釈します。

## 第１節　MapReduceの紹介

　MapReduceはHadoop計算フレームワークとして日常タスクを処理します。どうやって大規模計算を実行するには、MapReduceの思想の核心は「分割統治」というもので、並列処理の利点を十分に活用しています。

MapReduceのタスクプロセスは、2つの処理段階に分かれています：

- Map段階：この段階の主な役割は「分割」です。つまり、複雑なタスクをいくつかの単独なタスクに分解して並列処理します。Map段階のこれらのタスクは並行して計算でき、互いに依存関係がない。

- Reduce段階：この段階の主な役割は「統合」です。Map段階の結果を全体的に集約する。

![Hadoop - Architecture - GeeksforGeeks](D:\OneDrive\picture\Typora\BigData\Hadoop\mapreduce-workflow.png)

## 第２節　WordCount案例の分析

　公式の案例で、文字の単語出現回数を数えて統計する機能です。`hadoop-mapreduce-examples-2.9.2.jar`パッケージが`/opt/bigdata/servers/hadoop-2.9.2/share/hadoop/mapreduce`ディレクトリにいます。

![image-20240808152733378](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808152733378.png)

　HadoopサービスからダウンロードしてIdeaに導入します。WordCountクラスは目標ファイルです。

![image-20240808153229676](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808153229676.png)

　WordCountコードは三つの部分に分けます。

- Map：分割流れに対応して、Mapper親クラスを継承して親クラスのmapメソッドを書き直す必要です。Map階段のロジックがここに書きます。

![image-20240808160200736](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808160200736.png)

　入力値keyが一行文字のオフセット（offset）で、一応行数を理解していい、特に使われていません。入力値valueが一行文字の内容です。大体のロジックは、mapメソッドが一行文字を受けて空白で仕切りを行います。複数の単語を含める結果値`itr`がループ処理をします。一つ一つ単語を「<単語, 出現回数=1>」形式で`context`に書き込みます。

- Reduce：統合流れに対して、Reducer親クラスを継承して親クラスのreduceメソッドを書き直す必要です。Reduce階段のロジックがここに書きます。

![image-20240811153315099](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240811153315099.png)

　入力値keyがMap階段から出力のkey値で、key値ごとが一種の単語を表示します。valuesがMapから送られてきた出現回数のリストです。key値に対してvaluesが`[1,3,2]`ようなデータリストです。リストの各数字は、その行にその単語が出現した回数を表します。values総合と対応のkey値が＜key, value+＞形で`context`に書き込みます。

- Driver：タスクを運行するコードです。配置クラスの実装と入力値のチェック、Job対象の設定、入力と出力ファイルのアドレスと最後のJobタスクの実行という部分があります。

![image-20240812165236008](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240812165236008-1723449222995-1.png)

　MapReduceコーディングには、開発者がMap分割とReduce統合のロジックを関心するだけでいい、どうやって分割、統合内容を考慮する必要ありません。

## 第３節　手動的にWordCount機能を実現

　MapReduceコーディングの規範に従って自らWordCount機能のプログラムを実現します。IdeaツールでMavenプログラムを新作してMapper，Reducer，Driverコーディングをします。

**実現目標**

　指定のコンテストにの単語の出現回数を統計します。

入力ファイル：wc.txt

出力ファイル：下記のようです

```

```

**実現手順**

- Hadoop依頼を導入

```
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-hdfs</artifactId>
    <version>2.9.2</version>
</dependency>
```

- Mapper階段

```
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import java.io.IOException;

public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    Text text = new Text();
    IntWritable intWritable = new IntWritable(1);
    @Override
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, IntWritable>.Context context)
            throws IOException, InterruptedException {
        //一行文字を取得
        String str = value.toString();
        //文字を区切って単語数組になる
        String[] strs = str.split(" ");
        //出力
        for (String s : strs) {
            text.set(s);
            context.write(text, intWritable);
        }
    }
}
```

- Reducer階段

```
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    Integer sum;
    IntWritable intWritable = new IntWritable();
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Reducer<Text, IntWritable, Text, IntWritable>.Context context) {
        sum = 0;
        //累積加算
        for (IntWritable value : values) {
            sum += value.get();
        }
        //出力
        intWritable.set(sum);
    }
}
```

- Driver階段

```
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordcountDriver {
    public static void main(String[] args) throws IOException,
            ClassNotFoundException, InterruptedException {
        //配置ファイルを設定とJobを作成
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);
        //Mapper、Reducer、Driverクラスを添加
        job.setJarByClass(WordcountDriver.class);
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);
        //Mapの出力値のタイプ
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);
        //最終出力値のタイプ
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        //入力と出力ファイルのアドレス
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        //タスクをコミット
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

- プログラムを運行

　ここでローカル運行を選べて分散サービスを依頼しません。
