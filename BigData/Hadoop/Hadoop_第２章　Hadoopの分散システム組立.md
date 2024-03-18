# 分散大規模データ処理システム -- Hadoop-2

## 第２章　Hadoopの分散システム組み立て

**バージョン情報**

- HadoopはJava言語で書かれるため、Java環境（JVM）が必要で、 JDKバージョン：JDK8。
- 統一的にVMwareの仮想マシンを使用してLinuxの４つの節点を準備しておき、LinuxOS：Centos7。
- Hadoopの構築モードの選べ
  - 単一節点モード：単一節点モードは非クラスタ、本番環境ではこの方法は使用されません。
  - 単一節点模擬分布モード：単一節点、マルチスレッド（multi-thread）でクラスタの効果を模擬し、本番環境ではこの方法は使用されません。
  - **完全分散モード**：複数節点を持ち、実際の生産環境のモードと一致で、本番環境ではこのモードが推奨されます。

### 第１節　環境準備

**クラスタ環境**

　　Hadoopの分散システム組立の前にクラスタの環境設定が必要で、ここで大体の要点を紹介し、詳しい設定手続きが「」文章に移って了解できます。

- クラスタは全て4つのサービスを備えてあり、hostファイルにIPアドレスのマッピングが図のように示します。

![image-20240212154807536](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212154807536.png)

- firewall／selinux等の動きが停止です。

```
systemctl status firewalld
```

![image-20240212160400972](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212160400972.png)

- 4つのサーバの間にパスワード不要で互いに登録して通信することができます。

- 4つのサーバの時刻が一致に保証します。

**Hadoopの分配**

| 対象 | centos1           | centos2                      | centos3     | centos4     |
| ---- | ----------------- | ---------------------------- | ----------- | ----------- |
| HDFS | NameNode,DataNode | SecondaryNameNode,DataNode   | DataNode    | DataNode    |
| YARN | NodeManager       | NodeManager、ResourceManager | NodeManager | NodeManager |

### 第２節　Hadoopの組み立て
