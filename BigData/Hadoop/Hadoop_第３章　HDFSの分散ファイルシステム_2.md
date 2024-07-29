# 分散大規模データ処理システム -- Hadoop-4

# 第３章　HDFSの分散ファイルシステム-2

## 第４節　HDFSメタデータの管理仕組み

### 4.1　メタデータ管理の概説

#### FsimageとEdits

　　所謂のメタデータの管理が、HDFS内で既定のデータの様々な属性の記録、データ自体の記録、全ての変更操作の記録に関するものです。メタデータの記録量と記録の書き込み速さを兼ね合おうと、HDFSが二つのファイル形式を引用し、それはFsimageとEditsです。

**Fsimage鏡像ファイル**：メタデータの永続的な格納の箇所、HDFS内の全ての目録とファイルのメタデータ情報を含んでいるが、ファイルブロックの位置情報は含まれていません。ファイルブロックの位置情報はメモリにのみ保存されており、Dクラスタ起動ときにNameNodeがdatanodeに問い合わせて取得し、断続的に更新されます。

**Edits編集ログ**：全ての変更操作（ファイルの作成、削除、又は変更）を記録するログです。クライアントから行った変更操作はまずeditsファイルに記録され、記録の過程がメモリに実行します。

　　ディスクに格納したFsimageとメモリに最新の変更記録Editsに結びつけて完全のメタデータファイルに組み合わせてなります。

## 第５節　NNと2NN

　　メタデータ管理にはNameNodeやSecondaryNameNodeから共同に担当します。

**NameNode**：HDFSのメタデータを永続的に保存し、クライアントからのHDFSに対するさまざまな操作の請求を処理します。

**SecondaryNameNode**：NameNodeを協力してFsimageとEditsを併せて完全なメタデータを作成する役割を担当します。

　　SecondaryNameNode名称から見てNameNodeのバックアップサーバーみたいの存在で、ただ2NNがHDFS中にの一つの部品だけで、単独で運行するのは意味ありません。NNにも2NNにもメタデータの処理に関係があります。実はメタデータの処理が確かに一番目重要の部分です。

　　本節はどうメタデータの処理をめぐってNNや2NNの運行仕組み、メタデータの構造を紹介してきます。

### 5.1　メタデータ管理の流れ

![OIP1](D:\OneDrive\picture\Typora\BigData\Hadoop\OIP1.jpg)

NameNode部する

1. 最初NameNodeを初期化にして新たなFsimageとEditsファイルを作成します。その以外起動するときNameNodeが最新のFsimageとEditsファイルをメモリに読み込みます。
2. クライアントから変更操作をメタデータに実行するアクセスを受信します。
3. NameNodeが変更操作しか記録しません。全ての変更記録が先ずEditsファイルに追加していきます。一定の記録を積み重ねるまでに、元の編集されているedits_inprogress_001ファイルがedits_001に保存されて何の改修でも進行しません。その代わりにedits_inprogress_002ファイルが処理中のEditsファイルとします。一回のログローテート「log rotation」がそういう過程です。

　　NameNodeの役割が大体上記のように、纏めてのはクライアントのリクエストを受信、トランザクション記録、editsのログローテート3点であります。

Secondary NameNode部分：

1. NameNodeにログローテートさせる最初の触発点がSecondary NameNodeからCheckPoint（検査点）をする必要かどうかをNameNodeに問う。
2. 許可を得ったSecondary NameNodeがCheckPoint実行のリクエストを送信する。
3. 指令をもらったNameNodeがEditsのログローテートを進行する。
4. 最新の保存されたedits_inprogress_001とFsimageがSecondary NameNodeにダウンロードされる。
5. edits_inprogress_001とFsimageがメモリに読み込まれる。
6. edits_inprogress_001とFsimageがSecondary NameNodeのメモリに合併して新しいFsimageファイルを生み出します。一応Fsimage.chkpointと呼ぶ。
7. Fsimage.chkpointファイルをコピーしてNameNodeへ転送する。
8. Fsimage.chkpoint名称を改名して元のFsimageファイルを上書く。

　　一つ完全のメタデータの維持流れが大体上記のようで、処理方法は難しくなくてかなり巧妙だと思います。第二次ログローテートするときにedits_inprogress_002を取ってだけ、Secondary NameNodeにの現存Fsimageを合併していいんです。

### 5.2　FsimageとEditsファイルの分析

　　NameNodeサーバー`/opt/bigdata/servers/hadoop-2.9.2/data/tmp/dfs/name/current`目録にFsimageとEditsメタデータがあります。

![image-20240715160949061](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240715160949061.png)

- 最大数の結尾数`~225`のFsimageが最新で、そのに対して結尾数~225のEditsがログローテートしたの中に最新のです。
- edits_inprogress_~00256名称ファイルが編集中のEditsで、まだFsimageに併せてない。
- 複数のEdits内容が一括にFsimageに書き込みも可能です。
- seen_txidには最新のEditsファイルの結尾数256を記録し、毎回起動してseen_txidの数値を通ってedits_inprogressファイルが紛失していないか確認する。
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

**書庫ファイル**

　主にHDFSクラスターで大量の小さなファイルが存在する問題を解決するためです。大量の小さなファイルはNameNodeのメモリを占有するため、HDFSにとって大量の小さなファイルを保存することはNameNodeのメモリ資源の浪費を引き起こします。

　Hadoopアーカイブファイル（HARファイル）は、より効率的なファイルアーカイブツールです。HARファイルは、一連のファイルをアーカイブツールを使用して作成され、NameNodeのメモリ使用量を削減しながら、ファイルへの透明なアクセスを可能にします。簡単に言うと、HARファイルはNameNodeにとっては1つのファイルとして扱われ、メモリの浪費を減らしながらも、実際の操作では個々のファイルとして処理されます。
