# Kafka API
https://github.com/Shopify/sarama

## 生产者
```
//设置配置
config := sarama.NewConfig()
//等待服务器所有副本都保存成功后的响应
config.Producer.RequiredAcks = sarama.WaitForAll
//随机的分区类型
config.Producer.Partitioner = sarama.NewRandomPartitioner
//是否等待成功和失败后的响应,只有上面的RequireAcks设置不是NoReponse这里才有用.
config.Producer.Return.Successes = true
config.Producer.Return.Errors = true
//设置使用的kafka版本,如果低于V0_10_0_0版本,消息中的timestrap没有作用.需要消费和生产同时配置
config.Version = sarama.V0_11_0_0

 //使用配置,新建一个异步生产者
producer, e := sarama.NewAsyncProducer([]string{"IP:9092","IP:9092","IP:9092"}, config)
    if e != nil {
        panic(e)
    }
    defer producer.AsyncClose()

    //发送的消息,主题,key
    msg := &sarama.ProducerMessage{
        Topic: "logstash_test",
        Key:   sarama.StringEncoder("test"),
    }

    var value string
    for {
        value = "this is a message"
        //设置发送的真正内容
        fmt.Scanln(&value)
        //将字符串转化为字节数组
        msg.Value = sarama.ByteEncoder(value)
        fmt.Println(value)

        //使用通道发送
        producer.Input() <- msg

        //循环判断哪个通道发送过来数据.
        select {
        case suc := <-producer.Successes():
            fmt.Println("offset: ", suc.Offset, "timestamp: ", suc.Timestamp.String(), "partitions: ", suc.Partition)
        case fail := <-producer.Errors():
            fmt.Println("err: ", fail.Err)
        }
    }
```

- 首先新建一个config,用于配置生产者相关的配置项
- 通过config和一个包含一个或多个kafka服务器的字符串数组,新建一个producer
- 定义一个 生产信息 ,包括发送的主题,哪个分区,重试次数等等信息和消息内容.通过producer的输入通道,接受msg
- 如果配置中配置了,接收服务器反馈的响应,可以通过Successes和Errors通道来接受成功或失败的内容

### 同步生产者
```

config := sarama.NewConfig()
//  config.Producer.RequiredAcks = sarama.WaitForAll
//  config.Producer.Partitioner = sarama.NewRandomPartitioner
config.Producer.Return.Successes = true
config.Producer.Timeout = 5 * time.Second
p, err := sarama.NewSyncProducer(strings.Split("localhost:9092", ","), config)
defer p.Close()
if err != nil {
	fmt.Println("NewSyncProducer error=", err)
	painc(err)
}
 
v := "sync: " + strconv.Itoa(rand.New(rand.NewSource(time.Now().UnixNano())).Intn(10000))
	fmt.Fprintln(os.Stdout, v)
msg := &sarama.ProducerMessage{
	Topic: "test_topic",
	Value: sarama.ByteEncoder(v),
}
if _, _, err := p.SendMessage(msg); err != nil {
	fmt.Println("SendMessage error=", err)
	panic(err)
}
```

## 消费者

```
//配置
config := sarama.NewConfig()
//接收失败通知
config.Consumer.Return.Errors = true
//设置使用的kafka版本,如果低于V0_10_0_0版本,消息中的timestrap没有作用.需要消费和生产同时配置
config.Version = sarama.V0_11_0_0
//新建一个消费者
consumer, e := sarama.NewConsumer([]string{"IP:9092", "IP:9092", "IP:9092"}, config)
if e != nil {
    panic("error get consumer")
}
defer consumer.Close()

//根据消费者获取指定的主题分区的消费者,Offset这里指定为获取最新的消息.
    partitionConsumer, err := consumer.ConsumePartition("logstash_test", 0, sarama.OffsetNewest)
    if err != nil {
        fmt.Println("error get partition consumer", err)
    }
    defer partitionConsumer.Close()
    //循环等待接受消息.
    for {
        select {
        //接收消息通道和错误通道的内容.
        case msg := <-partitionConsumer.Messages():
            fmt.Println("msg offset: ", msg.Offset, " partition: ", msg.Partition, " timestrap: ", msg.Timestamp.Format("2006-Jan-02 15:04"), " value: ", string(msg.Value))
        case err := <-partitionConsumer.Errors():
            fmt.Println(err.Err)
        }
    }
```
* 配置
* 新建一个消费者
* 通过消费者,指定主题的分区,获取一个特定的分区消费者.
* 通过分区消费者接收消息.

