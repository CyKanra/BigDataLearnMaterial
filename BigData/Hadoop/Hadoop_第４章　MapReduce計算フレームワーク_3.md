# 分散大規模データ処理システム -- Hadoop-7

# 第４章　MapReduce計算フレームワーク-3

 MapReduceにのReduceTask運行原理について紹介する。

## 第７節　MapReduceの原理分析

### 7.2　ReduceTask運行仕組み

![image-20250604152300239](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250604152300239.png)

**Copy段階**

- MapTaskからデータを取得して先ずバッファに書き込む。一定閾値に達してまだディスクに転送する。Spillに似る流れである。
- MapTaskにのデータは同じの余りを持つ`<key：単語/value：単語数>`を集めてパーティションがあるデータです。なお、そのデータはMapTask流れにもう並び替えられてあった。
- 1つのReduceTaskは同じのパーティション番号のみを取得する。例えば、各MapTaskのpartition0はReduceTask1に集まる。ReduceTask数とMapTask数が同じです。

　ここにMapReduceにとって1つの重要な特性が見える。ロジックサイズを切片（spill）サイズに割ると複数のMapTaskをなすから、MapTaskにデータはMapTask数に割るとパーティションをなすまで、あと同じ番号のパーティションはReduceTaskに収集され、その流れにMapTask数、パーティション数、ReduceTask数は同じです。最初のMapTask一旦決まったら最後までその並行度で動いている。MapReduceの実行方案はその特性に合うかどうかは効率にかなり影響してくる。一方的に並行度を増加すると逆に効率的じゃなくなる。

　Hadoopに「分割」と「併合」概念はかなり重要で、その「分割」の仕組みは上記の流れに体現できる。

**Merge段階**

　Copy段階収集されるデータを併合する。

**Sort段階**

　併合したデータを`<key：単語, value：単語数>`のkey値で並び替える。