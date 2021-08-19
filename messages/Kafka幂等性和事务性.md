# Kafka幂等性和事务性
> 生产者重复生产消息。生产者进行retry会产生重试时，会重复产生消息。有了幂等性之后，在进行retry重试时，只会生成一个消息。  


## PID 和 Sequence Number
* PID: 每个新的Producer在初始化的时候会被分配一个唯一的PID，这个PID对用户是不可见的。
* Sequence Numbler:（对于每个PID，该Producer发送数据的每个<Topic, Partition>都对应一个从0开始单调递增的Sequence Number。


Broker端在缓存中保存了这seq number，对于接收的每条消息，如果其序号比Broker缓存中序号大于1则接受它，否则将其丢弃。这样就可以实现了消息重复提交了。但是，只能保证单个Producer对于同一个<Topic, Partition>的Exactly Once语义。不能保证同一个Producer一个topic不同的partion幂等。

![](Kafka%E5%B9%82%E7%AD%89%E6%80%A7%E5%92%8C%E4%BA%8B%E5%8A%A1%E6%80%A7/1-1.png)

实现幂等之后

![](Kafka%E5%B9%82%E7%AD%89%E6%80%A7%E5%92%8C%E4%BA%8B%E5%8A%A1%E6%80%A7/2.png)

## 幂等性和事务性的关系
> 事务属性实现前提是幂等性，即在配置事务属性transaction id时，必须还得配置幂等性；但是幂等性是可以独立使用的，不需要依赖事务属性。  
* 幂等性引入了Porducer ID
* 事务属性引入了Transaction Id属性

* enable.idempotence = true，transactional.id不设置：只支持幂等性。
* enable.idempotence = true，transactional.id设置：支持事务属性和幂等性
* enable.idempotence = false，transactional.id不设置：没有事务属性和幂等性的kafka
* enable.idempotence = false，transactional.id设置：无法获取到PID，此时会报错


### 如何保证消息不丢
1. 发送消息时,可以建立一个日志表,和业务处理在一个事务,定时扫描表发送没有被处理的消息
2. 消费端，消费消息之后，修改消息表的中消息状态为已处理成功。

### Kafka消费者角度，如何保证消息不被重复消费。
1. 通过seek操作
2. 通过kafka事务操作

### Kafka生产者角度，如何保证消息不重复生产
kakfka幂等性
