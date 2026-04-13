# 分散大規模データ処理システム -- Hadoop-7

# 第４章　MapReduce計算フレームワーク-4

　MapReduce特性について紹介する。

## 第８節　MapReduce特性の紹介

### 8.1　MapReduce中のCombiner

　Combinerとは何か、簡単に言うと、Reducerとほぼ同じ処理を行うコンポーネントですが、実行される場所が違う。

![image-20250728154715676](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250728154715676.png)

- Combiner役割は、Reducer流れに入る前にMapTaskの出力データを集約し、ノード間のネットワーク通信量を削減することです。
- Reducer同様にデータを集約するが、実行される場所が異なる。Combiner各MapTask内で、SpillとMergeの間に実行される。Reducerはは各ノード上で、MapTaskから受け取ったデータを集約する。
- Combinerの親クラスはReducerであり、処理内容もほぼ同じです。
- Combinerは、MapperとReducerに加えて存在するMapReduceのコンポーネントの一つです。
- Combinerを使用する前提として、適用しても最終結果に影響が出ない処理である必要がある。

　例えば、平均値を計算するMapReduceの処理では、Combinerをそのまま適用すると、最終結果が変わってしまう。

Combiner使う：

```
MapTask1出力：10,5,15		Combiner:(10+5+15)/3=10
MapTask2出力：2,6         	Combiner:(2+6)/2=4
Reduce階段の集約				(10+4)/2=7
```

Combiner使ってない：

```
(10+5+15+2+6)/5=7.6
```

　次に、Combiner の使い方を紹介する。先ほどのカスタムパーティションの例をもとに、一部改造しながら説明していく。

**Combiner**

　Reducerの実装とほぼ同じ書き方になる。区別は出力のkeyは必ずMapperの出力Keyと一致する。

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

　Combinerクラスを導入する。

　`PartitionCombiner` と `PartitionReducer` は同じ親クラス`Reducer`を継承しているため、基本的には同様に扱うことができる。ただし、本例では最終出力の key が `NullWritable` 型であるため、`PartitionReducer` クラスをそのままCombinerとして使用することはできない。

```
job.setCombinerClass(PartitionCombiner.class);
```

　最終的な出力ファイルの内容は変わない。ただし、ログにはCombinerの実行回数が記録されるようになる。

![image-20250729150743728](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250729150743728.png)

### 8.2　MapReduceの並べ替え

　並べ替えはデータ処理においては非常に重要な機能です。今回MapReduceフレームワークにおける並べ替え処理について説明する。

　MapTaskとReduceTask両方で、データはKey値に基づいて並べ替えが行われる。それはMapReduceのデフォルト動作であり、特別な設定をしない場合は文字列の順序（辞書順）で処理される。

- MapTask段階

  - 環形キャッシュから書き出すSpill段階にKey値の文字列順序で並び替える。

  - Spill段階が終わり、各Spillファイルを併合する時にも、並べ替えることがある。


- ReduceTask段階

  - 最後にデータが出力される時、Key値の文字列順序に整列されている。


　MapReduceでは、いくつかの並べ替え方法が提供されている。

- **局所並べ替え**：`<key, value>`データに対して並べ替えを行い、各出力ファイルにデータの順序を保証する。

- **全体並べ替え**：最終的な出力ファイルが必ず1つにし、そのファイル内のデータを全体として順序付ける。

- **グループ並べ替え（GroupingComparator）**：`<key, value>` データをグループ単位でまとめて処理する。 同一グループは1つのReduceTask 内でまとめて処理される。

- **二次並べ替え**：複数フィールドを組み合わせて並べ替えを行う。

　局部と全体の並べ替えは複数のファイルを出力するかどうかだけで決められる。カスタムタグ並べ替えはもっと詳しい説明が必要となるため、次に例を連れて紹介する。

#### 8.2.1　WritableComparableカスタムタグ並べ替え

　前述のとおり、Beanクラスの`toString()`メソッドを書き直すで最後の出力形式を制御できる。同様に、並べ替えはWritableComparableクラスを継承して`compareTo()`を書き直すことで実現できる。

　ここでは、設備使用時間データをもとに設備の銘柄の番号と使用時間の2つの項目で二次並べ替えを行う。

```
0a8cd1,kar_992134,149.187.152.169,4054
0ce73e,kar_334455,181.133.189.160,8938
034347,kar_334455,157.86.7.229,821
0cdf27,kar_992134,233.205.58.168,9709
0b66c9,kar_553322,236.9.100.245,9921
0c35e3,kar_999999,49.179.180.211,5672
0b6b5d,kar_786544,114.179.97.23,7648
01b85a,kar_553322,131.200.122.190,1863
0bdn27,kar_786544,233.185.58.152,371900
```

