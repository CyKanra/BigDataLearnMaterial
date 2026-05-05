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

　次に、Combinerの使い方を紹介する。先ほどのカスタムパーティションの例をもとに、一部改造しながら説明していく。

**Combiner**

　Reducerの実装とほぼ同じ書き方です。区別は出力のkeyは必ずMapperの出力Keyと一致する。

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

　MapTaskとReduceTask両方で、データはKey値に基づいて並べ替えが行われる。それはMapReduceのデフォルト動作であり、特別な設定をしない場合で文字列の順序（辞書順）で処理される。

MapTask段階

- 環形キャッシュから書き出すSpill段階にKey値の文字列順序で並び替える。

- Spill段階が終わり、各Spillファイルを併合する時にもKey値の文字列順序で並べ替える。

ReduceTask段階


- MapReduce流れの最後にデータが出力ファイルに書き込む時、出力ファイルごとに並べ替えを行う。


ReduceTask段階で幾つか並べ替え方法が提供される。

- 局所並べ替え：`<key, value>`データに対して並べ替えを行い、出力ファイルごとにデータの順序を保証する。

- 全体並べ替え：最終的な出力ファイルが必ず1つにし、そのファイル内のデータを全体として順序付ける。

- グループ並べ替え（GroupingComparator）：`<key, value>` データをグループ単位でまとめて処理する。 同一グループは1つのReduceTask 内でまとめて処理される。

- 二次並べ替え：複数フィールドを組み合わせて並べ替えを行う。

　ReduceTaskのデフォルトの並べ替えはMapTaskの入力値Keyに基づき、Key値の文字列順序に整列されている。sqlの`order by`みたい任意項目の並べ替えとは違う。また、並べ替え、グループ分けなどの処理はMapTaskからReduceTaskに併合される段階に役立つ。

　次に、カスタムタグ並べ替え案例を紹介する。

#### 8.2.1　WritableComparableカスタムタグ並べ替え

　前述のとおり、WritableComparableクラスの`toString()`メソッドを書き直すで最後の出力形式を制御する。同様に、並べ替えはWritableComparableクラスを継承して`compareTo()`を書き直すことで並べ替えの制御を実現できる。

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
@Data
public class SortBean implements WritableComparable<SortBean> {

