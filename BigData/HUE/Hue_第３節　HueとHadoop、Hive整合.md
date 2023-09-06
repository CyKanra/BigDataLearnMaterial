# データ交互工具ーHUE

## 第３節　HueとHadoop、Hive整合

　　/opt/bigdata/servers/hue/desktop/conf目録に入って、pseudo-distributed.iniファイル内容を改修する。

**HDFS、YARNと集成**

```
# [hadoop] -- [[hdfs_clusters]] -- [[[default]]]
fs_defaultfs=hdfs://centos1:9000
webhdfs_url=http://centos1:50070/webhdfs/v1
hadoop_conf_dir=/opt/bigdata/servers/hadoop-2.9.2/etc/hadoop
```

![image-20230814145522856](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230814145522856.png)

```
# [hadoop] -- [[yarn_clusters]] -- [[[default]]]
resourcemanager_host=centos2
resourcemanager_port=8032
submit_to=True
resourcemanager_api_url=http://centos2:8088
proxy_api_url=http://centos2:8088
history_server_api_url=http://centos2:19888
```

![image-20230814150837861](C:\Users\Izaya\AppData\Roaming\Typora\typora-user-images\image-20230814150837861.png)

**Hiveと集成**

```
# [beeswax]
hive_server_host=centos2
hive_server_port=10000
hive_conf_dir=/opt/lagou/servers/hive-2.3.7/conf
```