**Bean**

- `WritableComparable`を実装する。
- `compareTo()`メソッドを上書きすることで、複数の項目を使った比較を実現できる。
- `compareTo()`の戻り値は、比較結果を表す（正の値：大きい、0：等しい、負の値：小さい）。
- `compareTo()`と`toString()`メソッドは、Redcuerのファイルにデータを出力する段階では有効だ。

```
public class SortBean implements WritableComparable<SortBean> {

    private String id;         // 設備id
    private String model;      // 型番
    private String netIp;      // ネットIP
    private Long usageTime;     // 使用時間（分など）

    public String getModel() {
        return model;
    }

    public Long getUsageTime() {
        return usageTime;
    }

    public void setId(String id) {
        this.id = id;
    }

    public void setModel(String model) {
        this.model = model;
    }

    public void setNetIp(String netIp) {
        this.netIp = netIp;
    }

    public void setUsageTime(Long usageTime) {
        this.usageTime = usageTime;
    }

    @Override
    public String toString() {
        return id + "   " +
                model + "   " +
                netIp + "   " +
                usageTime;
    }

    @Override
    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeUTF(id);
        dataOutput.writeUTF(model);
        dataOutput.writeUTF(netIp);
        dataOutput.writeLong(usageTime);
    }

    @Override
    public void readFields(DataInput dataInput) throws IOException {
        this.id = dataInput.readUTF();
        this.model = dataInput.readUTF();
        this.netIp = dataInput.readUTF();
        this.usageTime = dataInput.readLong();
    }

    @Override
    public int compareTo(SortBean sortBean) {
        int result;
        String[] modelStr = sortBean.getModel().split("_");
        Long modelKey = Long.parseLong(modelStr[1]);
        String[] modelStr2 = model.split("_");
        Long modelKey2 =  Long.parseLong(modelStr2[1]);
        if (modelKey2 > modelKey) {
            result = 1;
        } else if (modelKey2 < modelKey) {
            result = -1;
        } else {
            if (usageTime > sortBean.getUsageTime()) {
                result = 1;
            } else if (usageTime < sortBean.getUsageTime()) {
                result = -1;
            } else {
                result = 0;
            }
        }
        return result;

    }
}
```

**Mapper**

- 出力メソッド`context.write()`のKeyはsortBean対象に変更し、valueには`NullWritable.get()`に設定する。
- Keyの順序で並べ替えするのはフレームワークの黙認動作であり、その際に呼び出されるのは`compareTo()`メソッドです。
- 今の`compareTo()`メソッド引数は同じ `SortBean` 型となるため、`sortBean`をKeyとして渡す必要がある。

```
public class SortMapper extends Mapper<LongWritable, Text, SortBean, NullWritable> {
    @Override
    protected void map(LongWritable key, Text value, Context context)
            throws IOException, InterruptedException {
        //一行文字を取得
        String str = value.toString();
        //文字を区切って単語数組になる
        String[] strs = str.split(",");
        SortBean sortBean = new SortBean();
        sortBean.setId(strs[0]);
        sortBean.setModel(strs[1]);
        sortBean.setNetIp(strs[2]);
        sortBean.setUsageTime(Long.parseLong(strs[3]));
        //出力
        context.write(sortBean, NullWritable.get());
    }
}
```

**Reducer**

　特別の変わりがない。

```
public class SortReducer extends Reducer<SortBean, NullWritable, NullWritable, SortBean> {
    @Override
    protected void reduce(SortBean key, Iterable<NullWritable> values, Context context) throws IOException, InterruptedException {
        for (NullWritable value : values) {
            context.write(value, key);
        }
    }
}
```

**Driver**

　Mapperの出力タイプは変更が必要です。

```
public class SortDriver {

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        //配置ファイルを設定とJobを作成
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);
        //Mapper、Reducer、Driverクラスを添加
        job.setJarByClass(SortDriver.class);
        job.setMapperClass(SortMapper.class);
        job.setReducerClass(SortReducer.class);
        //Mapの出力値のタイプ
        job.setMapOutputKeyClass(SortBean.class);
        job.setMapOutputValueClass(NullWritable.class);
        //最終出力値のタイプ
        job.setOutputKeyClass(NullWritable.class);
        job.setOutputValueClass(SortBean.class);

        job.setNumReduceTasks(0);
        //入力と出力ファイルのアドレス
        FileInputFormat.setInputPaths(job, new Path("D:\\partition.txt"));
        FileOutputFormat.setOutputPath(job, new Path("D:\\output"));
        //タスクをコミット
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

　複数フィールドの並べ替えです。

![image-20260413223135258](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260413223135258.png)

![image-20260413221050579](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260413221050579.png)

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
