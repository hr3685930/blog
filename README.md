# 个人博客
* [基础设施](infrastructure/README.md)
    * [openVPN安装](infrastructure/openVPN安装.md)
    * [corntab定时任务](infrastructure/corntab定时任务.md)
    * [Supervisor进程守护工具](infrastructure/supervisor.md)
    * [Prometheus+Grafana](infrastructure/Prometheus+Grafana.md)
    * [iptables](infrastructure/iptables.md)
    * [nginx配置](infrastructure/nginx配置.md)
    * [AB压测](infrastructure/AB压测.md)
    * [负载均衡](infrastructure/负载均衡.md)
    * [ELK安装](infrastructure/ELK安装.md)
    * [shell査漏](infrastructure/shell査漏.md)
* [协议](protocol/README.md)
    * [常用HTTP](protocol/常用HTTP.md)
    * [rpc](protocol/rpc.md)
    * [Gossip](protocol/Gossip协议.md)
    * [websocket](protocol/websocket.md)
    * [TCP和UDP区别](protocol/TCP-UDP.md)
    * [SSL证书链不完整问题](protocol/SSL证书链不完整问题.md)
    * [网络编程](protocol/网络编程.md)
    * [网络协议](protocol/网络协议.md)
* [容器](docker/README.md)
    * [Docker RUN、CMD 和 ENTRYPOINT区别](docker/qubei.md)
    * [Docker Swarm](docker/ds.md)
    * [Docker Stack](docker/DockerStack.md)
    * [容器-kubernetes篇](docker/容器-kubernetes篇.md)
    * [容器-Docker篇](docker/容器-Docker篇.md)
    * [k8s configmap](docker/k8sconfigmap.md)
    * [k8s 滚动更新](docker/k8s滚动更新.md)
    * [K8s pod探针](docker/K8spod探针.md)
    * [harbor私有仓库](docker/harbor私有仓库.md)
    * [Helm2](docker/Helm2.md)
    * [Helm3](docker/Helm3.md)
    * [理解 K8S 的设计精髓之 List-Watch机制和Informer模块](docker/List-Watch.md)
    * [k8s-链路(tracing)](docker/k8s-tracing.md)
    * [k8s-日志(logging)](docker/k8s-logging.md)
    * [k8s-监控(metrics)](docker/k8s-metrics.md)
    * [Cloud Native](docker/Cloud-Native.md)
* [PHP](laravel/README.md)
    * [Dingo API](laravel/DingoAPI.md)
    * [Laravel队列](laravel/Laravel队列.md)
    * [Laravel elasticsearch](laravel/Laravel-elasticsearch.md)
    * [基于RAML的接口描述规范 及 Doc&Mock服务构建](laravel/Doc&Mock服务构建.md)
    * [PHP常见运行模式](laravel/PHP常见运行模式.md)
    * [php集成测试](laravel/php集成测试.md)
* [Golang](golang/README.md)
    * [Go Modules依赖管理](golang/GoModules依赖管理.md)
    * [GO GPM模型](golang/GOGPM模型.md)
    * [进程、线程、协程](golang/进程、线程、协程.md)
* [分布式](distributed/README.md)
    * [CAP理论](distributed/CAP理论.md)
    * [分布式事务](distributed/分布式事务.md)
    * [分布式-锁](distributed/分布式-锁.md)
        * [Etcd-分布式锁](distributed/ETCD-分布式锁.md)
        * [Redis-分布式锁](distributed/redis-分布式锁.md)
    * [分布式-负载均衡](distributed/分布式-负载均衡.md)
    * [分布式-搜索引擎](distributed/分布式-搜索引擎.md)
    * [分布式-延时任务系统](distributed/分布式-延时任务系统.md)
    * [分布式-ID生成器](distributed/分布式-ID生成器.md)
    * [分布式事务协议](distributed/分布式事务协议.md)
    * [分布式高可靠](distributed/分布式高可靠.md)
    * [分布式存储](distributed/分布式存储.md)
    * [分布式通信](distributed/分布式通信.md)
    * [分布式计算-MapReduce、Stream、Actor 和流水线](distributed/分布式计算.md)
    * [分布式资源管理与负载调度](distributed/分布式资源管理与负载调度.md)
    * [Bully 算法、Raft算法、ZAB算法](distributed/sf.md)
