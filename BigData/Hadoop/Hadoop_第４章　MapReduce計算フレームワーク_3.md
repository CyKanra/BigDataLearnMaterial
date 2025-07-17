# 分散大規模データ処理システム -- Hadoop-7

# 第４章　MapReduce計算フレームワーク-3

 MapReduceにのReduceTask運行原理について紹介する。

## 第７節　MapReduceの原理分析

### 7.2　ReduceTask運行仕組み

![image-20250604152300239](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250604152300239.png)

**Copy段階**

- MapTaskからデータを取得して先ずバッファに書き込んでおく。一定閾値に達してまだディスクに転送する。Spillに似る流れである。
- MapTaskにのデータは同じの余りを持つ`<key：単語/value：単語数>`を集めるパーティションであるデータです。なお、そのデータはMapTask流れにもうkeyで並べ替えられてあった。
- ただ、図にのpartition0、partition1はMapTask数で決まっているじゃなく、ReduceTask数によって生まれる。ここMapTaskからパーティション状態を保って直接にReduceTaskに転送すると誤解しやすい

- ReduceTask数はコード層に設定でき、MapTask数とは関係ない。ロジックは<key：単語/value：単語数>をReduceTask数に割り、同じ余りを持つの値で仕切られて同じパーティションに入れると類似けど、実はMapTaskとは2つの流れです。


```
 job.setNumReduceTasks(3);
```

- 上図のようにReduceTask数とMapTask数を一致にする場合、MapTaskのパーティション結果とReduceTaskのパーティション結果が同じになる。どっちがkey値をTask数に割ると余りによってパーティションを生成する。同じパーティションが1つのReduceTaskに入れる。例えば、図の表示するように各MapTaskにのpartition0が全てReduceTask1に入れる。partition1はReduceTask2に入れる。

- デフォールトでReduceTask数は1です。その場合、全てのデータがpartition0に属し、ReduceTaskに入れる。

**Merge段階**

　Copy段階収集されるデータを統合する。例えば、各MapTaskから収集されるpartition0を1つ大きなpartition0になる。

**Sort段階**

　統合されたデータを`<key：単語, value：単語数>`のkey値で並べ替える。

**Reduce段階**

　Reduce段階のロジックはReduceメソッドに書いているものです。もしReduceTask数は1で、全てのデータがReduceTaskに収まている。Reduceメソッドに入力するを準備する。

![image-20250612072933416](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250612072933416.png)

　一気に全体的なデータをReduceメソッドに渡すことじゃない。keyごとに1回ずつReducerメソッドが呼ばれ、`Iterable<value>`が渡されて処理する。最後にHDFSファイルに結果を出力する。

　ReduceTask数は1なので、HDFSにの結果ファイルはpart-r-00000だけである。

![image-20240820074437964](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074437964.png)

![image-20240820074843969](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074843969.png)

### 7.3　ReduceTask並行度

　Reduce統合の流れに並行度の概念もある。MapTaskの並行度は切片で決まると違い、ReduceTask数量は手動的に設定できる。

```
 #3に設定する
 job.setNumReduceTasks(3);
```

-  ReduceTask=0場合は、 Reduce段階がないと表示する。出力ファイル数とMapTask数が一致になっている。
- ReduceTask=1はデフォルト値、出力ファイルは1つだけ。上の案例は1つファイルを出力する。

### 7.4　Shuffle仕組み

　Copy段階からReduceTaskへ転送する過程はShuffleと呼ばれる。それもMapTaskからReduceTaskにデータを転送する核心の流れです。

![image-20250621105021712](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250621105021712.png)

　MapTaskの`<key, value>`データは改めて仕切りを行われてあり、新しいパーティションを生成される。MapTaskとReduceTaskの仕切り処理は別々に考える。

　同じのkey値のデータを同じのReduceTaskに入れたいなら、同じのパーティションに割り振られていい。同じ番号持つパーティションは同じのReduceTaskに入れ、その動作はMapReduceにとっては黙認の行為です。

![image-20250628095029834](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250628095029834.png)

　上図はパーティションのソースコードで、key値をRedcueTask数に割ると相同的な余りを持つデータが同じのパーティション番号を返す。あと同じのパーティションに入れる。

### 7.5　カスタムパーティション

　実況にデフォルトのパーティション処理は全ての需要を満たせないので、そのためカスタム可能のは必要になる。公式は`Partitioner`クラスを継承し、`getPartition()`メソッドを実装するとカスタムになれる。

　下は設備の情報と使用時間のデータです。実現結果は同じな型番の接頭語を1つファイルに出力する。

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

需要分析：

- 同じな接頭語データが3種があり、1種が1ファイルに対するならReduceTask数も3つが要る。
- パーティションとReduceTaskが一対一なので、パーティション数をコントロールするだけでいい。
- カスタムあるので、`Partitioner`クラスの`getPartition()`メソッドを実装するのが必要です。
- データに対するBeanクラスを実装し、直列化Writableを継承する必要です。

**Mapper**

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

