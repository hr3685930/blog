# AB压测
## 压测命令
```
#连续发起n个请求，每个请求实施c次并发。一个请求结束后立即进行下一个请求。
ab -c 100 -n 1000 www.baidu.com/    #压测首页，*注意别漏掉最后的斜杠*
ab -c 100 -n 1000 ‍‍www.baidu.co‍‍m/index.html    #压测某一页面
```

### 主要关注三个值
* Requests per second : 每秒最多能处理几个Concurrency连接（QPS）
* 第一个Time per request : 平均每个请求的时间，是该例中一个请求（100个Concurrency连接）的耗时
* 第二个Time per request: 平均每个并发连接的时间，是该例中一个Concurrency连接的耗时

### 需要留意以下两点
* ab命令主要对被测试方有负载压力，而对发起方则几乎没有压力
* 该命令可以轻易击垮没有任何防护的普通站点

## 测试QPS
* 一般 -n 参数取10000次请求， 将 -c 参数从小到大测试
* top 命令监控主机资源消耗情况
* 当主机的 CPU、内存 某个资源消耗将近100%满负荷时即为站点的可支撑QPS

## QPS & 并发
* QPS = 并发 / 请求平均响应时间

## 查看并发
```
#通过当前web服务连接数来获取并发情况
netstat -anp | grep ESTABLISHED | wc -l
```