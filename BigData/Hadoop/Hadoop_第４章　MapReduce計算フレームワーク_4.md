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

### 8.2　MapReduceの並び替え
