# 分散大規模データ処理システム -- Hadoop-4

# 第３章　HDFSの分散ファイルシステム-2

## 第４節　HDFSメタデータの管理仕組み

### 4.1　メタデータ管理の概説

#### FsimageとEdits

　　所謂のメタデータの管理が、HDFS内で既定のデータの様々な属性の記録、データ自体の記録、全ての変更操作の記録に関するものです。メタデータの記録量と記録の書き込み速さを兼ね合おうと、HDFSが二つのファイル形式を引用し、それはFsimageとEditsです。

**Fsimage鏡像ファイル**：メタデータの永続的な格納の箇所、HDFS内の全ての目録とファイルのメタデータ情報を含んでいるが、ファイルブロックの位置情報は含まれていません。ファイルブロックの位置情報はメモリにのみ保存されており、Dクラスタ起動ときにNameNodeがdatanodeに問い合わせて取得し、断続的に更新されます。

**Edits編集ログ**：全ての変更操作（ファイルの作成、削除、又は変更）を記録するログです。クライアントから行った変更操作はまずeditsファイルに記録され、記録の過程がメモリに実行します。

　　ディスクに格納したFsimageとメモリに最新の変更記録Editsに結びつけて完全のメタデータファイルに組み合わせてなります。

#### NNと2NN

　　メタデータ管理にはNameNodeやSecondaryNameNodeから共同に担当します。

**NameNode**：HDFSのメタデータを永続的に保存し、クライアントからのHDFSに対するさまざまな操作の請求を処理します。

**SecondaryNameNode**：NameNodeを協力してFsimageとEditsを併せて完全なメタデータを作成する役割を担当します。

　　SecondaryNameNode名称から見てNameNodeのバックアップサーバーみたいの存在で、ただ2NNがHDFS中にの一つの部品だけで、単独で運行するのは意味ありません。NNにも2NNにもメタデータの処理に関係があります。実はメタデータの処理が確かに一番目重要の部分です。

　　本節はどうメタデータの処理をめぐってNNや2NNの運行仕組み、メタデータの構造を紹介してきます。

### 4.2　メタデータ管理の流れ

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

　　一つ完全のメタデータの維持流れが大体上記のようで、処理方法は難しくなくてかなり巧妙だと思います。第二次ログローテートするときにedits_inprogress_002を取ってだけ、Secondary NameNodeにの現存のFsimageを合併していいんです。後は、以上の処理流れが100パーセントデータの損失が保証できません。例えば、編集中のEditsファイルがログローテートしてないなのに、メモリにの状態で、NameNode停止してその部分のデータが失ってあります。

### 4.3　FsimageとEditsファイルの分析

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

### 4.4　FsimageとEditsファイルの内容

　　FsimageとEditsが全て序列化にされるため普通の編集ツールで照査できない。公式が方法を提供して反序列化にして可検査のxmlファイルになれます。

公式URL：[Apache Hadoop 2.9.2 – Offline Image Viewer Guide](https://hadoop.apache.org/docs/r2.9.2/hadoop-project-dist/hadoop-hdfs/HdfsImageViewer.html)

oiv：Offline Image Viewer View a Hadoop fsimage INPUTFILE using the specified PROCESSOR,saving the results in OUTPUTFILE.

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

　　次は新しいファイルに転換して格式化にします。

```
xmlstarlet fo fsimageTest.xml > fsioutput.xml
```

![image-20240717164837672](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240717164837672.png)

```
cat fsioutput.xml
```

![image-20240718162326360](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240718162326360.png)

　　最初`<version>`と`<NameSection>`部分が本節点のバージョン情報とNameNode
