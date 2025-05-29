# 分散大規模データ処理システム -- Hadoop-4

# 第３章　HDFSの分散ファイルシステム-4

## 第７節　NameNodeの故障処理

　HDFSファイルシステムのメタデータは、NameNodeによって管理、維持され、クライアントとのやり取りに欠かせない存在です。そのため、メタデータが破損または喪失した場合、NameNodeが正常に運行できなくなり、HDFS全体の運用に支障を来す恐れがある。

解決方法は以下の2つ：

- **Secondary NameNode（2NN）によるメタデータ復旧**

　2NNに保持されているメタデータを、NameNode節点にコピーしすることでメタデータの回復を試す。ただし、まだ2NNに転送されないedits_inprogress更新ログが2NNに含まれないため、その部分が失われる可能性がある。

- **HA（高可用性）構成の導入**

　NameNode の単一障害点（Single Point of Failure）を回避するために、HDFSのHA（High Availability）構成を導入する。この方法では、Zookeeperを利用してNameNodeのバックアップ節点を管理し、障害が発生した際には自動で障害切り替え（failover）を実現する。NameNodeが故障してもサービスを継続できる。

## 第８節　Hadoopの限定額、安全模式や書庫ファイル

**HDFSの限定額を配置**

　HDFSの限定額は、特定のディレクトリにアップロードされるファイル数と、ファイルの総容量を制限することです。

- 数量限定

```
#ディレクトリ下に1つに限定
hdfs dfsadmin -setQuota 1 /EditsTest

#ファイルをアップロード
hdfs dfs -put /root/fsioutput.xml /EditsTest
```

![image-20240729153746234](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240729153746234.png)

- 総容量限定

```
#4k容量限定
hdfs dfsadmin -setSpaceQuota 4k /EditsTest

#ファイルをアップロード
hdfs dfs -put /root/fsioutput.xml /EditsTest
```

![image-20240729155039117](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240729155039117.png)

　制限を違反するエラーメッセージを返している。

```
#限定条件を検査
hdfs dfs -count -q -h /EditsTest

#数量限定を解除
hdfs dfsadmin -clrQuota /EditsTest

#総容量限定を解除
hdfs dfsadmin -clrSpaceQuota /EditsTest
```

![image-20240729160306602](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240729160306602.png)

**HDFS安全模式**

　安全模式は、HDFSの特殊な状態です。この状態では、ファイルシステムはデータの読み取りリクエストのみを受け付け、削除や変更などの変更リクエストは受け付けない。NameNodeが起動する際、HDFSはまず安全模式に入り。各DataNodeを起動して利用可能な節点がブロックの情報を取得する。システム全体が安全基準に達するとHDFSは自動的に安全模式を解除する。

```
#手動的に安全模式入って
hdfs dfsadmin -safemode [enter|leave|get]
```

**HDFSアーカイブ**

　HDFSアーカイブ（archive）は、HDFSクラスタ上に大量の小さなファイルが存在するによる問題を解決するための手法です。小さなファイルが大量に存在すると、各ファイルが128MB単位のブロックで格納されるため、メモリ資源の浪費が発生した。

　この問題に対して、Hadoop書庫ファイル（HARファイル）が利用される。HARファイルは、一連の小さいファイルを合併して一つのHARファイルにまとめることで、NameNodeのメタデータ負荷を減らす。ただし、実際の処理はあくまで個別ファイル単位で処理する。

```
#アーカイブを要するディレクトリを作成
hdfs dfs -mkdir /archiveTest

#テストファイルをアップロード
hdfs dfs -put /root/fsimageTest.xml /archiveTest
hdfs dfs -put /root/hadoopTest.txt /archiveTest
hdfs dfs -put /root/wordCountTest.txt /archiveTest
```

![image-20240731102540902](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731102540902.png)

　全てのファイルをinput.harフォルダにアーカイブされる。HARファイルは`/archiveOutput`に出力する。

```
hadoop archive -archiveName input.har –p /archiveTest /archiveOutput
```

![image-20240731113250581](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731113250581.png)

![image-20240731113350041](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731113350041.png)

