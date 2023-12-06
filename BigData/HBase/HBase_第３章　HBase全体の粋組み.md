# HBase-分散型大規模非関係データベース -3

## 第３章　HBase全体の粋組み

![image-20231207065141414](D:\OneDrive\picture\Typora\image-20231207065141414.png)

  　　本章節は主にHBaseの全体の粋組みを紹介し、各部品役割を担当する内容を説明する。

**Client** 

　　ClientはHBaseに訪問するのインタフェースを含み、外部の請求を受けてHBaseの内部に訪問する。高速化するために幾つか情報をキャッシュに維持している。例えば、regionの位置情報などがある。

**Zookeeper**

HBaseは内蔵のZookeeperを使用することも、外部のZookeeperを使用することもできます。実際の現行環境では、一致性を保証するために通常外部Zookeeperが使用される。ZookeeperがHBaseで果たす役割は以下の通り：

- いつでもクラスタ内には1つのmasterがあり、全てのRegionのアドレスの情報を保存する。
- Region Serverの状態を監視してリアルタイムでMasterに通知する。

**HMaster**

- Region ServerにRegionを割り当てる
- Region Serverの負荷分散を担当
- 無効になったRegion Serverを発見し、その上のRegionを再割り当て
- HDFS上のゴミファイルの回収
- スキーマの更新リクエストを処理

**HRegion Server**

- HMasterによって割り当てられたRegionを維持し、これらのRegionに対するIOリクエストを処理
- Regionがランタイムで過大になった場合、そのRegionを分割する責任を担当

クライアントがHBase上のデータにアクセスするプロセスでは、HMasterの参加は必要ありません（ZookeeperおよびHRegion Serverへのアクセス、データの読み書きはHRegion Serverが担当）。HMasterは単にtableとHRegionのメタデータ情報を維持し、負荷が非常に低いです。