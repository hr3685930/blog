# flume 日志采集记录
![](flume%20%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E8%AE%B0%E5%BD%95/Pasted%20Graphic.png)
flumex-1.0-SNAPSHOT.jar

将打包文件和所需依赖包fastjson.jar上传到slave-3 /opt/cloudera/parcels/CDH/lib/flume-ng/lib/目录下

并改名为LogAnalysis.jar
![](flume%20%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E8%AE%B0%E5%BD%95/Pasted%20Graphic%204.png)


更改/opt/cloudera/parcels/CDH/etc/flume-ng/conf.dist
![](flume%20%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E8%AE%B0%E5%BD%95/411551262698_.pic_hd.jpg)

supervisor启动/etc/supervisor.conf

测试topic uat-apilog

生产
sh /opt/cloudera/parcels/KAFKA/bin/kafka-console-producer —broker-list slave-1:9092,slave-2:9092,slave-3:9092 —topic uat-apilog


测试数据
{“@timestamp”:”2018-12-17T09:48:30.376Z”,”beat”:{“hostname”:”iZbp108kw6wutrors3nlptZ”,”name”:”iZbp108kw6wutrors3nlptZ”,”version”:”5.1.2”},”input_type”:”log”,”message”:”223.104.23.159 - - [01/Sep/2018:17:48:29 +0800] {\”filePath\”:\”\\/data\\/walle\\/api-datacollect\\/20180830-182219\\/app\\/datacollect\\/2.0\”,\”requestDate\”:\”2018-12-17 17:48:29\”,\”apiName\”:\”datacollect2.net_info\”,\”objectName\”:\”datacollect2\”,\”runTime\”:0.011,\”request\”:{\”params\”:\”{\\\”0\\\”:{\\\”frequency\\\”:\\\”2.4GHz\\\”,\\\”netType\\\”:\\\”WIFI\\\”,\\\”operatorName\\\”:\\\”\\\”,\\\”operatorNo\\\”:\\\”\\\”,\\\”phoneNumber\\\”:\\\”\\\”,\\\”signal\\\”:-25,\\\”time\\\”:1535673894},\\\”1\\\”:{\\\”frequency\\\”:\\\”2.4GHz\\\”,\\\”netType\\\”:\\\”WIFI\\\”,\\\”operatorName\\\”:\\\”\\\”,\\\”operatorNo\\\”:\\\”\\\”,\\\”phoneNumber\\\”:\\\”\\\”,\\\”signal\\\”:-32,\\\”time\\\”:1535685516},\\\”2\\\”:{\\\”frequency\\\”:\\\”2.4GHz\\\”,\\\”netType\\\”:\\\”WIFI\\\”,\\\”operatorName\\\”:\\\”\\\”,\\\”operatorNo\\\”:\\\”\\\”,\\\”phoneNumber\\\”:\\\”\\\”,\\\”signal\\\”:-40,\\\”time\\\”:1535702991},\\\”3\\\”:{\\\”frequency\\\”:\\\”2.4GHz\\\”,\\\”netType\\\”:\\\”WIFI\\\”,\\\”operatorName\\\”:\\\”\\\”,\\\”operatorNo\\\”:\\\”\\\”,\\\”phoneNumber\\\”:\\\”\\\”,\\\”signal\\\”:-28,\\\”time\\\”:1535705220},\\\”4\\\”:{\\\”frequency\\\”:\\\”2.4GHz\\\”,\\\”netType\\\”:\\\”WIFI\\\”,\\\”operatorName\\\”:\\\”\\\”,\\\”operatorNo\\\”:\\\”\\\”,\\\”phoneNumber\\\”:\\\”\\\”,\\\”signal\\\”:-62,\\\”time\\\”:1535710671},\\\”5\\\”:{\\\”frequency\\\”:\\\”2.4GHz\\\”,\\\”netType\\\”:\\\”WIFI\\\”,\\\”operatorName\\\”:\\\”\\\”,\\\”operatorNo\\\”:\\\”\\\”,\\\”phoneNumber\\\”:\\\”\\\”,\\\”signal\\\”:-33,\\\”time\\\”:1535713356},\\\”6\\\”:{\\\”frequency\\\”:\\\”2.4GHz\\\”,\\\”netType\\\”:\\\”WIFI\\\”,\\\”operatorName\\\”:\\\”\\\”,\\\”operatorNo\\\”:\\\”\\\”,\\\”phoneNumber\\\”:\\\”\\\”,\\\”signal\\\”:-36,\\\”time\\\”:1535714812},\\\”msn\\\”:\\\”V101174800418\\\”,\\\”machineModel\\\”:\\\”v1\\\”,\\\”originalMachineModel\\\”:\\\”v1\\\”}\”},\”response\”:\”datacollect_response\”}”,”offset”:8236712343,”source”:”/data/belklog/api/app/onldatacollect2_2018-12-17.log”,”type”:”json-app-api”}

Tmp下的文件内容是否存在
![](flume%20%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E8%AE%B0%E5%BD%95/441551263180_.pic_hd.jpg)

hdfs dfs -ls /user/hive/warehouse/dates=2019-02-27/uatlog/(查询)

hdfs dfs -cat /user/hive/warehouse/dates=2019-02-27/uatlog/apilog20190227.1551261389867.data.tmp(查询文件内容)

如果排查不出flume无法采集数据的原因，尝试在cdh重启flume

Hdfs数据导入到hive(azkaban实现)
/usr/local/shell/LoadApilogHdfs

![](flume%20%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E8%AE%B0%E5%BD%95/Pasted%20Graphic%202.png)

Azkaban
![](flume%20%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E8%AE%B0%E5%BD%95/Pasted%20Graphic%201.png)
![](flume%20%E6%97%A5%E5%BF%97%E9%87%87%E9%9B%86%E8%AE%B0%E5%BD%95/Pasted%20Graphic%202%202.png)