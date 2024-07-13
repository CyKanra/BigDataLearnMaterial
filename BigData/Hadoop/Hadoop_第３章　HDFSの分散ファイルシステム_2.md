# 分散大規模データ処理システム -- Hadoop-4

# 第３章　HDFSの分散ファイルシステム-2

## 第４節　HDFSメタデータの管理仕組み

### 4.1　メタデータ管理の概説

#### FsimageとEdits

　　所謂のメタデータの管理が、HDFS内で既定のデータの様々な属性の記録、データ自体の記録、全ての変更操作の記録に関するものです。メタデータの記録量と記録の書き込み速さを兼ね合おうと、HDFSが二つのファイル形式を引用し、それはFsimageとEditsです。

**Fsimage鏡像ファイル**：メタデータの永続的な格納の箇所、HDFS内の全ての目録とファイルのメタデータ情報を含んでいるが、ファイルブロックの位置情報は含まれていません。ファイルブロックの位置情報はメモリ内にのみ保存されており、DataNodeがクラスタに参加したときにNameNodeがdatanodeに問い合わせて取得し、断続的に更新されます。

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

NameNode部分:

1. 最初NameNodeを初期化にして新たなFsimageとEditsファイルを作成します。その以外起動するときNameNodeが最新のFsimageとEditsファイルをメモリに読み込みます。
2. クライアントから変更操作をメタデータに実行するアクセスを受信します。
3. NameNodeが変更操作しか記録しません。全ての変更記録が先ずEditsファイルに追加していきます。一定の記録を積み重ねるまでに、元の編集されているedits_inprogress_001ファイルがedits_001に保存されて何の改修でも進行しません。その代わりにedits_inprogress_002ファイルが処理中のEditsファイルとします。一回のログローテート「log rotation」がそういう過程です。

　　NameNodeの役割が大体上記のように、纏めてのはクライアントのリクエストを受信、トランザクション記録、editsのログローテート3点であります。

Secondary NameNode部分：

1. NameNodeにログローテートさせる最初の触発点がSecondary NameNodeからCheckPoint（検査点）をする必要かどうかをNameNodeに問います。
2. 許可を得ったSecondary NameNodeがCheckPoint実行のリクエストを送信します。
3. 指令をもらったNameNodeがEditsのログローテートを進行します。
4. 最新の保存されたedits_inprogress_001とFsimageがSecondary NameNodeにダウンロードされます。
5. edits_inprogress_001とFsimageがメモリに読み込まれます。
6. edits_inprogress_001とFsimageがSecondary NameNodeのメモリに合併して新しいFsimageファイルを生み出します。一応Fsimage.chkpointと呼びます。
7. Fsimage.chkpointファイルをコピーしてNameNodeへ転送します。
8. Fsimage.chkpoint名称を改名して元のFsimageファイルを上書きます。

　　一つ完全のメタデータの維持流れが大体上記のようで、処理方法は難しくなくてかなり巧妙だと思います。第二次ログローテートするときにedits_inprogress_002を取ってだけ、Secondary NameNodeにの現存のFsimageを合併していいんです。後は、以上の処理流れが100パーセントデータの損失が保証できません。例えば、編集中のEditsファイルがログローテートしてないなのに、メモリにの状態で、NameNode停止してその部分のデータが失ってあります。

