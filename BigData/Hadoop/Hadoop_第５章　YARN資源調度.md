# 分散大規模データ処理システム -- Hadoop-10

# 第５章　YARN資源調度

## 第１節　YARNフレームワーク

![image-20250827164414246](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250827164414246.png)

**ResourceManager（RM）**：
　クライアントからのリクエストを処理し、ApplicationMasterの起動・監視、NodeManager の監視、リソースの割り当てとスケジューリングを行う。

**NodeManager（NM）**：
　単一ノード上のリソースを管理し、ResourceManagerからの命令やApplicationMasterからの命令を処理する。

**ApplicationMaster（AM）**：
 　データを分割し、アプリケーションのためにリソースを要求し、内部タスクへ割り当て、さらにタスクの監視と障害対応を行う。

**Container**：
 　任意のタスクを実行するための抽象的な環境。CPU、メモリなどの複数のリソースや環境変数、起動コマンドなど、タスク実行に必要な情報をカプセル化して容器に纏める。

## 第２節　YARN運行仕組み

![image-20260207125404012](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260207125404012.png)

**Taskの提出**（1-5）

- WordCount計算案例に`JavaPractice-1.0.0-SNAPSHOT.jar`運行パッケージがあり、それをあるノードにアップロードする。
- HDFSコマンドでパッケージを運行し、内部的には `YarnRunner` が呼び出され、ジョブの提出処理が始まる。その時、ResourceManagerへ計算任務Applicationの申請を提出する。
- ResourceManagerがApplicationIDと、HDFS上の一時ディレクトリのパスを返す。例えば、`hdfs://.../.staging/application_xxxx`。
- ノードはジョブに必要なファイルを先ほどのディレクトリにアップロードする。`job.split`分割情報、`job.xml`ジョブの設定情報、`wc.jar`運行パッケージを含む。
- 運行ファイル提出が完成したらResourceManagerに通知し、当のノードのApplicationMasterを起動する申請を提出する。

**ジョブの初期化**（6-9）

- RMは、申請されたApplicationを1つのTaskとして初期化し、余力があるノードを決める。
- NodeManagerはRMからTaskを受ける。
- NodeManagerがAppMasterを動かすためのContainerを作成する。
- NodeManagerがHDFSからジョブ資源をダウンロードする。そこまでAppMasterがジョブ実行の準備は完成した。

![image-20250828172350879](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250828172350879.png)

**タスク割り当て**（10-11）

- AppMasterはRMに複数の運行資源の要求を提出する。
- ジョブの初期化の流れを繰り返す。複数の初期化されたAppTaskが隊列に読み込む。後でNodeManagerに順次に割り当てる。
- 別のノードが要るかどうかのは`Job.split`の切片情報によって決める。

**タスク運行**（12）

- AppMasterがNodeManagerに対してMapTask実行の起動スクリプトを送信する。各 NodeManagerはContainerを起動し、その中でMapTaskを実行する。
- 全MapTaskが完了したら、その出力結果はAppMasterに報告する。AppMasterは再度RMにReduceTask用Containerの資源を申請する。

![image-20251031113704010](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20251031113704010.png)

**ReduceTask資源の申請**（13）

- RMに対してReduceTask用のContainerを申請する。
- RMはクラスタ全体のNodeManagerの状態を確認し、空きのあるノードにContainerを優先的に割り当てる。

**ReduceTaskの実行開始**（14-15）

- RMからの指示を受けたNodeManagerは、ReduceTaskのContainerを作成し、その中でReduceTaskプロセスを起動される。
- ReduceTaskは、AppMasterを通じてすべてのMapTask出力ファイルの所在を問い合わせる。
- この時点で自分の担当するパーティションのデータを、各NodeManager上のMapTask出力からネットワーク経由でダウンロード (Shuffle)する。
- 因みに、その自分の担当するパーティション数はジョブの最初に決まっていた。

**ジョブの終了**

- 各タスクはNodeManagerに完了を報告し、AppMasterがそれを集約して状態を確認する。
- 「ジョブ完了」通知を受けたResourceManagerはAppMasterの登録を解除する。
- NodeManagerがAppMasterのコンテナも含め、一時ファイルや作業ディレクトリなどを削除し、メモリやCPUなどのリソースが解放される。

**進捗の確認**

　ジョブを実行してる間に進捗を追って出力、管理するが要る。大体4つのところがある。

- Task → AppMaster：各NodeManager上のTaskがAppMasterに進捗を報告する。
- AppMaster → RM：ジョブ全体の状態を報告する。
- Client → AppMaster：定期ポーリングで最新状態を取得する。
- Client → User：コンソールに進行率やタスク件数を出力する。

　Yarnはジョブの進捗と状態をAppMasterに報告する。クライアントはAppMasterにリクエストを発送し、取得する情報をユーザに展示する。リクエストの間隔はmapreduce.client.completion.pollinterval変数でコントロールできる。

　上述の方式を除いて、クライアントが手動でwaitForCompletion()メソッドを呼び出して進捗と状態をチェックできる。

　ジョブが終わったらAppMasterとContainer が清理される。実行後のMapReduceジョブ状態を構造化してJobHistoryServerに格納される。

## 第２節　YARN調度策略

