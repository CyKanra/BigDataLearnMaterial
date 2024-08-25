# 分散大規模データ処理システム -- Hadoop-3

## 第３章　HDFSの分散ファイルシステム-1/2

### 第１節　HDFSの紹介

　HDFS（正式名称：Hadoop Distributed File System、Hadoop 分散ファイルシステム）はHadoopの中心的なコンポーネントであり、分散ストレージサービスを提供します。分散ファイルシステムは複数のコンピュータに跨り、超大規模データの保存と処理に必要な拡張性を実現します。

**重要な概念**

- Master/Slave構造

　HDFSは典型的なMaster/Slave構造を持っています。HDFSクラスタは通常、1つのNameNodeと複数のDataNodeで構成されます。NameNodeはクラスタの主節点（Master Node）であり、DataNodeは従節点（Slave Node）です。

- ブロックストレージ（block storage）

　HDFS内のファイルは物理的にブロック（block）で分割されて保存されます、ブロックのサイズは設定引数で規定できます。Hadoop 2.xでは既定のブロック大小は128Mです。

- 命名空間（NameSpace）

　HDFSは従来の階層型ファイル構造を支持しています。ユーザーやプログラムなどはディレクトリを作成し、その目録にファイルを保存できます。HDFSの命名空間の階層構造は、ほとんど一般人的なファイルシステムに似ています：ファイルの作成、削除、移動、名前変更が可能です。

　HDFSはクライアントに対して、抽象的な木構造のディレクトリを提供します。アクセス形式は`hdfs://Namenode_Hostname:port/test/input`です。 例：hdfs://centos1:9000/test/input 

　NameNodeはファイルシステムの名前空間を管理しており、ファイルシステム名前空間や属性の変更は全てNameNodeによって記録されます。 

- NameNodeのメタデータ管理

　ディレクトリ構造とファイルのブロック位置情報をメタデータと呼びます。NameNodeのメタデータは、各ファイルのブロック情報が記録します。

- DataNodeのデータストレージ

　ファイルの各ブロックの具体的なストレージ管理はDataNodeによって行われます。一つのブロックは複数のDataNodeで保存され、DataNodeは定期的に自分が保持しているブロック情報をNameNodeに報告します。

- 副本仕組み

　耐障害性を確保するために、ファイルのすべてのブロックには副本が存在します。各ファイルのブロック大小と副本数は設定可能です。HDFSは特定のファイルの副本数を指定でき、後から変更することも可能です。副本数の既定値は3です。 

- HDFSの利用に適した場合

　HDFSは一度の書き込みと複数回の読み出しに適した場面を想定して設計されています。ファイルのランダムな変更は支持していません（追記は支持されているが、ランダムな更新は支持されていません）。そのため、HDFSは大規模データ分析の基盤ストレージサービスに適していますが、オンラインストレージ（online storage）などのアプリケーションには適していません。

**HDFSの構造**

![image-20240311155255355](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240311155255355.png)

- NameNode(nn)：Hdfsクラスタの管理者、Master
  - Hdfsの名前空間（NameSpace）を管理
  - 副本情報を管理
  - ファイルブロックのマッピング情報を記録
  - クライアントの読み書き請求を処理
- DataNode：任務の実行者、Slave
  - NameNodeからの命令を受け、DataNodeが実際の操作を実行
  - データブロックを保存
  - データブロックの読み書きを担当
- Client：クライアント
  - ファイルをHDFSにアップロードする際、ClientはファイルをBlockに分割してアップロードを行う
  - NameNodeと情報をやり取りし、ファイルの位置情報を取得
  - DataNodeと情報をやり取り、ファイルの読み書きを行う
  - Clientを通じてコマンドを使用し、HDFSの管理やアクセスを実現する

### 第２節　クライアント操作

#### 2.1 Shellコマンド方式

- 二つの書き方、後ろの部分が同じで

```
hadoop fs コマンド
hdfs dfs コマンド

#同じな効果
hadoop fs -cat /wcoutput/part-r-00000
hdfs dfs -cat /wcoutput/part-r-00000
```

- 全てのコマンドを検査

```
hadoop fs
```

![image-20240409165231961](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240409165231961.png)

