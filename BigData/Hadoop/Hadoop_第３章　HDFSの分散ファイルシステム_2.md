# 分散大規模データ処理システム -- Hadoop-3

## 第３章　HDFSの分散ファイルシステム-2/4

### 第３節　HDFSの読み書き操作

#### 3.1 HDFS読み込み流れ

　下の図は、ファイルを読み込んでダウンロードする流れを示しています。仮に、読み込み対象のファイルが200Mサイズであれば、128Mと72Mに分割され、少なくとも2つのブロックに格納されます。それでは、200MBのファイルの場合におけるHDFSでの具体的な読み込みの流れを説明しましょう。

![061114_0923_LearnHDFSAB1](D:\OneDrive\picture\Typora\BigData\Hadoop\061114_0923_LearnHDFSAB1.webp)

**step1**：クライアントがDistributed FileSystemのインスタンスを作成します（①）。このインスタンスはNameNodeにリクエストを送信し、対象データのブロック情報や関連するアドレスを取得します。これらの重要な情報がすべてメタデータとしてNameNodeによって管理されます。

**step2**：対象ファイルのブロックの情報、対応する副本情報、格納されるDataNode情報などをNameNodeのメタデータから取得します（②）。ブロック数量が多すぎる場合、一部分のブロック情報を先に返却されるという方式が選択られる可能性もあります。最後に、ファイルを完全な状態に復元するため、各ブロック情報は一定の順序で継ぎ合わせされます。

**step3**：FSDataInputStreamインスタンスを作成します（④）、その中のread()メソッドを呼び出して、ストリーム形式で対象節点に請求を発出し、ブロックデータを取得します。

**step4**：200Mのファイルが128Mと72Mに分割されているため、少なくとも2回のDataNodeからブロックデータを獲得する流れが必要です。もちろん、各ブロックは複数の副本が存在するため、異なるDataNodeからデータも取得することも可能です。

![image-20240424220112529](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240424220112529.png)

　　どの節点を選択するかは、DataNodeの最短距離によって決まります。ここで言う「最短距離」とは、DataNode間のネットワーク上の距離を指します。例えば、同一サーバラック（server rack）内の節点に比べて他のサーバラック又はデータセンター（data center）が長くなります。ただ、そのサーバラック感知の機能が既定情況で閉じ、サーバラック感知策略に関する変数設定と脚本が必要です。ちょっと複雑なのでここで省きます。

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