    private String id;         // 設備id
    private String model;      // 型番
    private String netIp;      // ネットIP
    private Long usageTime;     // 使用時間（分など）

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

**結果**

　複数フィールドの並べ替え。

![image-20260413223135258](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260413223135258.png)

![image-20260413221050579](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260413221050579.png)

#### 8.2.2　GroupingComparatorグループ並べ替え

　GroupingComparatorは、MapReduceのReduce側で使われる機能コンポーネントであり、どのデータを1つのグループとしてReduceに渡すことを決める役割を持つ。

　デフォルトでは同じのKeyを持つデータを1つグループとしてReduce処理に渡す。例えば、Keyは注文ID場合、同じ注文IDデータが1つのグループに分配されるてReduceにあげる。ただし、KeyはBeanクラス場合で互いに異なり、同じの注文IDでグループされることができない。

　GroupingComparatorはその問題を解決するためです。GroupingComparatorをカスタマイズすることで、異なるkeyでも同じグループとして纏めてReduceロジックを実行する。

　ここはPartitionerと役割を混乱しやすい。同じのPartitionerのデータは1つのReduceTaskに渡される。GroupingComparatorはどんなデータをグループとしてReduce処理を実行する。例えば、注文Aと注文Bが同じパーティションにいるけど、不同の2回Reduce実行にいる。

　次は案例を連れて説明する。下に請求書ID、商品ID、価額を含める取引記録である。要件は同じ請求書に一番高い価額の取引を取得してファイルに出力する。

```
Order_0000005,Pdt_07,60.0
Order_0000002,Pdt_05,722.4
Order_0000008,Pdt_01,180.0
Order_0000001,Pdt_05,25.8
Order_0000007,Pdt_09,45.0
Order_0000003,Pdt_02,99.9
Order_0000006,Pdt_08,33.3
Order_0000004,Pdt_06,88.8
Order_0000002,Pdt_03,522.8
Order_0000009,Pdt_07,120.0
Order_0000005,Pdt_01,199.9
Order_0000010,Pdt_02,130.0
Order_0000008,Pdt_04,140.0
Order_0000007,Pdt_02,210.0
Order_0000001,Pdt_01,222.8
Order_0000005,Pdt_04,150.0
Order_0000006,Pdt_03,410.2
Order_0000004,Pdt_02,120.5
Order_0000009,Pdt_05,88.8
Order_0000002,Pdt_04,122.4
Order_0000010,Pdt_08,77.7
Order_0000007,Pdt_06,89.0
Order_0000008,Pdt_03,260.0
```

**Bean**

- `compareTo()`メソッドは1つのパーティションのデータに並べ替えをする。
- 最後の結果は同じの注文IDに商品価格が一番高いのを取得するので、注文IDに従ってグループして価額の降順で並べ替える。

```
@Data
public class OrderBean implements WritableComparable<OrderBean> {

    private String orderId;
    private String pdtId;
    private Double price;

    @Override
    public int compareTo(OrderBean o) {
        //注目ID順序
        int i = this.orderId.compareTo(o.orderId);
        if (i == 0) {
            //注目ID同じなら価額で並べ替える
            i = -this.price.compareTo(o.price);
        }
        return i;
    }

    @Override
    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeUTF(orderId);
        dataOutput.writeUTF(pdtId);
        dataOutput.writeDouble(price);
    }

    @Override
    public void readFields(DataInput dataInput) throws IOException {
        this.orderId = dataInput.readUTF();
        this.pdtId = dataInput.readUTF();
        this.price = dataInput.readDouble();
    }

    @Override
    public String toString() {
        return orderId +"\t"+ pdtId +"\t"+ price;
    }
}
```

**Group**

- `GroupingComparator`役割は元々異なるKeyのデータを集約してReduceに渡す。
- `compare()`意味は変数Aと変数Bを比較して0を返したら同じKeyに認められて1つの集約に入れる。

```
public class OrderGroupingComparator extends WritableComparator {

    public OrderGroupingComparator() {
        super(OrderBean.class, true);
    }

    @Override
    public int compare(WritableComparable a, WritableComparable b) {
        OrderBean first = (OrderBean) a;
        OrderBean second = (OrderBean) b;
        final int i = first.getOrderId().compareTo(second.getOrderId());
        if (i == 0) {
            System.out.println(first.getOrderId() + "----" +
                    second.getOrderId());
        }
        return i;
    }
}
```

**Reduce**

- `reduce()`メソッド変数Keyは、同じKeyを持つ集約のKeyです。
- `GroupingComparator`追加したら、このデータ集約（グループ）の一番目のKeyを取って`reduce()`に渡す。
- このグループのデータはもう価額の降順で並べ替えたので、一番目のKeyはこの注文に価額の一番高い商品です。

```
public class GroupReducer extends Reducer<OrderBean, NullWritable, OrderBean, NullWritable> {

    @Override
    protected void reduce(OrderBean key, Iterable<NullWritable> values, Context
            context) throws IOException, InterruptedException {
        System.out.println(----reduce処理----);
        context.write(key, NullWritable.get());
    }
}
```

**Partition**

- OrderIDをパーティションの判断条件にする

```
public class OrderPartitioner extends Partitioner<OrderBean, NullWritable> {
    @Override
    public int getPartition(OrderBean orderBean, NullWritable nullWritable, int i) {
        return (orderBean.getOrderId().hashCode() & Integer.MAX_VALUE) % i;
    }
}
```

**Map**

- データの読み込みと実装

```
public class GroupMapper extends Mapper<LongWritable, Text, OrderBean, NullWritable> {

    @Override
    protected void map(LongWritable key, Text value, Context context) throws
            IOException, InterruptedException {
        final String[] arr = value.toString().split(",");
        final OrderBean bean = new OrderBean();
        bean.setOrderId(arr[0]);
        bean.setPdtId(arr[1]);
        bean.setPrice(Double.parseDouble(arr[2]));
        context.write(bean, NullWritable.get());
    }
}
```

**Driver**

```
public class GroupDriver {

    public static void main(String[] args) throws IOException,
            ClassNotFoundException, InterruptedException {
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);

        job.setJarByClass(GroupDriver.class);
        job.setMapperClass(GroupMapper.class);
        job.setReducerClass(GroupReducer.class);

        job.setMapOutputKeyClass(OrderBean.class);
        job.setMapOutputValueClass(NullWritable.class);
        job.setOutputKeyClass(OrderBean.class);
        job.setOutputValueClass(NullWritable.class);

        FileInputFormat.setInputPaths(job, new Path("D:\\order.txt"));
        FileOutputFormat.setOutputPath(job, new Path("D:\\output"));
        
        job.setPartitionerClass(OrderPartitioner.class);
        job.setGroupingComparatorClass(OrderGroupingComparator.class);
        
        job.setNumReduceTasks(2);
        
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
        boolean b = job.waitForCompletion(true);
        System.exit(b ? 0 : 1);
    }
}
```

**結果**

　全ての結果は2つファイルに分配される。同一注文に価額一番高い商品を記録する。

![image-20260428230455763](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260428230455763.png)

![image-20260428230517579](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260428230517579.png)

![image-20260428230536985](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260428230536985.png)

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

　MapReduceのデフォルトであるTextInputFormatは、 ファイル単位で入力を分割する仕組みになっている。 そのため、ファイルサイズに関係なく1ファイル＝1スプリットとして扱われ、 それぞれが1つのMapTaskで処理される。 もし小さなファイルが大量にある場合、 それに対応して大量のMapTaskが生成されることになる。 しかし、各MapTaskが処理するデータ量は少なく、 初期化・リソース確保・終了処理などに時間がかかるため、 結果としてリソース効率が悪くなる。

　CombineTextInputFormatは、小さなファイルが多いケース向けの仕組みで、 複数の小ファイルを論理的にまとめて 1つのスプリットとして扱うことができる。 これにより、複数ファイルを1つのMapTaskで処理でき、 リソース利用効率を向上させることができる。

```
//InputFormatを指定しない場合、デフォルトはTextInputFormatが使われる
job.setInputFormatClass(CombineTextInputFormat.class);

//仮想スプリットの最大サイズを4MBに設定
CombineTextInputFormat.setMaxInputSplitSize(job, 4194304);
```

　CombineTextInputFormatの分割原理は、主に次の2つの段階に分かれる： 仮想的な分割（論理分割） 実際の切片生成 。ここでは、仮にsetMaxInputSplitSize = 4MBとする。対象の小ファイル： 1.txt → 2MB 2.txt → 7MB 3.txt → 0.3MB 4.txt → 8.2MB。

　仮想分割のロジックは入力ディレクトリ内の各ファイルサイズを、設定した最大値（4MB）と比較して処理する。

- ファイルサイズが最大値以下の場合 → そのまま1つのブロックとする。
  - 1.txt（2MB） 2MB < 4MB -> そのまま 1ブロック 
  - 3.txt（0.3MB） 0.3MB < 4MB -> 1ブロック 

- ファイルサイズが最大値を超えるが、2倍以下の場合 -> 均等に2つに分割する（小さすぎる分割を防ぐため）。
  - 2.txt（7MB）4MB < 7MB ≤ 8MB -> 2つに均等分割 -> 3.5MB + 3.5MB 

- ファイルサイズが最大値の2倍以上 の場合 -> まず最大値で1ブロック切り出し、残りをさらに分割する。
  - 4.txt（8.2MB） 8.2MB > 8MB -> まず 4MBを1ブロック -> 残り4.2MBは2分割（小さすぎる分割を防ぐ） -> 2.1MB + 2.1MB

　最後に2M、3.5M、3.5M、0.3M、4M、2.1M、2.1M、7個の仮想分割が残っている。

　切片生成の流れは仮想的に分割されたデータのサイズが、設定したsetMaxInputSplitSize以上かどうかを判断する。

- 以上の場合：そのまま1つの切片として扱う。
- 未満の場合：次の仮想ブロックと結合して、1つの切片を作る。

　最後に(2+3.5)M，(3.5+0.3+4)M，(2.1+2.1)M、3つ切片となる。

　CombineTextInputFormatざっくり言うと、小さいファイルはまとめる、大きいファイルは分割し、極端に小さい断片は作らないように調整するという仕組み。

**カスタムInputFormat**

　HDFSやMapReduceは、小ファイルの処理があまり得意ではない。 そのため、小ファイルが大量にある場合は効率が悪くなる。 この問題への対応として、 InputFormatをカスタマイズして小ファイルをまとめて扱う方法がある。

　要件：複数の小ファイルを1つのSequenceFileにまとめる。 SequenceFileはHadoopで使われるkey-value形式のバイナリファイル。 < key：ファイルパス＋ファイル名, value：ファイルの中身>

#### 8.4.2　OutputFormat

　OutputFormatはMapReduceデータ出力の親クラス、どんなデータ出力はOutputFormatを実装することがある。

　常に使うOutputFormatの子クラス：

- TextOutputFormat：デフォルトの出力形式は `TextOutputFormat` で、各レコードを1行のテキストとして書き出す。KeyとVauleは任意の型でもよく、`TextOutputFormat` は `toString()` メソッドを使ってそれらを文字列に変換する。
- SequenceFileOutputFormat：圧縮されやすいフォーマットなので、`SequenceFileOutputFormat` を出力形式として使えば、後続の MapReduce タスクの入力として利用しやすい。

**カスタムOutputFormat**
