# 分散大規模データ処理システム -- Hadoop-7

# 第４章　MapReduce計算フレームワーク-4

　MapReduce特性について紹介する。

## 第８節　MapReduce特性の紹介

### 8.1　MapReduce中のCombiner

　Combinerとは何、簡単に説明するならReducerと同じのものです。区別は実行する所が違う。

![image-20250728154715676](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250728154715676.png)

- CombinerはMapperとReducer以外にMapReduceコンポーネントの一つです。
- Combinerの親クラスはReducerで、それは何でReducerとはほとんど同じ。
- Combiner実行の位置は各節点に各MapTaskにSpill階段とMerge階段の間にいる。Reducerは任意のDateNodeに、それぞれの受け取るデータを集約する。
- Combinerの役割は、各MapTaskの出力をローカルで集約して、ネットワーク通信量を減らすことです。
- Combiner使いの前提は最後の結果に影響がない。

例えば、平均値計算のMR任務あり、Combinerの存在は最後の結果に及ぶ。

Combinerありの場合：

```
MapTask1出力：10,5,15		如果使用Combiner:(10+5+15)/3=10
MapTask2出力：2,6          如果使用Combiner:(2+6)/2=4
Reduce階段の集約				(10+4)/2=7
```

Combinerなしの場合：

```
(10+5+15+2+6)/5=7.6
```

　次はCombiner使い方を紹介する。

最近のカスタムパーティション案例を基づいて改造を行う。

**Combiner**

　Reducerの書き方に似る。区別は出力のkeyは必ずMapperの出力Keyと一致にする。

```
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class PartitionCombiner extends Reducer<Text, PartitionBean, Text, PartitionBean> {

    @Override
    protected void reduce(Text key, Iterable<PartitionBean> values, Reducer<Text, PartitionBean, Text, PartitionBean>.Context context) throws IOException, InterruptedException {
        for (PartitionBean bean : values) {
            context.write(key, bean);
        }
    }
}
```

**Driver**

　Combinerクラスを導入する。ある場合この案例は最後の出力値のkey値がNullWritableタイプなので、直接にPartitionReducerクラスを入れることは行けない。

```
job.setCombinerClass(PartitionCombiner.class);
```

　最後の出力ファイルは変わることがない。ただログにCombiner数が変わっている。

![image-20250729150743728](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250729150743728.png)

### 8.2　MapReduceの並べ替え

　並べ替えはデータ計算におけてはずっとなかなか重要な機能です。今回MapReduceフレームワークに並べ替え処理を紹介する。

　MapTaskとReduceTaskはデータのKey値で並べ替えを行うことがあり、それはMapReduceの黙認の動作です。業務流れに関わらず黙認の並べ替え方は文字列順序で処理する。

**MapTask**

- 環形キャッシュから書き出すSpill段階にKey値の文字列順序で並べ替える。
- Spill段階が終わり、各Spillファイトを併合する時に並び替えることがある。

**ReduceTask**

- 最後に全てのデータを出力する時、Key値の文字列順序で並べ替えてある。

　MapReduceは幾つか並べ替え方式を提供した。

**局部並び替え：**

　`<key, value>`データに並び替えを行う。各出力ファイルにデータ順序を確保する。

**全体並び替え：**

　最終の出力ファイルが必ず一つのみ、ファイルデータは順序がある。

**グループ並べ替え（GroupingComparator）：**

　`<key, value>`データをグループ化して並べ替える。それは一つのReduceTaskについてグループ化する。

**二次並べ替え：**

　同時に二つレコード項目について並べ替えを行う。

　局部と全体の並べ替えは複数のファイルを出力するかどうかだけで決められる。前の案例も使える。カスタムタグ並べ替えはもっと詳しく説明する必要です。次に案例を連れて紹介する。

#### 8.2.1　カスタムタグ並べ替え

　ここから案例を挙げてカスタムタグ並べ替えを紹介する。

　前にBeanクラスにの`toString()`メソッドを書き直すと最後の出力形式を決められる。並べ替えも、WritableComparableクラスを継承して`compareTo()`を書き直すによって実現する。

　

#### 8.2.2　GroupingComparator

　GroupingComparatorはReducer段階にコンポーネントの一つで、主に各ReduceTaskごとにデータをグループ化することを行う。SQLの`group by`に似ている。全体のデータをグループ化したいなら1つファイルを出力するだけでいい。



### 8.3　データ偏りの解決方法

　MapReduceにおけてデータ偏りとは、同じkeyが大量があり、同一のパーティションに割り当てられて1つのReduceTaskだけが重くなるという状況です。その中でこのTaskが非常に遅いために、全体の処理が遅くなり、最悪エラーが起こることもある。

　通用の解決方法はKey値に乱数を加える。

```
//案例
public class SkewedKeyMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    private Random random = new Random();
    private Text outKey = new Text();
    private IntWritable one = new IntWritable(1);

    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {
        String word = value.toString().trim();
        //0〜9の乱数を生成
        int randPrefix = random.nextInt(10);
        //Keyと組み合わせて分散度を高める
        outKey.set(randPrefix + "_" + word);
        context.write(outKey, one);
    }
}

```

　Redcuer段階にkeyを戻して集約を行う。

### 8.4　MapReduce入力と出力

　前の案件に入力と出力はデフォルトTextOutputFormat形式で進む。ここから別の入力と出力の形式について紹介する。

#### 8.4.1　InputFormat

　InputFormatはMapReduceに入力ファイルを読み込むに関する親クラスです。異なる入力ファイルに応じて、MapReduceが不同の子クラスを提供して処理できる。

　常に使うInputFormatの子クラス：

- TextInputFormat：通常のテキストファイル。MapReduceフレームワークのデフォルトの読み取り実装クラス。

- KeyValueTextInputFormat：1行のテキストデータを指定の区切り文字で読み取り、データをkey-value形式に変換。

- NLineInputFormat：データを指定した行数ごとに分割して読み取る。

- CombineTextInputFormat：小さなファイルをまとめて読み取り、Mapタスクの起動数を減らす。

- カスタムInputFormat：独自定義のInputFormat。

**CombineTextInputFormat**



**カスタムInputFormat**



#### 8.4.2　OutputFormat

　OutputFormatはMapReduceデータ出力の親クラス、どんなデータ出力はOutputFormatを実装することがある。

　常に使うOutputFormatの子クラス：

- TextOutputFormat：デフォルトの出力形式は `TextOutputFormat` で、各レコードを1行のテキストとして書き出す。KeyとVauleは任意の型でもよく、`TextOutputFormat` は `toString()` メソッドを使ってそれらを文字列に変換する。
- SequenceFileOutputFormat：圧縮されやすいフォーマットなので、`SequenceFileOutputFormat` を出力形式として使えば、後続の MapReduce タスクの入力として利用しやすい。

**カスタムOutputFormat**
