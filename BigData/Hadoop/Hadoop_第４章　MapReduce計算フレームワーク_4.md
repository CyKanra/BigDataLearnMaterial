# 分散大規模データ処理システム -- Hadoop-7

# 第４章　MapReduce計算フレームワーク-4

　MapReduce特性について紹介する。

## 第８節　MapReduce特性の紹介

### 8.1　MapReduce中のCombiner

　Combinerとは何、簡単に説明するならReducerと同じのものです。区別は実行する所が違う。

![image-20250728154715676](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250728154715676.png)

- CombinerはMapperとReducer以外のMRコンポーネントの一つです。
- Combinerの親クラスはReducerで、それは何でReducerとはほとんど同じ。
- Combiner実行の位置は各節点に各MapTaskにSpill階段とMerge階段の間にいる。Reducerは任意のDateNodeに、それぞれの受け取るデータを集合する。
- Combiner役割はMapTaskについて局部的に集合を行い、
