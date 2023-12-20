# HBase-分散型大規模非関係データベース -3

## 第３章　HBase全体の粋組み

　　本章節は主にHBaseの全体の粋組みを紹介し、各部品役割を担当する内容を説明する。

![image-20231207065141414](D:\OneDrive\picture\Typora\image-20231207065141414.png)

**Client** 

　　はHBaseに訪問するのインタフェースを含み、外部の請求を受けてHBaseの内部に訪問する。高速化するために幾つか情報をキャッシュに維持している。例えば、regionの位置情報などがある。

**Zookeeper**

　　HBaseは内蔵のZookeeperを使用することも、外部のZookeeperを使用することもできます。実際の現行環境では、一致性を保証するために通常外部Zookeeperが使用される。ZookeeperがHBaseで果たす役割は以下の通り：

- いつでもクラスタ内には1つのmasterがあると保証し、システムの高信頼性を実現した。
- 全てのRegionのアドレス情報を保存し、それはHBaseのメタデータです。

![image-20231211153406825](D:\OneDrive\picture\Typora\image-20231211153406825.png)

- そのため、何のデータの操作にはZookeeperを通して具体的な位置を取得しており、HBase自身がメーターデータを保存することがない。
- Region Serverの状態を監視してリアルタイムでMasterに通知し、一貫性を保証してある。

**HMaster（Master）**

　　整体のクラスター正常の運行を維持する役を担当する。

- HMasterはデータのメタデータ情報を管理する。
- Region ServerにRegionを分配してあげ、無効になったRegion Serverを発見し、その対応のRegionを再分配してある。
- 適当にRegionを分配を通してRegion Serverの負荷分散の効果を実現でき、例えば、あるRegion Serverが負荷が重すぎ、幾つかRegionを他のサーバに転移してある。
- HDFS上のゴミファイルの回収する。

**HRegion Server（Region Server）**

　　実際に働くの役と、最終にデータを読み書く操作はHRegion Serverが担当する。

- 割り当てられたRegionを維持し、あるRegion過大になった場合、そのRegionを分割する責任を担当する。
- データの読み書き流れは、ある時HMasterの参加は必要ない、Clientキャッシュからデータのアドレスを取得して直接に請求をHRegion Serverに発送する。

**HRegion（Region）**

　　データを格納する場所です。

![image-20231211150746303](D:\OneDrive\picture\Typora\image-20231211150746303.png)

- 各HRegionは複数のStoreで構成され、一つのStoreは1つの列族を保存する。表が複数の列族を持つ場合、それに対応して同数のStoreが存在する。
- 各Storeは1つのMemStoreと複数のStoreFileで構成されてある。MemStore内容はStoreの内容をメモリーに格納してる部分、ファイルに書き込まれるとStoreFile内容となる。StoreFileのデータはHFile形式で保存される。