- 常用のコマンド

```
#具体のコマンドに関連の引数を表す
hadoop fs -help mkdir

#ディレクトリの顕示
hadoop fs -ls /

#ディレクトリを作成
hadoop fs -mkdir -p /bigdata/test

#ディレクトリを消除し、空でないディレクトリが消除られない
hadoop fs -rmdir -p /bigdata/test
```

![image-20240410155836111](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240410155836111.png)

- 本地のファイルをHDFSにアップロードし、元のファイルを消除られてある

```
vim hadoopTest.txt

hadoop fs -moveFromLocal hadoopTest.txt /bigdata/test
```

![image-20240410161036302](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240410161036302.png)

- 目標ファイルをhadoopTestの中に追加し、元のファイルが保留される

```
vim hdfsTest.txt
hadoop fs -appendToFile hdfsTest.txt /bigdata/test/hadoopTest.txt

#全てのファイルを表し
hadoop fs -cat /bigdata/test/hadoopTest.txt

#ファイルの末尾内容を表す
hadoop fs -tail /bigdata/test/hadoopTest.txt
```

![image-20240410162514743](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240410162514743.png)

- ファイルをコピーして本地又はHDFSに発送し、二つの書き方が一様です

```
hadoop fs -copyFromLocal hdfsTest.txt /bigdata/test/
hadoop fs -put hdfsTest.txt /bigdata/test/

hadoop fs -copyToLocal /bigdata/test/hadoopTest.txt /root
hadoop fs -get /bigdata/test/hadoopTest.txt /root
```

![image-20240410192341567](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240410192341567.png)

```
#HDFSファイルをコピーすると別のアドレスに移動し
hadoop fs -cp /bigdata/test/hdfsTest.txt /bigdata

#HDFSファイルを消除
hadoop fs -rm /bigdata/hdfsTest.txt

#HDFSファイルの移動
hadoop fs -mv /bigdata/test/hdfsTest.txt /bigdata
```

![image-20240410200720453](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240410200720453.png)

```
#直接にHDFS上にファイルを作成
hadoop fs -touchz /bigdata/test/touchTest.txt
```

![image-20240415081436750](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240415081436750.png)

　ファイルの作成のコマンドあるけれど、HDFS上に書き込みのコマンドがありません。この点から見えてファイルの変更におけての支持が上手くない、ただ整体の大小の変わりで、データ内容の変更に係わらず場合が適用です。

```
#HDFSの権限管理コマンドがlinuxと同じで
hadoop fs -chmod 666 /bigdata/test/hadoopTest.txt
hadoop fs -chown root:root /bigdata/test/hadoopTest.txt
hadoop fs -chgrp root /bigdata/test/hadoopTest.txt
```

![image-20240415131401790](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240415131401790.png)

```
#ファイルの大小を表し
hadoop fs -du /bigdata/test/hadoopTest.txt
```

![image-20240415131023319](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240415131023319.png)

```
#ファイルの副本数を設定
hadoop fs -setrep 10 /bigdata/test/hadoopTest.txt
```

![image-20240415131803181](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240415131803181.png)

![image-20240415131835586](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240415131835586.png)

![image-20240415131845199](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240415131845199.png)

　　ここで設定された副本数はNameNodeのメタデータに記録されるだけで、実際にその数量の副本が存在するかどうかはDataNode数に依存します。現在はサーバが4台しかないため、最大で4つの副本しか持てません。節点数が10台に増えた場合にのみ、副本数は10に達することができます。

#### 2.2 JAVAクライアント

　　JAVAで外部からHadoopを使いにかかり内容が多すぎため、ここで一つの入門案例を挙げてだけ、どうJAVAを通してHadoopクラスタを使用することを表します。

- Maveプロジェクトに下の依頼を追加

