# 分散大規模データ処理システム -- Hadoop-5

# 第４章　MapReduce計算フレームワーク-1/3

　本章に入る前に、まずはMapReduceの利用状況について少し触れておく。

　Hadoopにおいては、MapReduceは非常に基本的かつ代表的な分散計算フレームワークであり、関連の知識が相当多いです。しかし、実際の運用では、MapReduceが直接使われるケースがそれほど多くない。理由は、より高性能で柔軟な処理基盤である Spark や Flink などのフレームワークに置き換えられつつある。また、MapReduceコーディングはやや複雑で、ビッグデータにおける要求を効率的に満たすことにならない。

　この本章は、主にMapReduceコーディングを読み取れるようになること、そして運行仕組みを理解することを目的としてする。コードを一から書くことは求めない。

　本章では、MapReduceのソースコードを共に仕組みや概念を説明していく。先にコードの解説を行い、基本的な理解を得た上で一定の概念があると、内部の運行仕組みと設計について理解しやすくなるでしょう。

## 第１節　MapReduceの紹介

　MapReduceは、Hadoopにおける分散計算フレームワークとして、日常的なデータ処理タスクを担う役割を果たす。その思想の核心は「分割（Map）」と「統合（Reduce）」という2つ策略がある。それを活用するんで、並行処理の特性を生かし、大規模データの効率的な処理を実現している。

MapReduceの処理流れは、以下の2段階に分けている：

- Map段階：入力データを複数の部分に「分割」し、個別の処理単位に分解する。これにより、複雑なタスクを複数の独立したタスクとして並列実行できるようになる。

- Reduce段階：Mapの結果を受け取り、分散されたデータを「統合」する。必要な処理を行った後、最終的な結果を出力する。

![Hadoop - Architecture - GeeksforGeeks](D:\OneDrive\picture\Typora\BigData\Hadoop\mapreduce-workflow.png)

## 第２節　WordCount公式案例にいて

　MapReduceの基本的な処理モデルを理解するために、文字列中の単語の出現回数を数えて統計するWordCount案例を紹介する。

　この機能は公式のサンプルはパッケージ`hadoop-mapreduce-examples-2.9.2.jar`に含まれる。通常`/opt/bigdata/servers/hadoop-2.9.2/share/hadoop/mapreduce`ディレクトリにいる。

![image-20240808152733378](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808152733378.png)

　このJarファイルをHadoopサービスからダウンロードし、Ideaに導入して利用することができる。WordCountクラスはその実行ロジックである。

![image-20240808153229676](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808153229676.png)

　WordCountコードは、以下の3つの主要部分で構成されている。

**Map階段**

- Mapper親クラスを継承して親クラスのmapメソッドを書き直して分割ロジックを記述する。

![image-20240808160200736](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240808160200736.png)

- 入力値keyが1行文字のオフセット（offset）で、行数と理解していい、特に使われてない。
- 入力値valueがその行の文字列内容です。

- ロジックとしては、1行の文字列を空白で分割し。複数の単語を含める結果値`itr`がループ処理をする。単語ごとに`<key：単語, value：出現回数=1>`形式で`context`に書き込む。

**Reduce階段**

- Reducer親クラスを継承して親クラスのreduceメソッドを書き直す統合処理のロジックを記述する。

![image-20240811153315099](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240811153315099.png)

- 入力値keyがMap階段から出力のkey値で、valuesがその単語に対応する出現回数の反復子（Iterable）です。例えば、`[1,1,1]`と理解していい。

　何で反復子には1で、それはMapメソッドに`<key：単語, value：出現回数=1>`を出力するため、`<key,1>`を収集してvalues集合をなして反復子に入れる。

- 単語keyに対応する総数`＜key：単語, sum：総数＞`形で`context`に入れる。最後に出力結果をファイルに書き込む。

**Driver**

- 主プロセス。配置の実装、Job対象の設定、入力と出力ファイルのアドレスの添加と、最後のJobタスクの実行などをまとめて定義する。
- 最後に `job.waitForCompletion(true)` を呼び出すんで、MapReduceジョブの実行が開始される。

![image-20240812165236008](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240812165236008-1723449222995-1.png)

