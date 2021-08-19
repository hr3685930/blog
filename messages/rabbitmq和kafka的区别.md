# rabbitmq和kafka的区别

## 吞吐量
Kafka吞吐量更高：
1.  Zero Copy机制，内核copy数据直接copy到网络设备，不必经过内核到用户再到内核的copy，减小了copy次数和上下文切换次数，大大提高了效率。
2. 磁盘顺序读写，减少了寻道等待的时间。
3. 批量处理机制，服务端批量存储，客户端主动批量pull数据，消息处理效率高。
4. 存储具有O(1)的复杂度，读物因为分区和segment，是O(log(n))的复杂度。
5. 分区机制，有助于提高吞吐量。

## 可靠性

Rabbitmq可靠性更好：
1. 确认机制（生产者和exchange，消费者和队列）
2. 支持事务，但会造成阻塞
3. 委托（添加回调来处理发送失败的消息）和备份交换器（将发送失败的消息存下来后面再处理）机制

## 高可用
1. rabbitmq采用mirror queue，即主从模式，数据是异步同步的，当消息过来，主从全部写完后，回ack，这样保障了数据的一致性。
2. 每个分区都可以有一个或多个副本，这些副本保存在不同的broker上，broker信息存储在zookeeper上，当broker不可用会重新选举leader。
> kafka支持同步负责消息和异步同步消息（有丢消息的可能），生产者从zk获取leader信息，发消息给leader，follower从leader pull数据然后回ack给leader。  

## 负载均衡
1. kafka通过zk和分区机制实现：zk记录broker信息，生产者可以获取到并通过策略完成负载均衡；通过分区，投递消息到不同分区，消费者通过服务组完成均衡消费。
2. 需要外部支持。

## 模型
1. rabbitmq：
	- producer，broker遵循AMQP（exchange，bind，queue），consumer
	- broker为中心，exchange分topic，direct，fanout和header，路由模式适合多种场景
	- consumer消费位置由broker通过确认机制保存
2. kafka：
	- producer，broker，consumer，未遵循AMQP
	- consumer为中心，获取消息模式由consumer自己决定
	- offset保存在消费者这边，broker无状态
	- 消息是名义上的永久存储，每个parttition按segment保存自己的消息为文件（可配置清理周期）
	- consumer可以通过重置offset消费历史消息
	- 需要绑定zk


### rabbitmq顺序性:
![](rabbitmq%E5%92%8Ckafka%E7%9A%84%E5%8C%BA%E5%88%AB/5b0c06647a1a58a50ca7241d2ec056b8.jpeg)

### 重复消费问题:
1. 保证消费者幂等性，重复消费也没有关系。
2. 消息定向消费，并且记录，过滤重复消息。

### Kafka消息保证生产的消息不丢失和不重复消费的问题:
[Kafka如何保证消息不丢失不重复 - 提拉没有米苏 - 博客园](https://www.cnblogs.com/cherish010/p/9764810.html)

> 从吞吐量上看，在不要求消息顺序情况下，Kafka完胜；在要求消息先后顺序的场景，性能应该稍逊RabbitMQ（此时Kafka的分片数只能为1）。  
> 从稳定性来看，RabbitMQ胜出，但是Kafka也并不逊色多少。  