```
<dependencies>
	<dependency>
		<groupId>junit</groupId>
		<artifactId>junit</artifactId>
		<version>RELEASE</version>
	</dependency>
	<dependency>
		<groupId>org.apache.logging.log4j</groupId>
		<artifactId>log4j-core</artifactId>
		<version>2.8.2</version>
	</dependency>
	<dependency>
		<groupId>org.apache.hadoop</groupId>
		<artifactId>hadoop-common</artifactId>
		<version>2.9.2</version>
	</dependency>
	<!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-client-->
	<dependency>
		<groupId>org.apache.hadoop</groupId>
		<artifactId>hadoop-client</artifactId>
		<version>2.9.2</version>
	</dependency>
	<!-- https://mvnrepository.com/artifact/org.apache.hadoop/hadoop-hdfs -->
	<dependency>
		<groupId>org.apache.hadoop</groupId>
		<artifactId>hadoop-hdfs</artifactId>
		<version>2.9.2</version>
	</dependency>
</dependencies>
```

　　デバッグしやすいように、`src/main/resources`目録に`log4j.properties`ファイルを作成して完全の運行ログを出せます。

```
log4j.rootLogger=INFO, stdout
log4j.appender.stdout=org.apache.log4j.ConsoleAppender
log4j.appender.stdout.layout=org.apache.log4j.PatternLayout
log4j.appender.stdout.layout.ConversionPattern=%d %p [%c] - %m%n
log4j.appender.logfile=org.apache.log4j.FileAppender
log4j.appender.logfile.File=target/spring.log
log4j.appender.logfile.layout=org.apache.log4j.PatternLayout
log4j.appender.logfile.layout.ConversionPattern=%d %p [%c] - %m%n
```

![image-20240416105022013](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240416105022013.png)

- メソッドの表し

```
public class HdfsClient {

        private FileSystem fs = null;
        
        @Before
        public void start() throws URISyntaxException, IOException, InterruptedException{
            Configuration configuration = new Configuration();
            configuration.set("fs.defaultFS", "hdfs://192.168.31.135:9000");
            fs = FileSystem.get(new URI("hdfs://192.168.31.135:9000"), configuration, "root");
        }
        
        @Test
        public void testMkdirs() throws IOException {
            fs.copyFromLocalFile(new Path("D:/log_info.log"), new Path("/bigdata/test"));
        }
        
        @After
        public void end() throws IOException {
            fs.close();
        }
}
```

　　それはファイルのアップロードを実現する案例で、全ての流れを理解が難しくないと思い、体大の観念を形成されてきたら入門については十分です。

### 第３節　HDFSの読み書き操作

#### 3.1 HDFS読み込み流れ

　　下の図はあるファイルを読み込んでダウンロードするの流れです。仮に読み込みの目標ファイルが200Mサイズにして、128Mと72Mに分割されて少なくとも二つのブロックで格納されました。次は200M場合でHDFSの具体の読み込み流れを説明しましょう。

![061114_0923_LearnHDFSAB1](D:\OneDrive\picture\Typora\BigData\Hadoop\061114_0923_LearnHDFSAB1.webp)

**step1**：クライアントがDistributed FileSystemのインスタンスを作成します（①）。このインスタンスはNameNodeに請求を発出して目標データのブロック情報や関連のアドレスなどを取得する作用があります。その重要な情報が全てメタデータに属してNameNodeに管理されます。

**step2**：目標ファイルのブロックの情報、対応の副本情報、格納されるDataNode情報などをNameNodeのメタデータから獲得します（②）。若しブロック数量が多すぎで、一部分のブロック情報を先に返却されるという方式を選択する可能性もあります。最後に完全の目標ファイルに継ぎ合わせるために各ブロック情報の返却が一定の順序で並べされます。

**step3**：FSDataInputStreamインスタンスを作成します（④）、その中のread()メソッドを呼び出してストリーム方式で目標節点に請求を発出してブロックデータを返却します。

**step4**：最初に200Mのファイルが128Mと72Mに分割されるため、せめて2回DataNodeからブロックデータを獲得する流れが必要です。ただ、ブロックごとの多数の副本が存在すると考えては別のDataNodeからデータも取得できます。

![image-20240424220112529](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240424220112529.png)

　　どっちの節点を選択してのがDataNodeの最短距によって決まります。所謂のDataNode最短距離がDataNodeのネットワーク節点の最短距離です。例えば、同一サーバラック（server rack）内の節点に比べて他のサーバラック又はデータセンター（data center）がより長い距離です。ただ、そのサーバラック感知の機能が既定情況で閉じ、サーバラック感知策略に関する変数設定と脚本が必要です。ちょっと複雑なのでここで省きます。