　MapReduce開発では、開発者が主にMapメソッドとReduceメソッドのロジック実装に集中すればよく、データの分割や統合処理の詳細な過程はフレームワーク側が自動的に処理し、触れることは必要ない。

## 第３節　手動的にWordCount機能を実現

　MapReduceコーディングの規範に従って自らWordCount機能のプログラムを実現します。IdeaツールでMavenプログラムを新作してMapper，Reducer，Driverコーディングをします。

**実現目標**

　指定のコンテストにの単語の出現回数を統計します。

入力ファイル：wc.txt

出力ファイル：大体下記のようです

```
Bigdata	2
Fink	3
HBase	2
Hive	2
```

**環境準備**

- HADOOP_HOME環境変数の設定

　hadoop-2.9.2インストールパッケージがダウンロードして、あるディレクトリに置いて解凍を実行します。

hadoop-2.9.2アドレス：[Apache Hadoop](https://hadoop.apache.org/release/2.9.2.html)

![image-20240815140854345](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815140854345.png)

　win10システムなら`Environment variables`を検索して環境変数の設定画面が出ています。下の`System variables`に`HADOOP_HOME`と`E:\Program Files\hadoop-2.9.2`新規をします。

![image-20240815141111477](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815141111477.png)

　次は、`System variables`に`Path`選択肢に入って`%HADOOP_HOME%\bin`アドレスを追加します。

![image-20240815141210278](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815141210278.png)

　`D:\InstallPackage\hadoop-2.9.2\etc\hadoop\hadoop-env.cmd`ファイルにJAVA_HOMEアドレスを追加します。

![image-20240815141430567](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815141430567.png)

　Hadoopサービスをwinにインストールするかどうか確認します。

```
hadoop version
```

![image-20240815141607264](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815141607264.png)

> HADOOP_HOMEのアドレスにもhadoop-env.cmdファイルのJAVA_HOMEにも、スペースが許さない。JAVA_HOMEに対してのアドレスが存在する限り、環境変数にのJAVA_HOMEと不一致にしても構わない。

**コーディング**

- Hadoopプログラムの構築に入って、Maven依頼を導入

```
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-common</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-client</artifactId>
    <version>2.9.2</version>
</dependency>
<dependency>
    <groupId>org.apache.hadoop</groupId>
    <artifactId>hadoop-hdfs</artifactId>
    <version>2.9.2</version>
</dependency>
```

- Mapper階段

```
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.LongWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Mapper;
import java.io.IOException;

public class WordCountMapper extends Mapper<LongWritable, Text, Text, IntWritable> {
    Text text = new Text();
    IntWritable intWritable = new IntWritable(1);
    @Override
    protected void map(LongWritable key, Text value, Mapper<LongWritable, Text, Text, IntWritable>.Context context)
            throws IOException, InterruptedException {
        //一行文字を取得
        String str = value.toString();
        //文字を区切って単語数組になる
        String[] strs = str.split(" ");
        //出力
        for (String s : strs) {
            text.set(s);
            context.write(text, intWritable);
        }
    }
}
```

- Reducer階段

```
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Reducer;

public class WordCountReducer extends Reducer<Text, IntWritable, Text, IntWritable> {
    Integer sum;
    IntWritable intWritable = new IntWritable();
    @Override
    protected void reduce(Text key, Iterable<IntWritable> values, Reducer<Text, IntWritable, Text, IntWritable>.Context context) 
            throws IOException, InterruptedException {
        sum = 0;
        //累積加算
        for (IntWritable value : values) {
            sum += value.get();
        }
        //出力
        intWritable.set(sum);
        context.write(key,intWritable);
    }
}
```

- Driver階段

```
import java.io.IOException;
import org.apache.hadoop.conf.Configuration;
import org.apache.hadoop.fs.Path;
import org.apache.hadoop.io.IntWritable;
import org.apache.hadoop.io.Text;
import org.apache.hadoop.mapreduce.Job;
import org.apache.hadoop.mapreduce.lib.input.FileInputFormat;
import org.apache.hadoop.mapreduce.lib.output.FileOutputFormat;

public class WordcountDriver {
    public static void main(String[] args) throws IOException,
            ClassNotFoundException, InterruptedException {
        //配置ファイルを設定とJobを作成
        Configuration configuration = new Configuration();
        Job job = Job.getInstance(configuration);
        //Mapper、Reducer、Driverクラスを添加
        job.setJarByClass(WordcountDriver.class);
        job.setMapperClass(WordCountMapper.class);
        job.setReducerClass(WordCountReducer.class);
        //Mapの出力値のタイプ
        job.setMapOutputKeyClass(Text.class);
        job.setMapOutputValueClass(IntWritable.class);
        //最終出力値のタイプ
        job.setOutputKeyClass(Text.class);
        job.setOutputValueClass(IntWritable.class);
        //入力と出力ファイルのアドレス
        FileInputFormat.setInputPaths(job, new Path(args[0]));
        FileOutputFormat.setOutputPath(job, new Path(args[1]));
        //タスクをコミット
        boolean result = job.waitForCompletion(true);
        System.exit(result ? 0 : 1);
    }
}
```

**プログラム運行（ローカル）**

- プログラムを運行

　ここでローカル運行を選べて分散サービスを依頼しません。入力と出力ファイルのアドレス引数をprogram arguments欄に添加します。因みに、出力アドレスが必ず存在しません。

![image-20240814150354113](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240814150354113.png)

![image-20240816095735458](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240816095735458.png)

　wc.txt内容が下記です。

```
hadoop Zookeeper Hive mapreduce yarn HBase
hdfs Spark Zookeeper hadoop mapreduce
mapreduce Fink yarn nodemanager Hive
NameNode nodemanager Spark
Fink Bigdata ResourceManager Bigdata Zookeeper
HBase Zookeeper Spark Fink
```

- 環境の配置

　その前にWindows環境の下で`D:\InstallPackage\hadoop-2.9.2\bin`にwinutils.exe、hadoop.dllというファイルが必要です。お勧め方法がネットから標準のをダウンロードしてローカルを上書きします。

ダウンロードURL：[GitHub - cdarlint/winutils: winutils.exe hadoop.dll and hdfs.dll binaries for hadoop windows](https://github.com/cdarlint/winutils)

　需要のディレクトリだけをダウンロードしてほしいと、`Git Bash here`を開けて下記のコマンドを入力します。

![image-20240816100042378](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240816100042378.png)

```
git clone --no-checkout https://github.com/cdarlint/winutils.git
#或いは
git clone --no-checkout https://github.com/cdarlint/winutils.git <ローカルディレクトリ>

#最上層のディレクトリに入って
cd winutils

git sparse-checkout init --cone

git sparse-checkout set hadoop-2.9.2/bin

git checkout
```

![image-20240815162659093](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815162659093.png)

　下図にのbinディレクトリを`D:\InstallPackage\hadoop-2.9.2`に上書きします。

![image-20240815163522007](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815163522007.png)

- WordcountDriverを運行

![image-20240815163842771](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815163842771.png)

　全て順調にしたら上図のように成功になります。下図の二つはwinutils.exe、hadoop.dll欠いてのエラーメッセージで、参考とできます。

![image-20240815144525696](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815144525696.png)

![image-20240815160024111](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240815160024111.png)

- `D:\output`結果の検査

![image-20240816095655450](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240816095655450.png)

- part-r-00000結果ファイルを開けて下図のように表れる

![image-20240816095901152](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240816095901152.png)

**プログラム運行（Hadoop分散式サービス）**

- 分散式サービスの準備

　第２章にHadoopのテストを真似てサービスに運行するつもりです。その前に`/wcoutput`の全てを消除しておく必要です。

![image-20240816152923496](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240816152923496.png)

```
#そのディレクトリに権限を与える
hdfs dfs -chmod -R 777 /
```

![image-20240816153712200](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240816153712200.png)

　エラーメッセージまだあるけれど、実際に消除できます。

![image-20240816153516317](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240816153516317.png)

- プログラムをパッケージ化にする

　ネット上で色々方法を紹介してあります。ここでIdeaツールを使ってパッケージ化にします。図の右側サイドバーMavenを開けてpackage選択肢をダブルクリックするとパッケージ化を進行し始めます

![image-20240820063405560](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820063405560.png)

![image-20240820063513855](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820063513855.png)

　プログラムの下にtargetディレクトリにパッケージがあります。

![image-20240820070613455](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820070613455.png)

　もし全てのコードをパッケージ化にしたくないなら、Mavenパッケージ化のプラグイン配置に下の内容を添加します。

```
<plugins>
    <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <configuration>
            <includes>
                <include>com/javapractice/hadoop/**</include>
            </includes>
        </configuration>
    </plugin>
</plugins>
```

![image-20240820072208378](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820072208378.png)

　`rz`コマンドを入力してパッケージをアップロードします。

![image-20240820071555383](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820071555383.png)

![image-20240820071622495](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820071622495.png)

- パッケージを運行

```
hadoop jar JavaPractice-1.0.0-SNAPSHOT.jar com.javapractice.hadoop.WordCountDriver /wcinput /wcoutput
```

![image-20240820074348930](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074348930.png)

![image-20240820074437964](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074437964.png)

　画面に出力内容が出ました。`http://192.168.31.135:19888/jobhistory`に入って対応のタスクログがあります。

![image-20240820074706094](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074706094.png)

```
#出力内容を検査
hdfs dfs -cat /wcoutput/part-r-00000
```

![image-20240820074843969](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240820074843969.png)

　ここまで手動的にWordCount機能を実現するのが終わりです。

## 第４節　Hadoopの直列化

　直列化は、ネットワーク通信でデータを送信したり、オブジェクトをファイルに永続化したりする際に、送信効率、格納空間など考えてオブジェクトをバイナリ構造に変換する過程です。

　先紹介したWordcount案例が、MapperクラスやReducerクラスには自分の独自の直列化型があり、例えばIntWritable、longWritable型がありますが、それらの型は一般的なJavaの基本型ではありません。

 　なぜHadoopはJava標準のSerializableを使わずに独自の直列化の形式を選んだのか？Hadoopでは、クラスター内の複数の節点間のプロセス通信がRPC（Remote Procedure Call）を通じて実現されます。RPCはメッセージを直列化にしてバイナリストリームに変換し、それをリモート節点に送信します。リモート節点は受信したバイナリデータを逆直列化して元のメッセージに戻します。そのため、RPCは以下のような特徴を追求します：

- **コンパクト**: データがよりコンパクトになり、ネットワーク帯域を効率的に利用できる。
- **高速性**: 直列化と逆直列化の性能負荷が低い。

　Hadoopは、独自のシリアライズ形式であるWritableを使用しており、これはJavaのシリアライズ形式であるSerializableよりもコンパクトで高速です。Serializableを使ってオブジェクトを直列化にすると、検証情報、ヘッダー、継承体系などの多くの追加情報が付加されます。

　Hadoopにの基本データ類型とJava伝統基本類型の対照表。

| Java類型 | Hadoop Writable類型 |
| -------- | ------------------- |
| boolean  | BooleanWritable     |
| byte     | ByteWritable        |
| int      | IntWritable         |
| float    | FloatWritable       |
| long     | LongWritable        |
| double   | DoubleWritable      |
| String   | Text                |
| map      | MapWritable         |
| array    | ArrayWritable       |

## 第５節　直列化インターフェース

　Haddoop内部がHaddoop自分のデータ類型によって通信します。もしBean対象をHadoopに渡してBean対象が必ずWritableインターフェースを実装します。

```
public class Student implements Writable {

    private Long id;
    private String name;
    private Integer age;
	//直列化
    @Override
    public void write(DataOutput dataOutput) throws IOException {
        dataOutput.writeLong(id);
        dataOutput.writeUTF(name);
        dataOutput.writeInt(age);
    }
	//逆直列化
    @Override
    public void readFields(DataInput dataInput) throws IOException {
        this.id = dataInput.readLong();
        this.name = dataInput.readUTF();
        this.age = dataInput.readInt();
    }
}
```

　Bean実体がWritableインターフェースを実装して二つ直列化write()と逆直列化readFields()メソッドが上書きする必要です。直列化と逆直列化の類型の扱いが必ず一致にします。後は、String類型の直列化がwriteUTF()メソッドに対応し、writeString()みたいメソッドがありません。
