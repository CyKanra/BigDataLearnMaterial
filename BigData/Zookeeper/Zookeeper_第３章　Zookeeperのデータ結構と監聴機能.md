# 分散型協調サービスフレームワーク -- Zookeeper-3

## 第３章　Zookeeperのデータ結構と機構

### 第１節　Zookeeperのデータ結構

　　ZooKeeperでは、データ情報は個々のデータ節点（node）に保存され、これらの節点はZNodeと呼ばれる。ZNodeはZooKeeper内での最小のデータ単位であり、ZNodeの下にまだznodeを配置し続ける。こないうにして階層構造を形成されるの命名空間がZNode Treeと呼ばれる。ZNode Treeは、伝統のファイルシステムに似た階層の木構造を使用して管理される。

![image-20231011152002194](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231011152002194.png)

　　ZooKeeper中に、毎データ節点がZNodeだと言われる、例えば、根目録（root directory）の下には2つの節点があり、それぞれふが「/app1」と「app2」です。「app1」の下には更に3つの子節点がある。全てのZNodeは階層的に構成され、このような木構造が形成された。ZNodeの節点経路（path）は、Linuxファイルシステムの経路と非常に似ており、斜線（/）で区切られたで表してくる。開発者はこの節点にデータを書き込んだり、節点の下にサ子節点を作成したりすることができる。

### 第２節　 ZNodeの類型

　　まず、ZooKeeperのznode treeがデータ節点から成り立っていることを理解した。次に、節点について詳しく説明しよう。ZooKeeperの節点は主に3つの類型に分類される：

- **持久節点（Persistent）**：これはZooKeeperで最も一般的な節点類型で、節点が作成された後、サーバー上に永続的に存在し、削除操作が実行されるまで保留してある。

- **臨時節点（Ephemeral）**: 臨時節点は自動的に削除される節点で、クライアント会話と結びついています。クライアント会話が終了すると、ノードは削除される。臨時節点は子節点を作成できない。

- **順序節点（Sequential）**: 順序節点は、節点名の後ろに数字の接尾辞を付けて作成される節点です。これにより、節点が作成された順序が保持される。

　　ZooKeeperのトランザクションには、節点データ更新など操作だけではない、ZooKeeperサーバーの状態を変更する操作を指し、節点の作成、削除、内容の更新なども含む。各トランザクション請求には、唯一の全局トランザクションID（ZXID）が割り当てられ、通常は64ビットの数字で表される。各ZXIDは一回の更新操作を示し、これらのZXIDからZooKeeperが更新操作請求を処理した順序を識別することができる。

### 第３節　 ZNodeの状態情報

　　zookeeper-3.4.14/bin目録に入ってzkCli.shを執行する。

![image-20231012152605729](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231012152605729.png)

```
# zookeeper訪問画面にZNodeの状態情報を検査
get /zookeeper
```

![image-20231012153124019](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231012153124019.png)

　　全ての節点情報が節点状態とメモリー内容の状態二つの部に分ける。以下に示すように毎変数を説明する。

- **cZxid (Create ZXID)**: znodeが作成された際のトランザクションIDを示す。
- **ctime (Create Time)**: znodeが作成された時間を示す。
- **mZxid (Modified ZXID)**: znodeが最後に変更された際のトランザクションIDを示す。
- **mtime (Modified Time)**: znodeが最後に変更された時間を示す。
- **pZxid**: このznodeの子節点リストが最後に変更された際のトランザクションIDを示す。子節点の内容が変更されてもpZxidは更新されません。子節点のリストの変更時のみ更新される。
- **cversion**: 子節点のバージョン番号、znode子節点の改修次数を示す。
- **dataVersion**: データのバージョン番号、データの改修次数を示す。
- **aclVersion**: ACL（Access Control List）のバージョン番号、ACLの改修次数を示す。
- **ephemeralOwner**: この臨時znodeが作成された際のセッションIDを示す。持続性節点の場合は値が0です。
- **dataLength**: znodeのデータフィールドの長さを示す。
- **numChildren**: 直下の子節点の数を示す。

### 第４節　 ZNodeのWatcher機構

　　ZooKeeperは、分散データの発行/購読機能を実現するためにWatche機構を使用している。典型的な発行/購読模式では、1つの主題対象を複数の購読者が同時に監聴できる一対多の関係を定義し、主題対象の状態が変化すると全ての購読者に通知され、それに応じた処理を行えるようにする。

　　ZooKeeperでは、このような分散通知機能を実現するためにWatche機構が導入されている。クライアントがサーバーに特定の事件を監聴するならWatcherを登録する必要です。サーバー上で指定された事件が発生すると、ZooKeeperは指定されたクライアントに事件通知を送信し、分散通知機能を実現する。



　　ZooKeeperのWatcher機構は、クライアント、クライアントのWatcherManager、ZooKeeperサーバーの3つの主要な部分から構成されている。

- クライアントはZooKeeperサーバに登録する際に、Watcherの象をクライアントのWatcherManagerに保存する。
- ZooKeeperサーバでWatcher事件が触発されると、サーバはクライアントに通知を送信する。
- その後、クライアントは対応のWatcher対象をWatcherManagerから取り出し、呼び出し返しを実行する。