　　その最短距離の副本を選択の流れが実はstep2にも終わりました。つまり、最良のブロック情報が一つだけを返却されます。もし当のブロックの取得が何か理由に失敗したら、改めてNameNodeと通信を立てて新しいメタデータを取得します。故障DataNodeも標記された次のデータ読み込みで当のDataNodeを二度と使いません。

![image009](D:\OneDrive\picture\Typora\BigData\Hadoop\image009.jpg)

**step5**：一番目の128Mブロックを取得するのを完成したら（⑤）、連接を閉じて二つ目の72Mブロックの取得を続けます（⑥）。データ伝送の過程中にクライアントがPacket（64kB）の伝送基本単位としてブロックデータを受け、その同時にデータの検証も行っています。

　　検証の最小の基本単位がChunk（512B）です。実際には、検証には4Bの検証値が含まれているため、Packetへの書き込みは516Bになります。データと検証値の比率は128:1であるため、128Mのブロックには1Mの検証ファイルが対応します。

　　戻り値不完全とかDataNode故障とか、当のDataNodeとの通信を直ぐに停止します。新しいブロックデータを取得するために上記の流れを繰り返させます。

**step6**：受けたブロックデータが先ず本地サーバにキャッシュされて、後で永続化にして実際ファイルに書き込みます。全てのブロック取得が終わったら、FSDataInputStreamインスタンスがclose()メソッドを呼び出してデータストリームを閉じて、NameNodeに通知していくことが必要ありません（➆）。

　　ここではHDFS読み込み流れが紹介し終わります。下記は流れのソースコードの展示で参考できます。

```
@Test
 public void testCopyToLocalFile() throws IOException, InterruptedException,　URISyntaxException{
	Configuration configuration = new Configuration();
 	FileSystem fs = FileSystem.get(new URI("hdfs://centos01:9000"), configuration, "root");
 }
	fs.copyToLocalFile(false, new Path("/bigdata/test/hadoopTest.txt"), new 
	Path("D:/hadoopTest.txt"), true);
	fs.close();
}
```

#### 3.2 HDFS書き込み流れ

　　HDFSの書き込み流れがちょっと複雑で、ここまだ200M目標ファイルを例をしてせめて二回のアップロード流れがあると説明しています。

![061114_0923_LearnHDFSAB2](D:\OneDrive\picture\Typora\BigData\Hadoop\061114_0923_LearnHDFSAB2.webp)

**step1**：先ず依然クライアントがDistributed FileSystemのインスタンスを作成します（①）。このインスタンスはNameNodeに請求を発出して目標ファイルをアップロードする可否を確認します（②）。

**step2**：NameNode がユーザーのファイル書き込みRPC要求を受け取ると、様々なチェックを実行します。例えば、ユーザーが関連する作成権限を持っているか、またそのファイルが既に存在しているかなどを確認します。これらのチェックに合格した場合、新しいファイルが作成され、その操作がトランザクションログに記録されます。

**step3**：アップロード許可を獲得した後でFSDataOutputStreamインスタンスを作成され、200M目標ファイルを幾つかのブロックに分割すると始めます。

　　書き込み流れが一括に全部ファイルを書き込むことではありません。1番目のブロックを分割されて出てから、対応の格納アドレスと副本アドレスを獲得して順序に各DataNodeにデータを発送したまでは、一つの連続、完全のアップロード流れです。前の終わりだ初めて、次の2番目のブロックのアップロード流れを始めて進行します。

　　その対応の格納アドレスを獲得する働きをDataStreamerから担当します（⑤）。読み込みと同じでブロックがパケット（packet）に分割される形式でData Queue隊列に格納されて目標のDataNodeに発送して行きます（④）。

　　副本の選択は読み込みと類似の放置策略があります。最初の副本はできるだけデータを書き込む節点に配置し、二番目の副本は最初のレプリカとは異なるラック（rack）にある節点に配置し、三番目の副本は二番目の副本と同じラックに配置します

