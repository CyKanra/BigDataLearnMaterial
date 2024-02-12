# 分散大規模データ処理システム -- Hadoop-2

## 第２章　Hadoopの分散システム組み立て

　　Hadoopの分散システム組立の前にクラスタの環境設定が必要ため、ここで大体の改修項目を紹介し、詳しい設定手続きが「」文章に移って了解できる。

- クラスタは全て4つのサービスを備えてあり、hostファイルにIPアドレスのマッピングが図のように示す。

![image-20240212154807536](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212154807536.png)

- firewall／selinux等の動きが停止してた。

```
systemctl status firewalld
```

![image-20240212160400972](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212160400972.png)

```
vim /etc/selinux/config
```

![image-20240212160439078](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212160439078.png)

- JDKやCentOSバージョン

![image-20240212161442705](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212161442705.png)

- 4つのサービスの間にパスワード不要で互いに登録して通信することができる。

- 中の一つマシンを選んで時間サービスとして、他のマシンはこのマシンと定期的に時間を同期する。例えば、10分ごとに一度時間を同期するように設定する。

```
vim /etc/ntp.conf
```

![image-20240212163854170](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212163854170.png)

- BIOSやシステム時刻を同期にする一致性性を保つ

```
vim /etc/sysconfig/ntpd
```

![image-20240212164252295](D:\OneDrive\picture\Typora\BigData\Hadoop\image-20240212164252295.png)

