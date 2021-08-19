# Kafka
> KafKa是一款由Apache软件基金会开源,由Scala编写的一个分布式发布订阅消息系统,Kafka最初是由LinkedIn开发，并于2011年初开源(知道这个就够了),KafKa它最初的目的是为了解决,统一,高效低延时,高通量(同时能传输的数据量)并且高可用一个消息平台.  

### 几个角色
* producer：
　　消息生产者，发布消息到 kafka 集群的终端或服务。
* broker：
　　kafka 集群中包含的服务器。
* topic：
　　每条发布到 kafka 集群的消息属于的类别，即 kafka 是面向 topic 的。
* partition：
　　partition 是物理上的概念，每个 topic 包含一个或多个 partition。kafka 分配的单位是 partition。
* consumer：
　　从 kafka 集群中消费消息的终端或服务。
* Consumer group：
　　high-level consumer API 中，每个 consumer 都属于一个 consumer group，每条消息只能被 consumer group 中的一个 Consumer 消费，但可以被多个 consumer group 消费。
* replica：
　　partition 的副本，保障 partition 的高可用。
* leader：
　　replica 中的一个角色， producer 和 consumer 只跟 leader 交互。
* follower：
　　replica 中的一个角色，从 leader 中复制数据。
* controller：
　　kafka 集群中的其中一个服务器，用来进行 leader election 以及 各种 failover。
* zookeeper：
　　kafka 通过 zookeeper 来存储集群的 meta 信息。


### 特征
* kafka接收到的消息最终会以文件的形式存在本地,保证了只要消息接受成功理论上就不会丢失
* KafKa通过append来实现消息的追加,保证消息都是有序的有先来后到的顺序
* KafKa集群有良好的容灾机制,比如有N台服务器,可以承受N-1台服务器故障是保证提交的消息不会丢失
* KafKa会更具Topic以及partition来进行消息在本地的物理划分
* KafKa依赖zookeeper实现了offset,你不用关心到你获取了那些消息KafKa会知道并且在你下次获取时接着给你
* 你可以获取任意一个offset的记录
* 消息可以在KafKa内保存很长的时间也可以很短,KafKa基于文件系统能存储消息的容量取决于硬盘空间
* KafKa的性能不会受到消息的数量影响

### Kafka 0拷贝
传统方式实现：
先读取、再发送，实际经过1~4四次copy。
1、第一次：将磁盘文件，读取到操作系统内核缓冲区；
2、第二次：将内核缓冲区的数据，copy到application应用程序的buffer；
3、第三步：将application应用程序buffer中的数据，copy到socket网络发送缓冲区(属于操作系统内核的缓冲区)；
4、第四次：将socket buffer的数据，copy到网卡，由网卡进行网络传输。

零拷贝 sendfile(in,out)
数据直接在内核完成输入和输出，不需要拷贝到用户空间再写出去。
Kafka数据写入磁盘前，数据先写到进程的内存空间。

### 安装
我这里使用编排的方式

```
version: ‘2’
services:
  zookeeper:
    image: wurstmeister/zookeeper
    ports:
      - “2181:2181”

  kafka:
    image: wurstmeister/kafka
    ports:
      - “9092:9092”
    environment:
      KAFKA_ADVERTISED_HOST_NAME: 10.10.172.54  #本机ip
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_CREATE_TOPICS: test
```