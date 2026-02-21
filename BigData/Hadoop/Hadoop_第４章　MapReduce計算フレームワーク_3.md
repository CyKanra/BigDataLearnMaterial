# 分散大規模データ処理システム -- Hadoop-7

# 第４章　MapReduce計算フレームワーク-3

 MapReduceにのReduceTask運行原理について紹介する。

## 第７節　MapReduceの原理分析

### 7.3　ReduceTask運行仕組み

![image-20250604152300239](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250604152300239.png)

**Copy段階**

- Copy段階とは、MapTask側で最終生成された統合済みファイルを、ReduceTask側で取得する処理を指す。
- ここ誤解しやすい点がある。図では、partition0、partition1のパーティション数、MapTask数、ReduceTask数が同じに描かれているため、何らか対応関係があるように見えるかも。しかし、実際には必ずしも同じである必要はない。
- 1つノード（node）で複数のMapTaskが実行されることがある。それはブロック（block）が切片サイズ（split size）に分割され、その数に応じてMapTaskが生成される。
- ReduceTaskの数はコードレベルに設定できる。切片サイズとは関係ない。


```
 job.setNumReduceTasks(3);
```

- ただし、パーティション数はReduceTask数と一致する。パーティションはReduceTask数を基準に生成されるため、最終的には各 ReduceTaskにちょうど1つずつ割り当てられる。

```
partition = hash(key) % numReducers
```

- デフォールトでReduceTask数は1に設定されている。その場合、全てのデータはpartition0に属て１つReduceTaskに集約される。

**Merge段階**

- Copy段階で収集されたデータは、その後に統合する。各MapTaskから同じパーティション番号（例：partition0）に属するデータを取得し、それらを1つに統合する。

**Sort段階**

- 統合と同時に、データは `<key: 単語, value: 出現回数>` の key 値に基づいて並べ替えられる。

**Reduce段階**

- Reduce段階の処理ロジックは、`reduce()` メソッド内に実装される。最終的な計算結果は、このメソッドで生成される。


![image-20250612072933416](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250612072933416.png)

　ReduceTask数は1つ場合で、すべてのデータが一度に`reduce()`に渡されるわけではない。`reduce()` メソッドがkeyごとに1回ずつ呼び出され、対応する値の集合（Iterable）が渡される。

　また、出力ファイルの数はReduceTaskの数と一致するため、HDFS上には `part-r-00000` のみが最終結果ファイルとして生成される。

![image-20240820074437964](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074437964.png)

![image-20240820074843969](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074843969.png)

### 7.4　Shuffle仕組み

　Copy段階からReduceTaskへ転送する一連の処理は、Shuffleと呼ばれる。これは、MapTaskからReduceTaskへ中間データを受け渡す際の中核となるプロセスです。

![image-20250621105021712](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250621105021712.png)

　同じkey値を持つデータは、同じパーティションに振り分けられる。そして、同じ番号のパーティションは1つの ReduceTask に割り当てられ、最終的に1つの出力ファイルとして生成される。

　この特性を利用すれば、特定のデータを同じファイルに纏めたい場合には、同じkey値を付与すればよいということになる。

![image-20250628095029834](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250628095029834.png)

　上図は、パーティション計算のソースコードを示している。key 値、ReduceTask 数、そしてパーティション番号の三者は相互に関連している。key値とReduceTask 数を適切に制御することで、データをどのReduceTaskに振り分けるかを決定できる。

### 7.5　カスタムパーティション

　実際の運用では、デフォルトのパーティション処理だけではすべての要件を満たせない場合がある。そのため、カスタムパーティションの実装が必要になることがある。

　Hadoopでは、`Partitioner`クラスを継承し、`getPartition()`メソッドを実装することで、独自のパーティション処理を定義できる。

　以下は、設備の情報と使用時間を記録したデータで、入力ファイル `partition.txt` に格納されている。

　本実装の目的は、同じ接頭語を持つ型番を同一のファイルに出力することです。

```
#設備id　型番　ネットIP　使用時間
0a8cd1 kar_909011 149.187.152.169 4054
0ce73e nex_334455 181.133.189.160 8938
034347 nex_334455 157.86.7.229 821
0cdf27 kar_992134 233.205.58.168 9709
0b66c9 lun_553322 236.9.100.245 9921
0c35e3 lun_999999 49.179.180.211 5672
0b6b5d lun_786544 114.179.97.23 7648
01b85a nex_909123 131.200.122.190 1863
```

