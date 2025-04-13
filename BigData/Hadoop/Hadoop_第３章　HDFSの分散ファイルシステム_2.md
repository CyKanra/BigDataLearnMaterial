# 分散大規模データ処理システム -- Hadoop-4

# 第３章　HDFSの分散ファイルシステム-2

## 第４節　HDFSメタデータの管理仕組み

### 4.1　メタデータ管理の概説

　メタデータは、HDFSに格納されたデータの様々な属性を記録するものです。例えば、データのサイズ、作成時刻、格納ディレクトリなど情報であり、言わば「スナップ写真」、或いは「管理目録」のような役割を果たす。

　データの更新する際には、メタデータ参照することで情報を素早く取得し、更新操作を実行することができる。もちろん、メタデータはデータ自身じゃなく、データとそのバックアップが全ての節点（DataNode）に失われてしまった場合、データの回復はできない。

**Fsimage鏡像ファイル**：メタデータを保存するファイルで、ディスク上に保持されるため、電源が落ちてもデータが失われることはない。

　ビッグデータを管理するFsimageファイルは、非常に大きなサイズになることがある。そのため、毎回更新記録をFsimageに反映すると処理が遅くなってしまった。

　この問題を解決するために、メタデータの記録量と書込み速度のバランスを取る手法として、Editsログを導入されている。更新操作を個別に記録し、効率的に処理する。

**Edits編集ログ**：全ての更新操作（新規、削除、改修など）を記録するログファイトです。その更新操作は先ずEditsファイルに書き込まれ、この書き込み処理はメモリ上で行う。 一定の量に蓄積されると、Editsの内容をFsimageに統合される。

　　ディスク上のFsimageファイルと、メモリ上のEditsログを組み合わせることで、完全なメタデータが構成される。なお、ファイルを格納しているブロックの位置情報は、どちらにも記録されてない。Hadoop起動する時に各DataNodeから取得し、リアタイで最新の情報を読み込む。

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
2. 統合の許可が返された場合、2NNは改めてリクエストを送信する。
3. リクエストを受信したNameNodeは、最新のEditsのログを2NNに転送する。
4. 編集中のedits_inprogress_002は対象外です。複数のedits_001あるならすべて転送される。
5. 2NNは、最新の2つのedits_001とFsimageをメモリに書き込んで統合し、新しいFsimage.chkpointファイルを生成する。
6. 生成したFsimage.chkpointファイルをNameNodeに転送する。
7. NameNodeは受け取ったFsimage.chkpoint名称を変更して元のFsimageファイルを上書きする。

　上記のようなSecondary NameNodeによるログ統合の一連の処理はCheckPointと呼ばれる。NameNodeのメタデータに関する更新処理の大部分は、Secondary NameNodeによって支持される

### 5.2　FsimageとEditsファイルの分析

　NameNodeサーバー`../hadoop-2.9.2/data/tmp/dfs/name/current`目録にFsimageとEditsメタデータがある。

![image-20240715160949061](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240715160949061.png)

- 末尾番号は225の`fsimage_00~0225`ファイルと、そのに対尾する`edits_00~0223-00~0225`のようなEditsログは、1セットの統合済みメタデータです。225までのEditsログは既に統合された。それ以降のEditsログはまだ処理しない。
- `edits_inprogress_00~0256`というファイル名は、記録中のまだFsimageに統合されない。
- 複数のEditsログファイルを纏めて一緒に統合するに限らない。更新内容を一括にEditsに書き込むことも見える。
- seen_txidには最新のEditsファイルの結尾数256を記録し、毎回起動してseen_txidの数値によってどこまで統合されたことを確認する。
- VERSIONにはクラスタの情報を記録する。

```
cat VERSION
```

![image-20240719162558943](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240719162558943.png)

![image-20240719162620471](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240719162620471.png)

　　**namespaceID**：HDFSファイルシステム内のすべてのファイルと目録を含む抽象的な概念です。一般的に一つのクラスタが一意のnamespaceID存在するだけ。複数の名前空間が必要な場合、Hadoopは連合HDFS（Federated HDFS）という働きを提供し、複数の独立したルートディレクトリ（root directory）に対して名前空間のを持てます。前提がhdfs-site.xml改修が必要です。

　　**clusterID**：HDFSクラスター全体を一意に識別するためのUUID（汎用一意識別子）です。HDFSクラスタが初期化されるときに生成され、クラスタ全体で一貫して使用されます。この節点が本クラスタに所属するかどうか判断します。namespaceIDを複数にできるけれど、clusterIDにはいけません。ただclusterIDを追加したり、新しいclusterID再生したりできます。前者が新しい節点を添加し、後者が初期化にします。

　　**blockpoolID**：HDFSのブロックプール（Block Pool）を一意に識別するための識別子です。ブロックプールは、HDFS内のデータブロックの集合であり、ファイルシステム内のデータを物理的に保存する単位です。`BP-1480344265-192.168.31.135-1711003810821`識別子がNameNodeのIPアドレスと`cTime`初期化の時刻を含みます。

- Secondary NameNodeにはedits_inprogressとseen_txid二つのを除いて大体同じです。