```
hdfs dfs -lsr /archiveOutput
```

![image-20240731114156986](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731114156986.png)

- _SUCCESS：アーカイブ成功した標識
- index：アーカイブ内のファイルやディレクトリのメタデータを保存する索引ファイル
- _masterindex：データのブロック索引情報を保存するファイル
- part-0：データを格納するところ

　アーカイブ生成の流れは、大体下図のように表される。本質的には、複数の小さなファイルを1つの別形式にまとめて保存する構造です。元のファイルはそのまま存在するため、ディスク容量を削減したい場合は、元のファイルを削除する必要がある。

![97084776](D:\OneDrive\picture\Typora\BigData\Hadoop\97084776.png)

　`har://`コマンドを付けてアーカイブファイトの内容を読み取ることができる。

```
#アーカイブのデータを検査
hadoop fs -ls har:///archiveOutput/input.har
```

![image-20240731141401899](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731141401899.png)

　アーカイブファイルの還元、圧縮の解凍操作に似る。

```
#存在したディレクトリが必要
hdfs dfs -mkdir /archive

#アーカイブを解除し、指定ディレクトリに還元
hadoop fs -cp har:///archiveOutput/input.har/* /archive

hdfs dfs -ls /archive
```

![image-20240731141317982](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731141317982.png)

　還元できため、元のファイルを削除しても構わない。

![image-20240731141555763](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731141555763.png)

　NameNodeにとっては、各ファイルが個別の入口として扱われており、構造上は変化してない、荷造りにして格納されるだけです。でも、書き込みなど更新操作が不可で、更新が必要なデータには適してない。ここは要注意です。

## 第９節　NameNodeの高負荷並行処理

　HDFSのメタデータ管理について学び、クライアントの更新アクセスがあるたびに、NameNodeはその内容を更新ログに記録することが分かる。一定量が蓄積されると、ディスクに書き込まれ、edits_logファイルを生成し、その一連の処理は時間がかかる。もし各プロセスに対して逐一ロックをかける設計にしてしまうと、HDFS全体の処理効率が大きく低くなる。

　そこで、所謂「高負荷の並行処理」は安全に直列処理を行いつつ、効率的に処理を切り替えるという設計です。

　HDFSではこの問題に対して分割ロック（Fine-Grained Locking）とダブルバッファリング（double buffering）仕組みを導入して対応する。

**分割ロック**

　アクセス数に関わらず、HDFSでは各更新に一意の自動増分txidが割り当てられ、そのIDの順番に従ってデータがディスクに書き込むという2点は保証されている。ただし、txidが割り当てとディスクへの書き込みをセットして1つロックに入れて処理するには、前者は一瞬終わるにもかかわらず、後続のディスク書き込みが完了するまで待機しなければならず、資源が長時間占有されることになる。それはかなり非効率的です。

　この問題を解決するために、HDFSはtxidの割り当てとディスクの書き込みを2つ部分に分離し、それぞれに単独のロックを適用する分割ロック（Fine-Grained Lock）を採用している。これにより、両者は独立のプロセスとして並行処理され、処理の待ち時間を削減している。

**ダブルバッファリング**

　txidが割り当てとディスクの書き込みは、現在それぞれ独立して動作するように分離されている。しかし、その2つの処理は実行の速度差がある。処理のバランスを取るためにはバッファリングが必要となる。

　HDFSはダブルバッファリング（Double Buffering）策略を導入してさらなる効率化を図っている。

![Double buffering](D:\OneDrive\picture\Typora\BigData\Hadoop\Double_buffering.png)

　プロセスは2つ同じバッファリングを作成することがある。1つはbufferA、クライアントから更新記録を受けてバッファリングに格納する。1つはbufferB、ディスクへデータを出力する。ディスクに書き込みより、バッファリングの方は早く上限に至って停止になる。そのため、bufferAはロックされる状態になってbufferBの転送が終わるまで待っている。bufferBの転送が終わったら、bufferAとbufferBを取り替え、待機中のスレッドが起こされてbufferBへ更新記録を書き込む。bufferAにいる更新記録ディスクへ書き込む。

　