* [架构设计](architecture_design/README.md)
	* [架构基础](architecture_design/架构基础.md)
	* [单服务器高性能模式:PPC与TPC](architecture_design/单服务器高性能模式PPC与TPC.md)
	* [单服务器高性能模式：Reactor与Proactor](architecture_design/单服务器高性能模式：Reactor与Proactor.md)
	* [高性能负载均衡](architecture_design/高性能负载均衡.md)
	* [FMEA故障模式与影响分析](architecture_design/FMEA故障模式与影响分析.md)
	* [计算高可用、存储高可用、业务高可用](architecture_design/计算高可用、存储高可用、业务高可用.md)
	* [并发数 QPS TPS](architecture_design/并发数.md)
	* [应对接口级故障-降级、熔断、限流、排队](architecture_design/应对接口级故障-降级、熔断、限流、排队.md)
	* [架构之可扩展性](architecture_design/架构之可扩展性.md)
	* [架构设计文档模板](architecture_design/架构设计文档模板.md)
	* [技术演进的模式](architecture_design/技术演进的模式.md)
	* [应用可靠性保障评估](infrastructure/应用可靠性保障评估.md)
* [数据存储服务](db/README.md)
    * [redis](db/redis.md)
    * [Redis-Cluster](db/Redis-Cluster.md)
    * [redis-内存淘汰机制](db/redis-内存淘汰机制.md)
    * [Redis优化](db/Redis优化.md)
    * [TIDB](db/TIDB.md)
    * [mysql调优](db/mysql调优.md)
    * [mysql慢查询](db/mysql慢查询.md)
    * [mysql二进制日志-binlog](db/mysql二进制日志-binlog.md)
    * [mysql 高可用集群galera cluster](db/mysql高可用集群galeracluster.md)
    * [ETCD API](db/ETCD-API.md)
* [消息中间件](messages/README.md)
    * [RabbitMQ](messages/RabbitMQ.md)
    * [Kafka](messages/Kafka.md)
    * [rabbitmq和kafka的区别](messages/rabbitmq和kafka的区别.md)
    * [Kafka API](messages/KafkaAPI.md)
    * [Kafka幂等性和事务性](messages/Kafka幂等性和事务性.md)
* [微服务](msa/README.md)
    * [DDD-领域驱动建模](msa/DDD-领域驱动建模.md)
    * [服务发现与注册](msa/服务发现与注册.md)
    * [kong实践](msa/kong实践.md)
    * [istio-简介](msa/istio-简介.md)
    * [istio-安装](msa/istio-安装.md)
    * [istio-配置请求的路由规则](msa/istio-配置请求的路由规则.md)
    * [istio-日志采集](msa/istio-日志采集.md)
    * [istio-故障注入](msa/istio-故障注入.md)
    * [微服务改造](msa/微服务改造.md)
* [serverless](serverless/README.md)
    * [knative-API聚合层(转)](serverless/knative-API.md)
    * [Knative-eventing](serverless/Knative-eventing.md)
        * [Knative Eventing原理](serverless/Knative-Eventing原理.md)
    * [Knative-serving](serverless/Knative-serving.md)
        * [knative-serving(Serving Client)](serverless/knative-Serving-Client.md)
        * [knative-serving(WebSocket和gRPC服务)](serverless/knative-servingWebSocket和gRPC服务.md)
        * [knative-serving(健康检查)](serverless/knative-serving健康检查.md)
        * [knative-serving(Autoscaler)](serverless/knative-servingAutoscaler.md)
        * [knative-serving(流量灰度和版本管理)](serverless/knative-serving流量灰度和版本管理.md)
        * [knative-serving(服务路由管理)](serverless/knative-serving服务路由管理.md)
    * [Tekton原理](serverless/Tekton原理.md)
* [devops](devops/README.md)
    * [Gitlab Auto DevOps(安装)](devops/Gitlab-Auto-DevOps安装.md)
    * [Gitlab Auto DevOps(组件)](devops/Gitlab-Auto-DevOps组件.md)
* [大数据](bigdata/README.md)
    * [CDH坑点记录](bigdata/CDH坑点记录.md)
    * [Apache Flink](bigdata/Apache-Flink.md)
    * [HBase原理](bigdata/HBase原理.md)
    * [flume 日志采集记录](bigdata/flume日志采集记录.md)
* [随笔](other/README.md)
    * [时间管理](other/time.md)