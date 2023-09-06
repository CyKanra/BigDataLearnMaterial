# データ交互工具ーHUE

## 第２節　Hueインストール

　　HueのインストールがHiveのような簡単ではない、公式にはコンパイル済みのソフトウェアパッケージは提供されない、Githubからパッケージをダウンロードし、相関の依頼をインストールし、コンパイルしてインストールする必要がある。以下にHueのダウンロード、コンパイル、インストールの手順を詳しく説明する。

備考：Hueインストールした前にMySQLがインストールしない節点を選んでいい。幾つか依頼パッケージ間にバージョンが不一致してある。ここでCentos03節点を選んでインストールするつもりです。

| ソフト | Centos1 | Centos2 | Centos3 | Centos4 |
| ------ | ------- | ------- | ------- | ------- |
| Hadoop | √       | √       | √       | √       |
| Hive   |         | √       |         |         |
| MySQL  |         | √       |         |         |
| Hue    |         |         | √       |         |

**ソフトウェアの準備**

Hue-release-4.3.0リンク：[Release release-4.3.0 · cloudera/hue · GitHub](https://github.com/cloudera/hue/releases/tag/release-4.3.0)

![image-20230808170132635](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230808170132635.png)

zip拡張子のパッケージをダウンロードしてください。

```
#SecureCRTツールへHueパッケージを導入
rz
```

![image-20230810153606965](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230810153606965.png)

![image-20230810153712815](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230810153712815.png)

```
#パッケージを解凍前に、若しunzip解凍命令が実施できない、yum方式でインストールしてください
yum install unzip
```

![image-20230810154154135](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230810154154135.png)

```
unzip hue-release-4.3.0.zip
```

![image-20230810154337653](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230810154337653.png)

**相関の依頼をインストール**

　　Hueのある結成がpythonから引用するため、事前にpython環境を準備するは必要です。以下はバージョン情報から了解して、Hueの4.3.0バージョンはPython 2.7.x.以上が要する。

![image-20230810160215133](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230810160215133.png)

```
#検査バージョン情報
python --version

#他の依頼をインストール
yum install ant asciidoc cyrus-sasl-devel cyrus-sasl-gssapicyrus-sasl-plain gcc gcc-c++ krb5-devel libffi-devel libxml2-devel libxslt-devel make mysql mysql-devel openldap-devel　python-devel sqlite-devel gmp-devel

#その同期ツールもインストールしており、公式が説明されないけれど、Hueインストール時に錯誤が発生してある。
yum install -y rsync
```

備考：依頼の準備がちょっと面倒、各人のサービスが不同だために以上の内容を参考として、錯誤情報によって欠ける依頼を直接にインストールしていい。この文章はCentOS/RHEL 7.Xに適するだけ、他の情況はこのリンクを参考してください。

依頼情報：[Dependencies :: Hue SQL Assistant Documentation (gethue.com)](https://docs.gethue.com/administrator/installation/dependencies/)

**Mavenインストール**

同様にソフトパッケージをダウンロードし、仮想機に導入し、でも今度、環境変数を添加する必要です。ここで apache-maven-3.6.3-bin.tar.gzバージョンを選ぶ。

 maven-3.6.3リンク：[Index of /dist/maven/maven-3/3.6.3/binaries (apache.org)](https://archive.apache.org/dist/maven/maven-3/3.6.3/binaries/)

```
#パッケージを解凍
tar -vxzf apache-maven-3.6.3-bin.tar.gz

#環境変数を添加
vim /etc/profile

#最後に环境变量を添加
export MAVEN_HOME=/opt/bigdata/servers/apache-maven-3.6.3
export PATH=$PATH:$MAVEN_HOME/bin
```

![image-20230811121015844](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811121015844.png)

```
#配置ファイルを発効にして
source /etc/profile

#検証
mvn --version
```

![image-20230811121315043](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811121315043.png)

**コンパイルインストール**

```
#Hue目録に入って
cd /opt/bigdata/servers/hue-release-4.3.0

#PREFIX＋インストール経路
PREFIX=/opt/bigdata/servers/ make install
```

若しコンパイルが失敗なら、返却情報によって欠け依頼のパッケージを探す。

![image-20230811140528498](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811140528498.png)

python-develが欠けて知ってた、下記のようにインストールし直して解決できるんです。

![image-20230811140533536](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811140533536.png)

　　明るい錯誤提示がないなら、コンパイルが成功だと言われる。

**Hadoop配置ファイルの改修**

hdfs-site.xmlファイルに以下の内容を追加

```
<!-- HUE -->
<property>
	<name>dfs.webhdfs.enabled</name>
	<value>true</value>
</property>
<property>
	<name>dfs.permissions.enabled</name>
	<value>false</value>
</property>
```

![image-20230811162046459](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811162046459.png)

core-site.xmlファイルに以下の内容を追加

```
<!-- HUE -->
<property>
	<name>hadoop.proxyuser.hue.hosts</name>
	<value>*</value>
</property>
<property>
	<name>hadoop.proxyuser.hue.groups</name>
	<value>*</value>
</property>
<property>
	<name>hadoop.proxyuser.hdfs.hosts</name>
	<value>*</value>
</property>
<property>
	<name>hadoop.proxyuser.hdfs.groups</name>
	<value>*</value>
</property>
```

![image-20230811162403071](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811162403071.png)

httpfs-site.xmlファイルに以下の内容を追加

```
<!-- HUE -->
<property>
	<name>httpfs.proxyuser.hue.hosts</name>
	<value>*</value>
</property>
<property>
	<name>httpfs.proxyuser.hue.groups</name>
	<value>*</value>
</property>
```

![image-20230811170057659](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811170057659.png)

準備済みファイルを各節点に分配する

```
scp -r hadoop/ root@centos2:$PWD
```

![image-20230811172516781](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811172516781.png)

　　再起動HDFS全てのサービス

```
#centos1
start-dfs.sh / stop-dfs.sh
mr-jobhistory-daemon.sh start / stop historyserver

#centos2
start-yarn.sh / stop-yarn.sh
```

**Hue配置**

```
#配置目録に入る
cd hue/desktop/conf

#复制一份HUE的配置文件，并修改复制的配置文件
cp pseudo-distributed.ini.tmpl pseudo-distributed.ini
vim pseudo-distributed.ini
```

![image-20230811190623420](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811190623420.png)

```
#以下の内容ように改修、図の右下隅の行目を注意して早く探せる
#[desktop]
http_host=centos3
http_port=8000
is_hue_4=true
time_zone=Asia/Tokyo
dev=true
server_user=hue
server_group=hue
default_user=hue
```

![image-20230811204502343](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811204502343.png)

```
#211行目ぐらい、幾つか錯誤提示を避ける
app_blacklist=search
```

![image-20230811204507869](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811204507869.png)

```
#[[database]]、Hue黙認にSQLiteデータベース使って関連データを記録し、ここMySQLに差し替える
engine=mysql
host=centos2
port=3306
user=hive
password=12345678
name=hue
```

![image-20230811204518450](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811204518450.png)

**MysqlデータベースHue作成**

```
#centos2節点にMysql入れ、ここHiveユーザを使って、pseudo-distributed.iniと一致です
mysql -uhive -p12345678

#データベース作成
create database hue;
```

![image-20230811205805347](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811205805347.png)

**Hueデータベース初期化**

```
#目録に入れ
cd /opt/bigdata/servers/hue/build/env/bin

#初期化
./hue syncdb
./hue migrate
```

![image-20230811211119457](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811211119457.png)

![image-20230811211125401](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811211125401.png)

　　Hueデータベースを検査し、さっき作成されたデータベースがテーブルも生成してある。

![image-20230811210901921](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230811210901921.png)

**Hue起動**

```
#hueユーザとユーザグループを添加
groupadd hue
useradd -g hue hue
```

![image-20230813154621286](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230813154621286.png)

　　その扱いなければ、上の錯誤がある。

```
#実行
cd /opt/bigdata/servers/hue/build/env/bin
./supervisor
```

![image-20230813155321032](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230813155321032.png)

　　以上のメッセージが見えるなら、Hueサービスも成功して起動された。

　　ブラウザにサービスのIPアドレスとボート号を輸入：192.168.31.131:8000、以下の内容が表れる。最初に訪問する時にユーザとパスワードが必要です（hue/123456）。

![image-20230813181950556](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230813181950556.png)

![image-20230813182850188](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230813182850188.png)

　　順調に登録するなら、このような画面が見えて、右上のエラーメッセージは今暫く没却していい。
