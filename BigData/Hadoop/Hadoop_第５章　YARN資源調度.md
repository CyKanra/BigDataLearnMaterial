# 分散大規模データ処理システム -- Hadoop-10

# 第５章　YARN資源調度-1

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

![image-20250828172350879](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20250828172350879.png)

**Taskの提出**

1. WordCount計算案例を紹介する時、`JavaPractice-1.0.0-SNAPSHOT.jar`パッケージを運行する前にノードにアップロードすることがある。
2. パッケージを運行して、先ずResourceManagerへ計算任務Applicationの申請を提出する。
3. 