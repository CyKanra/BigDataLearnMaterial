# 分散大規模データ処理システム -- Hadoop-4

# 第３章　HDFSの分散ファイルシステム-3

　本章では、HDFSにおけるメタデータ管理の仕組み、ならびにSecondary NameNodeによるチェックポイント処理の役割について説明する。

## 第４節　HDFSメタデータの管理仕組み

### 4.1　メタデータ管理の概説

　メタデータは、HDFSに格納されたデータの様々な属性を記録するものです。例えば、データのサイズ、作成時刻、格納ディレクトリなど情報であり、言わば「スナップ写真」、或いは「管理目録」のような役割を果たす。

　データの更新する際には、メタデータ参照することで情報を素早く取得し、更新操作を実行することができる。もちろん、メタデータはデータ自身じゃなく、データとそのバックアップが全ての節点（DataNode）に失われてしまった場合、データの回復はできない。

**Fsimage鏡像ファイル**：スナップショット（snapshot）形式で保持されたメタデータ。起動時にこれを読み込む。ディスク上に保持されるため、電源が落ちてもデータが失われることはない。

　ビッグデータを管理するFsimageファイルは、非常に大きなサイズになることがある。そのため、毎回更新記録をFsimageに反映する処理が遅くなっている。この問題を解決するため、メタデータの記録量と書込み速度のバランスを取る手法として、Editsログを導入されている。更新操作を個別に記録し、効率的に処理する。

**Edits編集ログ**：ファイルシステムへの変更履歴、全ての更新操作（新規、削除、改修など）を記録するログ。その更新操作は先ずEditsファイルに書き込まれ、この書き込み処理はメモリ上で行う。 一定の量に蓄積されると、Editsの内容をFsimageに統合される。

　ディスク上のFsimageファイルと、メモリ上のEditsログを組み合わせることで、完全なメタデータが構成される。

## 第５節　NNと2NN

　メタデータの管理は、NameNodeとSecondaryNameNode2つの節点によって共同で行う。

**NameNode**：メタデータを保存するところ、クライアントからの要求を処理する。

**SecondaryNameNode**：NameNodeを補助し、FsimageとEditsを統合する役割を担当する。

　本節では、メタデータの処理に関連するNameNode(NN)やSecondaryNameNode(2NN)の動作の仕組みや、メタデータの構造について紹介する。

### 5.1　メタデータ管理の流れ

![OIP1](D:\OneDrive\picture\Typora\BigData\Hadoop\OIP1.jpg)

NameNode(NN)部分：

1. 初めてHadoopを起動する際には、初期化を行い、Fsimageファイルを新規作成する必要がある。それ以外の場合は通常通り起動し、NameNodeは最新のFsimageファイルを読み込む。
2. クライアントから要求を受信する。
3. NameNodeが変更操作のみを記録する。変更操作は先ずEditsファイルに追記される。ある程度記録が蓄積されると、編集中だったedits_inprogress_001ファイルがedits_001という名前で保存され、編集が終了した状態になる。そして、新たedits_inprogress_002ファイルを作成され、以後の更新記録はこのファイルに書き込まれる。このような一連の流れが、1回の完全の更新処理のサイクルとなる。

Secondary NameNode(2NN)部分：

1. Secondary NameNodeはNameNodeに対して、更新ログの統合作業が必要かどうかを問い合わせる信号を送信する。
2. 統合の許可が返された場合、2NNは改めてリクエストを送信する。その統合可否の条件は設定できる。
3. リクエストを受信したNameNodeは、最新のEditsのログを2NNに転送する。
4. 編集中のedits_inprogress_002は対象外です。複数のedits_001あるならすべて転送される。
5. 2NNは、最新の2つのedits_001とFsimageをメモリに書き込んで統合し、新しいFsimage.chkpointファイルを生成する。
6. 生成したFsimage.chkpointファイルをNameNodeに転送する。
7. NameNodeは受け取ったFsimage.chkpoint名称を変更して元のFsimageファイルを上書きする。
8. 今はedits_inprogress_002を除いてFsimageは最新記録になる。