## cluster消费者
```
groupID := "group-1"
topic := "test_topic"
config := cluster.NewConfig()
config.Group.Return.Notifications = true
config.Consumer.Offsets.CommitInterval = 1 * time.Second
config.Consumer.Offsets.Initial = sarama.OffsetNewest //初始从最新的offset开始
c, err := cluster.NewConsumer(strings.Split("localhost:9092", ","), groupID, strings.Split(topic, ","), config)
if err != nil {
  panic(err)
  return
}
defer c.Close()
go func(c *cluster.Consumer) {
   errors := c.Errors()
   noti := c.Notifications()
   for {
       select {
       case err := <-errors:
            panic(err)
       case <-noti:
       }
    }
}(c)
 
for msg := range c.Messages() {
   fmt.Fprintf(os.Stdout, "%s/%d/%d\t%s\n", msg.Topic, msg.Partition, msg.Offset, msg.Value)
   c.MarkOffset(msg, "") //MarkOffset 并不是实时写入kafka，有可能在程序crash时丢掉未提交的offset
}
```


## 客户端
> 客户端可以用来获取消费者和生产者,还可以获取kafka的broker信息和topic信息,以及每个topic中的offset等.  

```
config := sarama.NewConfig()
config.Version = sarama.V0_10_0_0
client, err := sarama.NewClient([]string{"IP:9092", "IP:9092", "IP:9092"}, config)
if err != nil {
    panic("client create error")
}
defer client.Close()
//获取主题的名称集合
topics, err := client.Topics()
if err != nil {
    panic("get topics err")
}
for _, e := range topics {
    fmt.Println(e)
}
//获取broker集合
brokers := client.Brokers()
//输出每个机器的地址
for _, broker := range brokers {
    fmt.Println(broker.Addr())
}
```


## sarama选项
### config结构体
```
config := sarama.NewConfig()
c.Net.MaxOpenRequests = 5
c.Net.DialTimeout = 30 * time.Second
c.Net.ReadTimeout = 30 * time.Second
c.Net.WriteTimeout = 30 * time.Second
c.Net.SASL.Handshake = true

c.Metadata.Retry.Max = 3
c.Metadata.Retry.Backoff = 250 * time.Millisecond
c.Metadata.RefreshFrequency = 10 * time.Minute
c.Metadata.Full = true

c.Producer.MaxMessageBytes = 1000000
c.Producer.RequiredAcks = WaitForLocal
c.Producer.Timeout = 10 * time.Second
c.Producer.Partitioner = NewHashPartitioner  //选择分区的分区选择器.用于选择主题的分区
c.Producer.Retry.Max = 3 //重试次数
c.Producer.Retry.Backoff = 100 * time.Millisecond
c.Producer.Return.Errors = true  //是否接收返回的错误消息,当发生错误时会放到Error这个通道中.从它里面获取错误消息

//抓取数据的大小设置
c.Consumer.Fetch.Min = 1
c.Consumer.Fetch.Default = 32768

c.Consumer.Retry.Backoff = 2 * time.Second //失败后再次尝试的间隔时间
c.Consumer.MaxWaitTime = 250 * time.Millisecond  //最大等待时间
c.Consumer.MaxProcessingTime = 100 * time.Millisecond
c.Consumer.Return.Errors = false  //是否接收返回的错误消息,当发生错误时会放到Error这个通道中.从它里面获取错误消息
c.Consumer.Offsets.CommitInterval = 1 * time.Second // 提交跟新Offset的频率
c.Consumer.Offsets.Initial = OffsetNewest // 指定Offset,也就是从哪里获取消息,默认时从主题的开始获取.

c.ClientID = defaultClientID
c.ChannelBufferSize = 256  //通道缓存大小
c.Version = minVersion //指定kafka版本,不指定,使用最小版本,高版本的新功能可能无法正常使用.
c.MetricRegistry = metrics.NewRegistry()
```

### 生产者的分区的分割器
```
sarama.NewManualPartitioner() //返回一个手动选择分区的分割器,也就是获取msg中指定的`partition`
sarama.NewRandomPartitioner() //通过随机函数随机获取一个分区号
sarama.NewRoundRobinPartitioner() //环形选择,也就是在所有分区中循环选择一个
sarama.NewHashPartitioner() //通过msg中的key生成hash值,选择分区
```

### 生产者的消息ProducerMessage
```
Topic string // kafka 主题
Key Encoder //用于选择分区,和分割器的NewHashPartitioner联合使用,决定当前消息被保存在哪个分区
Value Encoder  //消息的内容.

Headers []RecordHeader //在生产者和消费者之间传递的键值对,

Metadata interface{} //sarama 用于传递数据使用

//下面的内容有生产者返回后的内容填充.
Offset int64 // 返回新发布的消息的偏移量
Partition int32 //返回的信息的保存分区
Timestamp time.Time //保存在服务端的消息时间

retries int
flags   flagSet
```

### 消费者信息ConsumerMessage
```
// ConsumerMessage encapsulates a Kafka message returned by the consumer.
type ConsumerMessage struct {
    Key, Value     []byte  //key和保存的值
    Topic          string //要消费的主题
    Partition      int32 //要消费的分区
    Offset         int64 //要消费的消息的位置,从哪里开始消费,最开始的,还是最后的
    Timestamp      time.Time       // only set if kafka is version 0.10+, 内部时间
    BlockTimestamp time.Time       // only set if kafka is version 0.10+, outer (compressed) block timestamp
    Headers        []*RecordHeader // only set if kafka is version 0.11+
}
```