　Hadoop / YARNの代表的な3つのスケジューラはFIFO、Capacity Scheduler和Fair Schedulerある。Hadoopデフォルト状態はCapacity Scheduler策略です。

　yarn-default.xml設定ファイルに「yarn.resourcemanager.scheduler.class」変数は対応のスケジューラをコントロールできる。

![image-20260115063839733](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260115063839733.png)

**FIFO Scheduler**

　一番理解しやすいスケジューラで、投入順に先に来るジョブを先ず実行される。小規模・単一ユーザー環境向け、本番環境に似合わない。大きなジョブを実行するとき他の待たなければならない。学習用途、検証環境などこのモードをお勧めする。

![image-20260115065739691](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260115065739691.png)

**Capacity Scheduler**

　Apache Hadoopのデフォルトのスケジューラです。クラスタを待ち行列（Queue、キュー）に分割し、各キューに最低保証容量（Capacity）を割り当てる。空きがあれば他キューが一時的に利用可能です。そうすると同時に複数のジョブを処理することができる。

　単独の1つキューにはFIFO策略で処理する。また各キューの下に、さらにサブキューが作られることもあって資源を分けられる。

![image-20260116065451201](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260116065451201.png)

 **Fair Scheduler**

　公平スケジューラは各ジョブに資源を公平に分けるとする。

　例えば、ユーザAだけがジョブを実行し、クラスタ資源の100%が割り当てられる。ユーザBはジョブを1つ投入してユーザAから50％資源を引き出してユーザBに割り当てる。次はユーザBがもう1つジョブ2を投入して続ける。Bの取り分を分割してジョブごとに25%資源を持っている。

　注意のところは Fair Schedulerは完全にジョブ数で資源を割り当てて、ユーザとジョブを併用して分割される。

#### スケジューラの設定

　スケジューラの設定はちょっと複雑、どのスケジューラ策略を使うはyarn-site.xmlファイトに設定する。スケジューラ下の具体的なパラメータは専用の設定ファイルに書く。

**Capacity Scheduler**

　下記の設定をyarn-site.xmlファイルに追加する。でもCapacity Schedulerは元々デフォルト設定で、何も書かなくていい。

```
<!-- capacity -->
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.capacity.CapacityScheduler</value>
</property>
```

　詳細のはcapacity-scheduler.xmlに設定できる。

![image-20260211102958955](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260211102958955.png)

　capacity-scheduler.xmlに設定パラメータがあり、説明によって変更できる。

![image-20260211103255740](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20260211103255740.png)

**FIFO Scheduler**

　専用のfifo-scheduler.xmlファイルが不要、単純に「先着順」で回すだけなので、細かい設定がほとんど無い。

```
<!-- FIFO -->
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fifo.FifoScheduler</value>
</property>
```

 **Fair Scheduler**

　複数ユーザーの間に資源の隔離を設定したい場合があっては、Fair Scheduler策略しか選ばない。

　以下はyarn-site.xmlにの設定、専用のfair-scheduler.xmlを新規する必要です

```
<!-- Fair Scheduler を使う -->
<property>
    <name>yarn.resourcemanager.scheduler.class</name>
    <value>org.apache.hadoop.yarn.server.resourcemanager.scheduler.fair.FairScheduler</value>
</property>

<!-- Fair Scheduler の設定ファイル -->
<property>
    <name>yarn.scheduler.fair.allocation.file</name>
    <value>/etc/hadoop/conf/fair-scheduler.xml</value>
</property>

<!-- キュー（pool）が無ければ自動作成 -->
<property>
    <name>yarn.scheduler.fair.allow-undeclared-pools</name>
    <value>true</value>
</property>
```

　fair-scheduler.xml内容の案例

```
<?xml version="1.0" encoding="UTF-8"?>
<allocations>
    <!-- 通用設定、全員向け -->
    <queueMaxAppsDefault>50</queueMaxAppsDefault>
    <userMaxAppsDefault>20</userMaxAppsDefault>

    <!-- 指定ユーザーの設定 -->
    <queue name="root">
        <!-- ユーザーuserA：優先度高、最低保証あり-->
        <queue name="userA">
        
        	<!-- 優先度高いほうが最低保証あり -->
        	<priority>5</priority>
            <minResources>4096mb,4vcores</minResources>
            
            <!-- 複数のユーザー競争関係で、資源の占有率75% -->
            <weight>3.0</weight>
            
            <!-- 最大のジョブ数30 -->
            <maxRunningApps>30</maxRunningApps>
        </queue>

        <!-- ユーザーuserB：最低保証は小さめ、でも空いてたら使える -->
        <queue name="userB">
        
        	<!-- 優先度低いほうは最低保証弱い -->
        	<priority>1</priority>
            <minResources>1024mb,1vcores</minResources>
            
            <!-- 資源の占有率25% -->
            <weight>1.0</weight>
            
            <!-- 最大のジョブ数10 -->
            <maxRunningApps>10</maxRunningApps>
        </queue>
    </queue>
</allocations>
```



ここまでは、Hadoopの講解が終わりです。

大体Hadoopの基礎知識がほとんど含まれ、一応ここに終えて次の主題に入る。

以後、Hadoopの実戦の講解あれば更新を続ける。

よろしくお願いいたします。
