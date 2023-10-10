# データ採集工具 -- Flume-2

## 第２節　Flumeインストール

Flume公式サイト：[Welcome to Apache Flume — Apache Flume](https://flume.apache.org/)

ドキュメント：[Flume 1.11.0 User Guide — Apache Flume](https://flume.apache.org/releases/content/1.11.0/FlumeUserGuide.html)

ダウンロード：[Index of /dist/flume (apache.org)](http://archive.apache.org/dist/flume/)

　　ここで1.9.0版を選び、apache-flume-1.9.0-bin.tar.gzパッケージをダウンロードし上げて本地サーバにアップロードする。

```
#パッケージを解凍
tar zxfv apache-flume-1.9.0-bin.tar.gz
#ファイル名を改修
mv apache-flume-1.9.0-bin flume-1.9.0
```

![image-20230922160218706](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230922160218706.png)

**環境変数改修**

```
vim /etc/profile

#以下の内容を添加
export FLUME_HOME=/opt/bigdata/servers/flume-1.9.0
export PATH=$PATH:$FLUME_HOME/bin

#配置ファイルを効く
source /etc/profile
```

![image-20230922162001633](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230922162001633.png)

**Flume配置改修**

```
#配置目録に入って
cd $FLUME_HOME/conf
mv flume-env.sh.template flume-env.sh
vim flume-env.sh

#以下の内容を改修
export JAVA_HOME=/opt/bigdata/servers/jdk1.8.0_231
```

![image-20230922164617958](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230922164617958.png)