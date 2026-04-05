# ビッグデータ高速計算エンジンSpark-2

# 第２章　Sparkの分散システム組み立て

## 第２節　Sparkの部署モード

　Apache Sparkには、いくつかの実行モードがある。

**ローカルモード**

　最も簡単なモードで、すべての処理が1台のマシンのJVM上で実行される。

**疑似分散モード**

　1台のマシン上でクラスター環境を擬似的に再現するモード。関連するプロセスもすべて同じマシン上で動く。

**分散モード**

　実際のクラスター上で動作するモード。主に次の3種類がある：

- Standalone：Spark標準のリソース管理を使用
- YARN：YARNのリソース管理を使用
- Mesos：Mesosのリソース管理を使用

### 2.1　ローカルモード

　ローカルモードは単一マシン上で動作し、Sparkクラスタを依存する必要がない。主にテストや検証用に使われる。最も簡単な実行方式で、すべての処理は1台のマシンのJVM内で行われる。

　このモードでは、1台のマシン上の複数スレッドを使って分散処理を疑似的に再現する。そのため、開発したアプリのロジック確認によく使われる。

　設定も簡単で、Sparkを解凍して基本設定を少し調整するだけで使える。MasterやWorkerの起動は不要で、Hadoopサービスも基本的には必要ない（HDFSを使う場合を除く）。

ローカルモードの起動オプション：

- local：1スレッドで実行
- local[N]：N個のスレッドで実行
- local[\*]：CPUの全コアを使用
- local[N, M]：Nは使用するコア数、Mはタスクの最大失敗許容回数

※ Mを指定しない場合は、デフォルトで1回となる。

- HadoopとSparkサービス停止

```
#Hadoop停止
stop-dfs.sh

#Sparkサービス停止
#もしHadoopのstop-all.sh名称を変わったら直接に実行できる
stop-all.sh

#検証
jps
```

![image-20260405115026196](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405115026196.png)

- Sparkローカルモード起動

```
spark-shell --master local[*]
```

![image-20260405111644720](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405111644720.png)

　起動が失敗した。原因はspark-defaults.confファイルにSparkのログがHDFSに格納されるのが設定され、HDFSサービスを見つけられないでエラーが発生した。

![image-20260405112123189](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405112123189.png)

　解決方法は下の2行を註解にしてHDFSを使わなくさせる。成功したらSparkSubmitスレッドが現れる。

```
#spark.eventLog.enabled           true
#spark.eventLog.dir               hdfs://centos1:9000/spark-eventlog
```

![image-20260405114658527](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405114658527.png)

![image-20260405114735997](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405114735997.png)

![image-20260405115052991](D:\OneDrive\picture\Typora\BigData\Spark\image-20260405115052991.png)

### 2.2　疑似分散モード

　疑似分散モードは、1台のマシン上でクラスター環境を擬似的に再現するモード。関連するプロセスもすべて同じマシン上で動作する。

※ クラスター用のリソース管理サービスを起動する必要はない。

**起動形式**

- `local-cluster[N, cores, memory]`
  - **N**：仮想的に用意するWorker（Slave）ノードの数
  - **cores**：各Workerが持つCPUコア数
  - **memory**：各Workerに割り当てるメモリ容量