![image-20240716144751072](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716144751072.png)

- 他の普通の節点がメタデータがない。

![image-20240716145030989](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716145030989.png)

### 5.3　FsimageとEditsファイルの内容

　　FsimageとEditsが全て序列化にされるため普通の編集ツールで照査できない。公式が方法を提供して反序列化にして可検査のxmlファイルになれます。

公式URL：[Apache Hadoop 2.9.2 – Offline Image Viewer Guide](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/HdfsImageViewer.html)

####  Fsimage内容

　　oiv：Offline Image Viewer View a Hadoop fsimage INPUTFILE using the specified PROCESSOR,saving the results in OUTPUTFILE.

　　oivコマンドを使ってFsimageファイルを反序列化にできます。

```
cd /opt/bigdata/servers/hadoop-2.9.2/data/tmp/dfs/name/current

#Hadoopサービス停止状態でも実行でき
hdfs oiv -p XML -i fsimage_0000000000000000253 -o /root/fsimageTest.xml
```

![image-20240716160142940](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716160142940.png)

![image-20240716160202722](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240716160202722.png)

```
#自分コンピューターにダウンロード、多分Downloads目録にいる
sz fsimageTest.xml
```

![image-20240722165614346](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240722165614346.png)

　　もしNotepad++にXMLツールがあり、Ctrl+Alt+Shift+Bを押してXMLファイルを格式化にできます。

![image-20240722170324363](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240722170324363.png)

　　べつの方法がサーバーにxmlstarletツールを介して検査もできます。

```
sudo yum install xmlstarlet
```

　　もし上のコマンド失敗したら、多分Centos7のyumの鏡像（mirror）ダウンロードアドレスが失効になりました。`/etc/yum.repos.d/CentOS-Base.repo`このファイルを新しい鏡像アドレスに変更し、依頼をダウンロードしていい。