　上記のようなSecondary NameNodeによるログ統合の一連の処理はCheckPointと呼ばれる。NameNodeのメタデータに関する更新処理は、大部分Secondary NameNodeによって支持する。

### 5.2　FsimageとEdits

　NameNodeサーバー`../hadoop-2.9.2/data/tmp/dfs/name/current`目録にFsimageとEditsメタデータがある。

![image-20240715160949061](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240715160949061.png)

- 末尾番号は225の`fsimage_00~0225`ファイルと、そのに対尾する`edits_00~0223-00~0225`のようなEditsログは、1セットの統合済みメタデータです。225までのEditsログは既に統合された。それ以降のEditsログはまだ処理しない。
- `edits_inprogress_00~0256`というファイル名は、記録中のまだFsimageに統合されない。
- Editsログは必ずしも1ファイルずつ統合されるわけではない。複数のEditsログファイル(243`〜`255)を合わせて一括にEditsに書き込むことも見える。
- seen_txidファイルには、まだ処理されないEditsファイルの結尾数番号(256)を記録する。Hadoop起動する際、この値を元にどこまで統合が済んでいるかを確認する。
- VERSIONには、クラスタに関するバージョン情報などの基本情報が保存されている。

　 次VERSIONファイルの内容を紹介する。

```
cat VERSION
```

![image-20240719162558943](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240719162558943.png)

![image-20240719162620471](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240719162620471.png)

　**namespaceID**：namespaceとは、HDFSファイルシステムにおけるディレクトリ構造と空間の概念を指す。namespaceIDはその名前空間を対応する一意の識別子です。1つの節点が複数の名前空間に属することも可能です。

　**clusterID**：HDFSクラスタ全体を識別するため一意のIDです。クラスタが初期化される際に自動的に生成され、すべての節点で共通して使用される。このIDを使って、この節点が本クラスタに属するかどうかを判断する。namespaceIDは複数設定可能ですが、clusterIDはクラスタごとに1つしか設定できない。

　**blockpoolID**：ブロックプール（Block Pool）とは、HDFSにおけるデータブロックの集合を意味する。blockpoolIDはこのブロックプールを一意に識別するIDであり、例えば、`BP-1480344265-192.168.31.135-1711003810821`のような形式で、NameNodeのIPアドレスと`cTime`初期化の時刻情報が含まれる。

- edits_inprogressとseen_txidを除いて、NameNodeとファイル結構は大体同じです。

![image-20240716144751072](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716144751072.png)

- 他の普通の節点がメタデータがない。

![image-20240716145030989](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716145030989.png)

### 5.3　FsimageとEdits内容の分析

　FsimageおよびEditsがすべて序列化される形式で保存されており、通常のテキストエディタでは内容を直接確認できない。Hadoop公式はこれを反序列化してXML形式で可視化する方法を提供している。

公式URL：[Apache Hadoop 2.9.2 – Offline Image Viewer Guide](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/HdfsImageViewer.html)

####  Fsimage

　oivコマンドを使用することでFsimageファイルをXML形式に変換できる。

> oiv：Offline Image Viewer View a Hadoop fsimage INPUTFILE using the specified PROCESSOR,saving the results in OUTPUTFILE.

```
cd /opt/bigdata/servers/hadoop-2.9.2/data/tmp/dfs/name/current

#Hadoopサービス停止状態でも実行でき
hdfs oiv -p XML -i fsimage_0000000000000000253 -o /root/fsimageTest.xml
```

![image-20240716160142940](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716160142940.png)

![image-20240716160202722](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716160202722.png)

```
#ローカルにダウンロード
sz fsimageTest.xml
```

![image-20240722165614346](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240722165614346.png)

　Notepad++にXMLツールがあれば、XMLファイルを整形して表示する（Ctrl+Alt+Shift+B）。

![image-20240722170324363](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240722170324363.png)

　代替として、Linuxサーバー上で xmlstarlet ツールを利用し、XMLの検査・整形が可能です。

```
sudo yum install xmlstarlet
```

　もしインストール失敗したら、Centos7のyumの鏡像（mirror）ダウンロードアドレスが失効になったかもしれない。`/etc/yum.repos.d/CentOS-Base.repo`このファイルを新しい鏡像アドレスに変更し、依頼を改めてダウンロードしていい。

