# 分散型協調サービスフレームワーク -- Zookeeper-4

## 第４章　Zookeeperの基本使用

### 第１節　Zookeeperの命令操作

　　先ずは、bin目録下にzkClientに介してZookeeperの操作卓に入り、連接したの後でシステムがZookeeperに関する環境情報を表すことがある。

```
# 当のサーバを連接
./zkcli.sh

# 指定されるサーバを連接（port:2181）
./zkCli.sh -server ip:port
```

![image-20231017165730630](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231017165730630.png)

　　「help」を入力してZookeeperの可用命令が画面に表示できる。

![image-20231017165835192](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231017165835192.png)

**節点の作成**

　　create命令を使ってZookeeperの一つの節点を作成する。

```
# [-s]：順序節点、[-e]：臨時節点、[-e]なし：持久節点
create [-s][-e] path data
```

```
# 順序節点作成
create -s /zk-test 123
```

![image-20231018145352669](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231018145352669.png)

```
# 臨時節点
create -e /zk-temp 123
```

![image-20231018145903016](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231018145903016.png)

```
# 持久節点
create /zk-test 123
```

![image-20231018150702412](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231018150702412.png)

```
# 根目録の検査
ls /
```

![image-20231018150819195](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231018150819195.png)

　　拡張子の不同によって各節点の差別が理解できるはずだと思う。若し順序臨時節点を作成したい、「create -s -e /zk-temp 123」ような二つの変数を付いていい。

　　会話を退出して再Zookeeperサーバに入って根目録の検査にする。臨時節点が消除された。

![image-20231018151956450](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20231018151956450.png)

**節点の読込**

　　ls命令で指定される節点の下に全ての節点を示し、ただ、当の節点の一級の子節点のみが表れることです。

```
ls  path
```

　　get命令で指定される節点の属性やデータ内容などの情報を示す。

```
get path 
```

![image-20231018173456976](D:\OneDrive\图片\Typora\image-20231018173456976.png)

**節点の更新**

　　set命令で指定される節点のデータ内容を更新する。

```
set path data
```

![image-20231018175011750](D:\OneDrive\图片\Typora\image-20231018175011750.png)

　　二つの図を比較して変更時刻（mtime）を更新された、データ長さが３から４になった。データバージョンにとって毎回変更が発生するに伴い、バージョン番号が累増にする。つまり、内部データのファイルが節点のバージョンに一々対応する関係が存在です。

**節点の消除**

　　delete命令で指定される節点を消除する。ただ、当の節点は子節点が存在するなら、節点の消除ができない。必ず先子節点を消除した、親節点の消除ができる。

```
delete path
```

![image-20231018180823621](D:\OneDrive\图片\Typora\image-20231018180823621.png)
