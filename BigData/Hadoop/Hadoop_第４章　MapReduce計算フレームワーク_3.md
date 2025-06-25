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

　Copy段階からReduceTaskへ転送する過程はShuffleです。それもMapTaskからReduceTaskにデータを転送する核心の流れです。

![image-20250621105021712](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250621105021712.png)

　MapTaskの`<key, value>`データを改めて仕切りを行い、新しいパーティションを生成される。次、同じ番号持つパーティションは同じのReduceTaskに入れる。

　同じのkey値のデータを同じのReduceTaskに入れたいなら、同じのパーティションに割り振られていい。ただ、実はその動作はMapReduceにとっては黙認の行為です。

![image-20250621112951891](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250621112951891.png)

　上図はパーティションのソースコード、key値をRedcueTask数に割ると相同的な余りを持つデータが同じのパーティション番号を返す。あと同じのパーティションに入れる。