**要件分析：**

- 同じ接頭語を持つデータは3種類あるため、それぞれを1つのファイルに出力するには、ReduceTaskは少なくとも3つ必要です。
- パーティションを独自に制御するため、`Partitioner` クラスを継承し、`getPartition()` メソッドを実装する必要がある。
- また、入力データを扱うためのBeanクラスを作成し、直列化のために `Writable` インターフェースを実装する必要がある。

**Mapper**

- 1行分のデータ（value）を取得し、Bean クラスに格納する。
- 2番目の項目の接頭部分を出力keyとして設定する。key値ごとに異なるファイルへ振り分けられるになる。

```
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import java.io.IOException;

public class PartitionMapper extends Mapper<LongWritable, Text, Text, PartitionBean> {

    Text model = new Text();
    @Override
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, PartitionBean>.Context context) throws IOException, InterruptedException {
        //一行文字を取得
        String str = value.toString();
        
        //文字を区切って項目リストになる
        String[] strs = str.split(" ");
        
        //型番項目の区切り、入力Keyとして
        String[] modelStr = strs[1].split("_");
        String modelKey = modelStr[0];
        
        //実体Beanクラスの実装
        PartitionBean partitionBean = new PartitionBean();
        partitionBean.setId(strs[0]);
        partitionBean.setModel(strs[1]);
        partitionBean.setNetIp(strs[2]);
        partitionBean.setUsageTime(strs[3]);
        
        //出力
        model.set(modelKey);
        context.write(model, partitionBean);
    }
}
```

**Reducer**

- 特別な処理は要らない。
- 今回は、Mapper 側で同一keyに対する集計処理を行わないため、`<key, partitionBean>`形式のままでファイルに出力する。

**Bean**

- Mapperの出力値とするBeanクラスは`Writable`インターフェースを実装する必要です。
- `write()`と`readFields()`2つメソッドを書き直し、直列化および逆直列化の処理を実装する。
- 出力ファイルのデータ形式は上書きの`toString()`メソッドの内容によって決まる。

```
import org.apache.hadoop.io.Writable;
import java.io.DataInput;
import java.io.DataOutput;
import java.io.IOException;

public class PartitionBean implements Writable {

    private String id;         // 設備id
    private String model;      // 型番
    private String netIp;      // ネットIP
    private String usageTime;     // 使用時間（分など）

    //セッター
    public void setId(String id) {
        this.id = id;
    }

    public void setModel(String model) {
        this.model = model;
    }

    public void setNetIp(String netIp) {
        this.netIp = netIp;
    }

    public void setUsageTime(String usageTime) {
        this.usageTime = usageTime;
    }

    @Override
    public String toString() {
        return id + " | " +
                model + " | " +
                netIp + " | " +
                usageTime;
    }

    @Override
    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeUTF(id);
        dataOutput.writeUTF(model);
        dataOutput.writeUTF(netIp);
        dataOutput.writeUTF(usageTime);
    }

    @Override
    public void readFields(DataInput dataInput) throws IOException {
        this.id = dataInput.readUTF();
        this.model = dataInput.readUTF();
        this.netIp = dataInput.readUTF();
        this.usageTime = dataInput.readUTF();
    }
}
```

**Partitioner**

- Partitionerを継承し、`getPartition()`メソッドを書き直す。
- keyの種類によって3つのパーティションに振り分けるよう実装する。

```
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Partitioner;

public class CustomPartitioner extends Partitioner<Text,PartitionBean> {
    @Override
    public int getPartition(Text text, PartitionBean partitionBean, int numPartitions) {
        int partition=0;
        final String appkey = text.toString();
        if(appkey.equals("kar")){
            partition=1;
        } else if (appkey.equals("nex")){
            partition=2;
        } else {
            partition=0;
        }
        return partition;
    }
 }
```

**Driver**

- Reducer処理を実装していないため、独自の Reducer クラスの指定や出力値の型設定は省略できる。
- `job.setNumReduceTasks(3)`ReduceTask数を3に指定してパーティション数と一致にさせる。

