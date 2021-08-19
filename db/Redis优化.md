# Redis 优化

![](Redis%E4%BC%98%E5%8C%96/F1ED51ED-4625-4904-9402-CA8A3B5A6969.png)

第1次优化(hmset命令将两条hmset合并)

![](Redis%E4%BC%98%E5%8C%96/306D4287-F7A1-4B44-BF5B-61375772E0B0.png)

第2次优化(set和expire合并)

![](Redis%E4%BC%98%E5%8C%96/0569D373-0D4E-433B-BD78-77AA3BD88852.png)

第3次优化(pipeline)
> 注意的是在RedisCluster中使用pipeline时必须满足pipeline打包的所有命令key在RedisCluster的同一个slot上。  
如果打包命令的key不在同一个slot上，就会报错。所以我们需要分两批打包：
![](Redis%E4%BC%98%E5%8C%96/46F7A990-2513-4E2C-821F-EDD99445E3CB.png)
这些命令还是需要2次网络交互

第4次优化(hashtag)
什么意思呢？我们知道，RedisCluster总计有16*1024=16384个slot。那么执行一条Redis命令时，其key对应的是哪个slot呢？是利用这样一个计算公式得到的：slot = CRC16(key)%16384
也就是说，默认情况下，key在哪个slot上，与key有关。那么，我们能否只让key在哪个slot上与部分key有关呢？

当然可以，这就是hashtag特性。用法非常简单，假设一个key是mall:sale:freq:ctrl:860000000000001，我们只需要用{}将key中我们需要的那部分包括起来即可。
例如，我们只想让其根据用户IMEI计算即可，那么key是这样的：mall:sale:freq:ctrl:{860000000000001}。只要key中有{860000000000001}这一部分，就一定落在同一个slot上。

![](Redis%E4%BC%98%E5%8C%96/D76AE500-F82D-4C21-B05C-0688EC2513AD.png)
优化后，5条Redis命令压缩到3条Redis命令，并且3条Redis命令只需要发送一次，并且结果也一次就能全部返回。简直完美！！


* 注意事项
我们在使用hashtag特性时，一定要注意，不能把key的离散性变得非常差。
以本文为例，没有利用hashtag特性之前，key是这样的：mall:sale:freq:ctrl:860000000000001，
很明显这种key由于与用户相关，所以离散性非常好。
而使用hashtag以后，key是这样的：
mall:sale:freq:ctrl:{860000000000001}，
这种key还是与用户相关，所以离散性依然非常好。
我们千万不要这样来使用hashtag特性，例如将key设置为：
mall:{sale:freq:ctrl}:860000000000001。
这样的话，无论有多少个用户多少个key，
其{}中的内容完全一样都是sale:freq:ctrl，
也就是说，所有的key都会落在同一个slot上，
导致整个Redis集群出现严重的倾斜问题。