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

- Copy段階収集されるデータを統合する。例えば、各MapTaskから収集されるpartition0を1つ大きなpartition0になる。
- どうやってReduceTaskが自分に属するパーティションを見つける？それは各 MapTask は、出力結果を「分区（パーティション）」ごとにローカルファイルに保存しており、各ファイルは `map_output_[partitionID]` のような形式で保持されている。ReduceTaskは、そのファイルによって対応のアドレスからMaptask結果を取得する。

**Sort段階**

　統合されたデータを`<key：単語, value：単語数>`のkey値で並べ替える。

**Reduce段階**

　Reduce段階のロジックはReduceメソッドに書いているものです。もしReduceTask数は1で、全てのデータがReduceTaskに収まている。Reduceメソッドに入力するを準備する。

![image-20250612072933416](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250612072933416.png)

　一気に全体的なデータをReduceメソッドに渡すことじゃない。keyごとに1回ずつReducerメソッドが呼ばれ、`Iterable<value>`が渡されて処理する。最後にHDFSファイルに結果を出力する。

　ReduceTask数は1なので、HDFSにの結果ファイルはpart-r-00000だけである。

![image-20240820074437964](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074437964.png)

![image-20240820074843969](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074843969.png)

### 7.4　Shuffle仕組み

　Copy段階からReduceTaskへ転送する過程はShuffleと呼ばれる。それもMapTaskからReduceTaskにデータを転送する核心の流れです。

![image-20250621105021712](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250621105021712.png)

　MapTaskの`<key, value>`データは改めて仕切りを行われてあり、新しいパーティションを生成される。MapTaskとReduceTaskの仕切り処理は別々に考える。

　同じのkey値のデータを1つReduceTaskに入れたいなら、同じのパーティションに割り振られていい。パーティションとReduceTask数量が同じの場合、同じ番号持つパーティションは1つのReduceTaskに入れ、その動作はMapReduceにとっては黙認の行為です。

![image-20250628095029834](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250628095029834.png)

　上図はパーティションのソースコードで、key値をRedcueTask数に割ると相同的な余りを持つデータが同じのパーティション番号を返す。あと同じのパーティションに入れる。

### 7.5　カスタムパーティション

　実況にデフォルトのパーティション処理は全ての需要を満たせないので、そのためカスタム可能のは必要になる。公式は`Partitioner`クラスを継承し、`getPartition()`メソッドを実装するとカスタムになれる。

　下は設備の情報と使用時間のデータで、入力ファイル`partition.txt`格納される。実現結果は同じな接頭語の型番を1つファイルに出力する。

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

**需要分析：**

- 同じな接頭語データが3種があり、1種が1ファイルに格納するならReduceTask数はせめて3つが要る。
- カスタムあるので、`Partitioner`クラスの`getPartition()`メソッドを実装するのが必要です。
- データに対するBeanクラスを実装し、直列化Writableを継承する必要です。

**Mapper**

- 一行データvalueを取得し、Beanクラスに入れる。
- 二番目項目の前の部分を出力Keyとして渡す。パーティション処理に銘柄によって異なるファイトに入れることがある。

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

- 何も書かなくていい。
- Mapperに出力Keyを巡って集合処理がないので、Mapper`<key, partitionBean>`形式のままでファイトに出力する。

**Bean**

- Mapperの出力値としてBeanクラスがWritableを継承する必要です。
- `write`と`readFields`二つメソッドを書き直す。
- 出力ファイルのデータ形式は書き直した`toString()`で決まる。

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

- Partitionerを継承して`getPartition()`メソッドを書き直す。
- 銘柄によって3つのパーティションに区切って戻す。

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

- Redcuerの部分がないので、Redcuerクラス導入と出力値タイプの設定を省略できる。
- `job.setNumReduceTasks(3)`ReduceTask数を3つにしてパーティション数と一致にさせる。

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

　最後の出力結果は上図のように表示する。同じ銘柄のデータが同じなファイルに入れる。Reducer工程がないので、出力形式はMapper出力の様子です。

　もし最初のKey値を除きたい、Reducerがこうのように書ける。

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

　reduceメソッドの入力変数は同じKeyを持つの反復子です。1つずつ取り出し、nullにして出力する。

　勿論、PartitionDriverにコメントアウトされた部分を解ける。

```
job.setReducerClass(PartitionReducer.class);
job.setOutputKeyClass(NullWritable.class);
job.setOutputValueClass(PartitionBean.class);
```

![image-20250721103049942](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721103049942.png)

　結果はこうになる。もし縦棒を外したいならPartitionBeanクラスの`toString()`を書き直す。

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

　Reduceは統合の工程ですけど、並行度の概念もある。MapTaskと切片サイズで決まると違い、ReduceTask数量は手動的に設定できる。

```
 #3に設定する
 job.setNumReduceTasks(3);
```

-  ReduceTask=0場合は、 Reduce段階がないと表示する。出力ファイル数とMapTask数が一致になっている。
-  ReduceTask=1はデフォルト値、出力ファイルは1つだけ。
-  ReduceTask数決まったら、パーティション数もReduceTask数に関わって決められる。

　ただ、先カスタムパーティション案例は、パーティション数が手動的に設定できる。その場合、パーティション数がReduceTask数と異なるなら何が起こる。

**ReduceTask=0場合：**

　Reduce処理は完全にスキップされ、Mapの出力がそのまま最終出力になる。でもデータサイズが128Mにまだ至らないので、1つMapTasks数だけで最後に1つファイルに出力する。

![image-20250721173733683](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721173733683.png)

![image-20250721173752371](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721173752371.png)

**ReduceTask=1場合：**

　全てのデータを纏めて1つファイルに出力する。結果はReduceTask=0と同じです。

**ReduceTask=2場合：**

　エラーが起こってある。上記の場合を除いてパーティション数より小さくするならプログラムが運行できない。

![image-20250721174226014](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721174226014.png)

**ReduceTask>3場合：**

　複数の空きファイルが出る。

![image-20250721174750484](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250721174750484.png)



ここまで本章の講解が終わりです。

次の講解はMapReduceについて幾つか特性を紹介します。

よろしくお願いいたします。
