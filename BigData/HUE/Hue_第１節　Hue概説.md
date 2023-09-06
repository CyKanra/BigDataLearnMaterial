# データ交互工具ーHUE

## 第１節　Hue概説

　　Hue（Hadoop User Experience）は、オープンソースのApache Hadoop UIシステムであり、最初はCloudera Desktopから進化して、Clouderaがオープンソースコミュニティに贈るものです。HueはPythonのウェブフレームワークDjangoに基づいて実装されてる。Hueを使用することで、Web制御盤を通じてブラウザ上でHadoopクラスタと対話し、データを分析、処理できる。Hueが支持する機能には以下がある：

- デフォルトでは、軽量なSQLiteデータベースを使用してセッションデータ、ユーザー認証および権限管理を管理し、必要に応じてMySQL、PostgreSQL、Oracleにカスタマイズできます。
- HDFSにアクセスするためのファイルブラウザ機能。
- Hiveエディタを使用してHiveクエリの開発と実行が可能。
- Solrを基にした検索アプリケーションのサポートで、データの視覚化とダッシュボードが提供されます。
- Impalaを使用した対話型クエリアプリケーションのサポート。
- Sparkエディタとダッシュボードの提供。
- Pigエディタのサポートとスクリプトタスクの実行が可能。
- Oozieエディタを提供し、ワークフローやコーディネータ、バンドルの提出と監視が可能。
- HBaseブラウザをサポートし、データの視覚化、クエリ、HBaseテーブルの編集が可能。
- Metastoreブラウザを提供し、HiveのメタデータとHCatalogへのアクセスが可能。
- Jobブラウザを提供し、MapReduce Job（MR1/MR2-YARN）にアクセスが可能。
- ジョブデザイナーをサポートし、MapReduce/Streaming/Javaジョブの作成が可能。
- Sqoop 2エディタとダッシュボードを提供。
- ZooKeeperブラウザとエディタをサポート。
- MySQL、PostgreSQL、SQLite、Oracleデータベースのクエリエディタを提供。

　　以上の要点が今に理解する必要ない、一言で言えば、HueはHadoopクラスタの操作とHadoopに関する技術の扱いを容易に行うことができる界面フレームワークです。

Hue公式サイト：[Hue - The open source SQL Assistant for Data Warehouses (gethue.com)](https://gethue.com/)

Hue公式ユーザー手引き：[Hue Guide :: Hue SQL Assistant Documentation (gethue.com)](https://docs.gethue.com/)

Hue公式インストール文献：[Install :: Hue SQL Assistant Documentation (gethue.com)](https://docs.gethue.com/administrator/installation/install/)

Hueダウンロードリンク：[Tags · cloudera/hue · GitHub](https://github.com/cloudera/hue/tags)![Managed File Transfer by Linoma Software - ahtech.com.au](https://image.slidesharecdn.com/presentation2-180418102950/95/big-data-86-638.jpg?cb=1524047741)
