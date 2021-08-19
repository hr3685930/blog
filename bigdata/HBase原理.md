# HBase原理

> HDFS提供了分布式底层文件系统，解决了基础设施层面文件存储的问题，但是对于文件内容的查询等操作需要依赖于MapReduce计算框架，且效率低下。HBase支持在HDFS之上存储大文件，并提供比HDFS更高效检索数据的框架，下面是学习的一写资料整理。  

## 总体架构
下图是网上找到的，虽然版本比较旧，涉及的一些内容在新版本上已有更新，但是基本表述清楚了HBase的整体架构和核心原理。
![](HBase%E5%8E%9F%E7%90%86/hbase.jpeg)数据流程
要了解数据流程，除了知道前面大图中涉及的HBase基本组件之外，还需要了解以下知识点：
* HRegion由多个Store组成（每个Store对应一个clumne family），Store内部包含memstore和多个HFile（多个HFile的原因是Minor Compact会生成很多的storeFile，其实storeFile就是HFile）；
* region server上的所有region都共享同一个HLog。
## 读取流程
1. client访问zookeeper获取hbase:meta表的地址；
2. 通过查询hbase:meta表，client获取到目标对象region所在的region server；（在老版本中是三层结构，新版本为两层）
3. client直接去region server访问数据;
4. client会将region server与region的映射缓存到本地，只有在发生读取错误的情况下才会重新走一遍前面的查询流程。
## 写入流程
1. client请求到达region server，region server在写完HLog以后，数据写入的下一个目标就是region的memstore；
2. 写入到memstore后，该次写入请求就可以被返回，HBase即认为该次数据写入成功（支持三种刷盘方式）；
	1. 通过全局内存控制，触发memstore刷盘操作
	2. 手动触发memstore刷盘操作
	3. memstore上限触发数据刷盘
3. 每次memstore的刷盘都会相应生成一个存储文件storeFile（即HFile在HBase层的轻量级封装）；
4. region server通过compact把大量小的HFile进行文件合并，生成大的HFile文件（支持两种压缩类型）；
	1. Minor Compact
	2. Major Compact（对整个region下相同CF的所有HFile进行compact，清理过期或者被删除的数据）

## Region分裂
HBase同样提供了region的 split方案来解决大的HFile造成数据查询时间过长问题。
一个较大的region(指其内部的所有sotre总和达到阀值)通过split操作，会生成两个小的region，称之为Daughter。
* 流程：
	1. region先更改ZK中该region的状态为SPLITING；
	2. Master检测到region状态改变；
	3. region会在存储目录下新建.split文件夹用于保存split后的daughter region信息；
	4. Parent region关闭数据写入并触发flush操作，保证所有写入Parent region的数据都能持久化；
	5. 在.split文件夹下新建两个region，称之为daughter A、daughter B；
	6. Daughter A、Daughter B拷贝到HBase根目录下，形成两个新的region；
	7. Parent region通知修改.META.表后下线，不再提供服务；
	8. Daughter A、Daughter B上线，开始向外提供服务；
	9. 如果开启了balance_switch服务，split后的region将会被重新分布。