解決の方法：[緊急対応！CentOS 7 サポート終了後のyumエラー解消法 #ShellScript - Qiita](https://qiita.com/owayo/items/81c843fb11d27b217433)

```
#既存依頼を消除
sudo yum clean all

#依頼をダウンロード
sudo yum makecache
```

　次、xmlファイルを整形する。

```
xmlstarlet fo fsimageTest.xml > fsioutput.xml
```

![image-20240717164837672](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240717164837672.png)

```
cat fsioutput.xml
```

![image-20240724144054058](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240724144054058-1721799762791-1.png)

- `<txid>`:`seen_txid` と一致するID値

- `<INodeSection>`:メタデータ
- `<inode>`:1つのファイルやディレクトリの状態
- `<id>`：`<inode>`の一意な識別子
- `<type>`：`<inode>`のタイプ、DIRECTORYまたはFILEなど。
- `<name>`：HDFSにその対象の位置。
- `<mtime>`：最終更新時刻（ミリ秒）
- `<permission>`：所有者とアクセス権限

　`<inode>`のtypeはFILE類の場合、`<block>`タブが付属し、hsfsTest.txtファイルに対応のブロック情報を持つ。ただ、ブロックの物理的な格納場所が記録されない。クラスタ起動時に、安全モード（safemode）に入り、各DataNodeが自身のブロック情報をNameNodeへ報告する。一定の間隔でブロック情報をNameNodeに送信し、正確な配置が把握される。

![image-20240724162039062](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240724162039062.png)

![image-20240724162208628](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240724162208628.png)

　`<INodeDirectorySection>`はファイトとフォルダのディレクトリ構造（親子関係）を表す。`<parent>` と複数の `<child>` タグの数字は`<inode>`のidに対応する。

![image-20240725143913777](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725143913777-1721886036645-1.png)

####  Edits

　もっと分かりやすく説明するために、例を挙げて紹介する。

```
#フォルダを作成
hdfs dfs -mkdir /EditsTest

#ファイルを導入
hdfs dfs -put /root/wordCountTest.txt /EditsTest
```

![image-20240725152414091](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725152414091.png)

　oevコマンドを使ってEditsファイルをXML形式に変換できる。

> oev：Offline edits viewer Parse a Hadoop edits log file INPUT_FILE and save results in OUTPUT_FILE

```
cd /opt/bigdata/servers/hadoop-2.9.2/data/tmp/dfs/name/current

hdfs oev -p XML -i edits_inprogress_0000000000000000289 -o /root/edits.xml
```

![image-20240725152646505](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725152646505.png)

```
cat edits.xml
```

![image-20240725152927720](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725152927720-1721889000826-3.png)

　2回の更新操作は、赤枠のように示されている。1つ目の`<OPCODE>`タブには`OP_MKDIR`が書かれ、mkdirコマンドに対応してフォルダの作成操作を表す。2つ目に`OP_ADD`はファイルのアップロードを意味し、それぞれ異なる更新タイプに応じて適切なコードが記録されている。

## 第６節　checkpoint周期

　checkpointの周期は変更可変です。公式ではhdfs-default.xmlにある。`dfs.namenode.checkpoint.period`引数で設定できると説明されている。しかし実際は、hdfs-default.xmlファイルが存在せず、これはあくまでデフォルト値の定義をまとめた仮想的なファイル名として使われる。本当に役立つのはhdfs-site.xmlファイルです。

![image-20240726160229611](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240726160229611.png)

　以下は、チェックポイントに関する3種類の設定例です。

```
<!-- 一時的な間隔でのチェックポイント設定（秒単位） -->
<property>
	<name>dfs.namenode.checkpoint.period</name>
	<value>3600</value>
</property>
<!-- 100万件に達したらチェックポイント信号を送信 -->
<property>
	<name>dfs.namenode.checkpoint.txns</name>
	<value>1000000</value>
</property>
<!-- 1分ごとに変更操作の有無を確認 -->
<property>
	<name>dfs.namenode.checkpoint.check.period</name>
	<value>60</value>
</property>
```