解決の方法：[緊急対応！CentOS 7 サポート終了後のyumエラー解消法 #ShellScript - Qiita](https://qiita.com/owayo/items/81c843fb11d27b217433)

```
#既存依頼を消除
sudo yum clean all

#依頼をダウンロード
sudo yum makecache
```

　　次は新しいファイルに転換して格式化にしました。

```
xmlstarlet fo fsimageTest.xml > fsioutput.xml
```

![image-20240717164837672](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240717164837672.png)

```
cat fsioutput.xml
```

![image-20240724144054058](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240724144054058-1721799762791-1.png)

　　頭部`<version>`と`<NameSection>`包まれた部分が別にHDFSファイルシステムのバージョン情報と名前空間の特定のメタデータ情報です。中に`＜txid＞`対しての数字が`seen_txid`と同じです。

　　次は重点で、`<INodeSection>`包まれたの記録は所為のメタデータです。`<inode>`に一回変更操作含まれます。

`<id>`：`<inode>`の一意の識別子です。

`<type>`：`<inode>`の種類、DIRECTORY`またはFILEなどの値を含む。

`<name>`：これは目録やファイルの名前です。

`<mtime>`：当の目録内に何でも変更に限り、この最後の変更の時刻をミリ秒単位で示す。「modification time」の略です

`<permission>`：ファイル又は目録の権限。

　　下図がtypeがFILE類型の`<inode>`で、比べてして余分な`<block>`がhsfsTest.txtファイルに対してのブロック情報です。ブロックの一意の識別子、変更バージョン、ブロック大小など含まれ、ただブロックの所在の節点情報がありません。クラスタ起動の時に、安全模式（safemode）に入ってあり、各節点がNameNodeに自分持ちのブロック情報を報告します。後で一定の間隔でその流れを繰り返してあり、ブロックの格納情報が最新に保証します。

![image-20240724162039062](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240724162039062.png)

![image-20240724162208628](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240724162208628.png)

　　`<INodeSection>`内の各`<inode>`が実行の前後順で並べます。実際の目録結構を表すメタデータがファイルの後ろにいます。

![image-20240725143913777](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725143913777-1721886036645-1.png)

####  Edits内容

　　もっと説明しやすいため、ここ幾つか案例を用意しております

```
hdfs dfs -mkdir /EditsTest

hdfs dfs -put /root/wordCountTest.txt /EditsTest
```

![image-20240725152414091](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725152414091.png)

　　oev：Offline edits viewer Parse a Hadoop edits log file INPUT_FILE and save results in OUTPUT_FILE

　　oevコマンドを使ってEditsファイルを反序列化にできます。

```
cd /opt/bigdata/servers/hadoop-2.9.2/data/tmp/dfs/name/current

hdfs oev -p XML -i edits_inprogress_0000000000000000289 -o /root/edits.xml
```

![image-20240725152646505](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725152646505.png)

```
cat edits.xml
```

![image-20240725152927720](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240725152927720-1721889000826-3.png)

　　先二次の操作が図に見えます。大部分のタグの意味が分かりやすい、ここ`<OPCODE>`を取り出して説明してきます。一番目の`<OPCODE>`が`OP_MKDIR`を書くと、ちょうどmkdirコマンドに対応して目録を作成の流れを表示します。二番目の`OP_ADD`が新しいファイルを添加の意味です。

## 第６節　checkpoint周期

　　checkpoint周期の設定が可変です。公式で右側目録の最低にhdfs-default.xmlに設定できます。対応の引数が`dfs.namenode.checkpoint.period`です。実際にhdfs-default.xml存在しないんで、ただデフォルト設定を説明しやすいために名付ける。本当に役立つのがhdfs-site.xmlです。

![image-20240726160229611](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240726160229611.png)

　　以下の幾つか別の設定を参照できます。

```
<!-- 一時定時 -->
<property>
	<name>dfs.namenode.checkpoint.period</name>
	<value>3600</value>
</property>
<!-- 一分で変更操作を検査し、1百万次までcheckpoint信号を送信 -->
<property>
	<name>dfs.namenode.checkpoint.txns</name>
	<value>1000000</value>
</property>
<property>
	<name>dfs.namenode.checkpoint.check.period</name>
	<value>60</value>
</property>
```

## 第７節　NameNodeの故障処理

　HDFSファイルシステムのメタデータがNameNodeによって管理、維持され、クライアントと対話する必要があるためです。NameNodeが故障するとメタデータが破損または喪失になるなら、NameNodeが正常に運行できません。

1. SecondaryNameNode（2NN）のメタデータをNameNode（NN）の節点にコピーする。この方法では、edits_inprogressメタデータが存在しないため一部が失われる可能性がる。
2. HDFSのHA（高可用性）クラスタを構築し、NameNodeの単一障害点の問題を解決する。Zookeeperを利用してHAを実現し、1つ活躍NameNodeと1つ控えNameNodeを設定する。

## 第８節　Hadoopの限定額、安全模式や書庫ファイル

**HDFSの限定額を配置**

　HDFSファイルの限定額設定では、特定のディレクトリにアップロードされるファイル数量やファイルの総容量を制限してきます。

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

　安全模式は、HDFSの特殊な状態です。この状態では、ファイルシステムはデータの読み取りリクエストのみを受け付け、削除や変更などの変更リクエストは受け付けません。NameNodeが起動する際、HDFSはまず安全模式に入ります。DataNodeが起動すると、利用可能な節点がブロックの状態をNameNodeに報告します。システム全体が安全基準に達するとHDFSは自動的に安全模式を解除します。

　HDFSが安全模式にファイルブロックは複製操作を行いません。安全模式が解除されるとNameNodeはDataNodeにブロックの複製品を添加するように指示します。

```
#手動的に安全模式入って
hdfs dfsadmin -safemode [enter|leave|get]
```

**HDFSアーカイブ**

　HDFSアーカイブ（archive）は、HDFSクラスタで大量の小さなファイルが存在する問題を解決するための方案です。大量の小さなファイルは、128Mブロック形式でNameNodeのメモリに格納するため、メモリ資源の浪費を引き起こします。

　Hadoopアーカイブファイル（HARファイル）は、より効率的なファイルアーカイブツールです。HARファイルは一連のファイルを合併して作成されます。NameNodeのメモリ使用量を削減しながら、実際の操作でNameNodeにとっては単独の小さなファイルとして処理します。

```
#アーカイブを要するディレクトリを作成
hdfs dfs -mkdir /archiveTest

#ファイルをアップロード
hdfs dfs -put /root/fsimageTest.xml /archiveTest
hdfs dfs -put /root/hadoopTest.txt /archiveTest
hdfs dfs -put /root/wordCountTest.txt /archiveTest
```

![image-20240731102540902](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731102540902.png)

　全てのファイルを一つinput.har名称ファイルにアーカイブします。出力結果が/archiveOutputに置きます。

```
hadoop archive -archiveName input.har –p /archiveTest /archiveOutput
```

![image-20240731113250581](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731113250581.png)

![image-20240731113350041](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731113350041.png)

```
hdfs dfs -lsr /archiveOutput
```

![image-20240731114156986](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731114156986.png)

　アーカイブ流れが大体その下図のように、アーカイブされたファイルが別の結構で保存します。

![97084776](D:\OneDrive\picture\Typora\BigData\Hadoop\97084776.png)

- _SUCCESS：アーカイブ成功した標識
- index：アーカイブ内のファイルやディレクトリのメタデータを保存する索引ファイル
- _masterindex：アーカイブ内のブロック索引情報を保存するファイル
- part-0：アーカイブされた実データを保存するファイル

```
#原始のデータを検査
hadoop fs -ls har:///archiveOutput/input.har
```

![image-20240731141401899](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731141401899.png)

```
#ここディレクトリ作成が必要で
hdfs dfs -mkdir /archive

#アーカイブを解除し、指定ディレクトリに還元
hadoop fs -cp har:///archiveOutput/input.har/* /archive

hdfs dfs -ls /archive
```

![image-20240731141317982](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731141317982.png)

![image-20240731141555763](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240731141555763.png)

　　NameNodeにとっては、まだ3つファイルに一つ一つ対応することが変わってない、荷造りにして格納するだけです。
