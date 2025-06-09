# 分散大規模データ処理システム -- Hadoop-7

# 第４章　MapReduce計算フレームワーク-3

 MapReduceにのReduceTask運行原理について紹介する。

## 第７節　MapReduceの原理分析

### 7.2　ReduceTask運行仕組み

![image-20250604152300239](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250604152300239.png)

**Copy段階**

- MapTaskからデータを取得して先ずバッファに書き込んでおく。一定閾値に達してまだディスクに転送する。Spillに似る流れである。
- MapTaskにのデータは同じの余りを持つ`<key：単語/value：単語数>`を集めるパーティションであるデータです。なお、そのデータはMapTask流れにもうkeyで並び替えられてあった。
- 図にのpartition0、partition1はReduceTask数で決まっている。そのReduceTask数はコード層に設定できる。ロジックは<key：単語/value：単語数>をReduceTask数に割ると同じ余りを持つの値で仕切られるのが一様ですけど、MapTaskとは異なる流れです。

- 上図のようにReduceTask数とMapTask数を一致にする場合、MapTaskのパーティション結果とReduceTaskのパーティション結果が同じになる。各MapTaskにのpartition0が何も変わってなく全てReduceTask1に入れる。

- デフォールトでReduceTask数は1です。その場合、partition0だけあって全てReduceTaskに入れる。

**Merge段階**

　Copy段階収集されるデータを併合する。

**Sort段階**

　併合したデータを`<key：単語, value：単語数>`のkey値で並び替える。

**Reduce段階**