**step4**：データを書き込む前に先ずクライアントのFSDataOutputStreamが各節点が正常かどうか確認します。クライアントは最前の1番目の節点に請求を発送し、クライアントに返却を受信すると1番目の節点の存在を確認しました。その後1番目の節点が2番目の節点に請求を発送します。正常に受信したの2番目の節点がクライアントに応答してあげます。最後の節点まで確認したと一つの完全の通信チャンネルが建てました。その通信チャンネルがData Pipelineとも呼ばれます（⑥）。

　　もしその逐次の応答にある節点が故障かと判断したら、改めてNameNodeから新しい節点リストを獲得し、故障の節点が不可用マークを付けます。

**step5**：正式にブロックのアップロードが開始します（➆）。DFSOutputStreamはパケットをData Pipeline最初のDataNode節点へ書き込みます。最初の節点はパケットを受信して保存し、Data Pipelineにの次の節点にパケットを発送します。同様に、2番目の節点は受信してパケットを保存したら、それをData Pipelineの3番目の節点に書き込みます（⑧）。

　　最後のDataNode節点が正常に当のパケットを受信したら、前の発送者節点に成功の確認情報を返却します。逐次に前の節点に報告してクライアントのDFSOutputStreamが1番号の節点から返却信号を受信したまで、一つの完全の通信回路が形成されます（⑩）。

　　パケットもDFSOutputStreamの内部に専門の応答隊列（ack queue）に維持することがあります。直ぐに各節点に発送られるパケットがこの応答隊列にData Queue隊列から転移します。FSDataOutputStreamが正常に返事を受信してack queueからそのパケットを削除します（⑨）。

**step6**：上記の様にパケット（packet）を単位として一つ一つで各節点に転送します。一つの完全のブロックがアップロードした初めて、クライアントがNameNodeに2番目のブロックのアクセスを請求します。残りの過程がstep4とstep5の流れを繰り返すんです。

　　なら、データ書き込みの過程でPipelineの中のあるDataNode節点が書き込みに失敗した場合はどうしますか。

　　先ず、Pipelineのデータストリーム（Data Stream）が閉じられます。応答隊列（ack queue）にそのパケットをData Queue隊列に帰還して隊列の最前に追加します。パケットのデータ損失を防ぎます。

　　次、正常なDataNode節点に保存されたブロックのバージョンIDがアップグレードされます。これにより、故障のDataNode節点が一度正常に戻ったら不一致のバージョンIDをNameNodeに発見され、NameNodeが故障のDataNode節点に回復処理を行います。誤りバージョンIDにのデータを消除したり、不足の更新記録を追加したりすると節点の一致性を保証します。新しいバージョンIDの下に失敗した節点がPipelineから削除されます。

　　最後に、前の流れを続けて残りのデータをPipeline内の他の2つの節点に書き込みます。

　　Pipeline内の複数の節点がデータ書き込みに失敗した場合でも、成功したブロックの数が `dfs.replication.min`（既定値1）に達すれば、書き込みは成功したと見なされます。その後、NameNodeはブロックを他のノードにコピーすると、副本数が`dfs.replication`設定値に達してまで処理を行います。

**step7**：一番目のブロックをアップロードが終了だと、DataStreamが再びNameNodeに請求を発出します。上記の操作を繰り返して全てのブロックを各節点に書き込む後は、クライアントclose()メソッドを呼び出してIOストリームを閉じます（⑪）。

**step8**：NameNodeに知らせて書き込み流れが完成しました。NameNodeが今回のログとメタデータなどに最新の情報を書き込んで現在のデータを更新します。

　　ここではHDFS書き込み流れが紹介し終わります。下記は流れのソースコードの展示で参考できます。

```
@Test
public void putFileToHDFS() throws IOException, InterruptedException, URISyntaxException {
    Configuration configuration = new Configuration();
    FileSystem fs = FileSystem.get(new URI("hdfs://centos01:9000"), configuration, "root");
    FileInputStream fis = new FileInputStream(new File("D:/hadoopTest.txt"));
    FSDataOutputStream fos = fs.create(new Path("/bigdata/test/hadoopTest.txt"));
    IOUtils.copyBytes(fis, fos, configuration);
    IOUtils.closeStream(fos);
    IOUtils.closeStream(fis);
    fs.close();
}
```