```
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;
import java.io.IOException;

public class PartitionDriver {

    public static void main(String[] args) throws IOException, InterruptedException, ClassNotFoundException {
        //配置ファイルを設定とJobを作成
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration, "PartitionDriver");
        //Mapper、Reducer、Driverクラスを添加
        job.setJarByClass(PartitionDriver.class);
        job.setMapperClass(PartitionMapper.class);
//        job.setReducerClass(PartitionReducer.class);
		job.setPartitionerClass(CustomPartitioner.class);
        //Mapの出力値のタイプ
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(PartitionBean.class);
        //最終出力値のタイプ
//        job.setOutputKeyClass(NullWritable.class);
//        job.setOutputValueClass(PartitionBean.class);
		//ReduceTask数設定
        job.setNumReduceTasks(3);
        //入力と出力ファイルのアドレス
        FileInputFormat.setInputPaths(job, new Path("D:\\partition.txt"));
        FileOutputFormat.setOutputPath(job, new Path("D:\\output"));
        //タスクをコミット
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

**結果：**

![image-20250721090655645](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721090655645.png)

![image-20250721090749595](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721090749595.png)

![image-20250721090825687](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721090825687.png)

　最終的な出力結果は、上図のとおりです。同じ key 値を持つデータは、同一のファイルにまとめて出力される。今回はReducer処理を行っていないため、出力データの形式はMapperの出力内容と同じになる。

　もし最初Keyが不要で、元のデータ構造をそのまま保持したい場合は、Reducerを下のように実装できる

```
import org.apache.hadoop.io.NullWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;
import java.io.IOException;

public class PartitionReducer extends Reducer<Text, PartitionBean, NullWritable, PartitionBean> {

    @Override
    protected void reduce(Text key, Iterable<PartitionBean> values, Reducer<Text, PartitionBean, NullWritable, PartitionBean>.Context context) throws IOException, InterruptedException {
        for (PartitionBean bean : values) {
            context.write(NullWritable.get(), bean);
        }
    }
}
```

　`reduce()` メソッドの入力引数は、同じkeyを持つ値の集合（Iterable）です。それらを1つずつ取り出し、必要に応じて再構成して出力する。

　なお、`PartitionDriver` でコメントアウトしている部分は有効化する必要がある。

```
job.setReducerClass(PartitionReducer.class);
job.setOutputKeyClass(NullWritable.class);
job.setOutputValueClass(PartitionBean.class);
```

![image-20250721103049942](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721103049942.png)

　結果はこうになる。

　出力形式を変更したい場合は、PartitionBeanクラスの`toString()`メソッドを書き直してできる。

```
@Override
public String toString() {
    return id + " " +
            model + " " +
            netIp + " " +
            usageTime;
}
```

### 7.5　ReduceTask並行度

　Reduceは統合の工程ですが、並行度という概念も存在する。MapTaskの並行度が切片サイズで決まるのと異なり、ReduceTaskの数は手動で設定することができる。

```
 #3に設定する
 job.setNumReduceTasks(3);
```

-  ReduceTask=0場合は Reduces過程がないで、パーティション処理もない。出力ファイル数はMapTask数と一致する。
-  ReduceTask=1はデフォルト値であり、出力ファイルは1つだけ生成される。
-  ReduceTask数決まったら、パーティション数はReduceTask数に繋がって決定する。

　先ほどカスタムパーティション案例は、パーティションの振り分けロジックを手動で定義している。もし、パーティション数がReduceTask数と一致しない場合、何が起こる。

**ReduceTask=0場合：**

　Reduce処理は完全にスキップされ、Mapの出力がそのまま最終結果として出力される。

　ただし、入力データサイズが128M に満たないので、MapTask は1つだけ生成されるため、最終的な出力ファイルも1つになる。

![image-20250721173733683](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721173733683.png)

![image-20250721173752371](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721173752371.png)

**ReduceTask=1場合：**

　全てのデータを纏めて1つファイルに出力する。

**ReduceTask=2場合：**

　この場合はエラー発生する。（ReduceTask=1のケースを除き）ReduceTask数をパーティション数より小さく設定すると、プログラムは正常に実行できない。

![image-20250721174226014](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721174226014.png)

**ReduceTask>3場合：**

　複数の空きファイルが生成されている。

![image-20250721174750484](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721174750484.png)

　手動でパーティションの振り分けロジックを定義してパーティション数もう完全にReduceTask数で決定されなくなった。そのため、ReduceTask 数を適切に設定し、パーティション数と一致させるよう注意する必要がある。



ここまで本章の講解が終わりです。

次の講解はMapReduceについて幾つか特性を紹介します。

よろしくお願いいたします。
