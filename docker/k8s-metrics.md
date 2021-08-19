# k8s-监控(metrics)
## 监控
> Metrics - 用于记录可聚合的数据.例如,队列的当前深度可被定义为一个度量值,在元素入队或出队时被更新.HTTP 请求个数可被定义为一个计数器,新请求到来时进行累加.  

### 监控类型
> 比较常见的像 CPU、内存、网络这种资源类的一个指标,通常这些指标会以数值、百分比的单位进行统计,是最常见的一个监控方式.这种监控方式在常规的监控里面,类似项目 zabbix,这些系统都是可以做到的.  

- 性能监控
性能监控指的就是 APM 监控,也就是说常见的一些应用性能类的监控指标的检查.通常是通过一些 Hook 的机制在虚拟机层、字节码执行层通过隐式调用,或者是在应用层显示注入,获取更深层次的一个监控指标,一般是用来应用的调优和诊断的,比较常见的类似像 jvm 或者 php 的 Zend Engine,通过一些常见的 Hook 机制,拿到类似像 jvm 里面的 GC 的次数,网络连接数的一些指标，通过这种方式来进行应用的性能诊断和调优.

- 安全监控
安全监控主要是对安全进行的一系列的监控策略,类似像越权管理、安全漏洞扫描等等.

- 事件监控
事件监控是 K8s 中比较另类的一种监控方式.为大家介绍了在 K8s 中的一个设计理念,就是基于状态机的一个状态转换.从正常的状态转换成另一个正常的状态的时候,会发生一个 normal 的事件,而从一个正常状态转换成一个异常状态的时候,会发生一个 warning 的事件.通常情况下warning 的事件是我们比较关心的,而事件监控就是可以把 normal 的事件或者是 warning 事件离线到一个数据中心,然后通过数据中心的分析以及报警,把相应的一些异常通过像钉钉或者是短信、邮件的方式进行暴露,弥补常规监控的一些缺陷和弊端.
 
### 监控指标
- 核心指标
从 Kubelet、cAdvisor 等获取度量数据, 再由metrics-server提供给 Dashboard、HPA 控制器等使用.

- 自定义指标
由Prometheus Adapter提供API custom.metrics.k8s.io,由此可支持任意Prometheus采集到的指标.

> 核心指标只包含node和pod的cpu、内存等,一般来说,核心指标作HPA已经足够,但如果想根据自定义指标:如请求qps/5xx错误数来实现HPA,就需要使用自定义指标了,目前Kubernetes中自定义指标一般由Prometheus来提供,再利用k8s-prometheus-adpater聚合到apiserver,实现和核心指标（metric-server)同样的效果.  

-  如何获取监控数据？
Metrics-Server通过kubelet获取监控数据.

- 如何提供监控数据？
Metrics-Server通过metrics API提供监控数据.

先说下API聚合机制,API聚合机制是k8s 1.7版本引入的特性,能将用户扩展API注册至API Server上.
API Server在此之前只提供k8s资源对象的API,包括资源对象的增删查改功能.举例来说,yaml配置文件中的apiVersion字段描述的即是API名.有了API聚合机制之后,用户可以发布自己的API,而Metrics-Server用到的metrics API和custom metrics API均属于API聚合机制的应用,用户可通过配置APIService资源对象以使用API聚合机制,如下是metrics API的配置文件：
```
apiVersion: apiregistration.k8s.io/v1beta1
kind: APIService
metadata:
  name: v1beta1.metrics.k8s.io
spec:
  service:
    name: metrics-server
    namespace: kube-system
  group: metrics.k8s.io
  version: v1beta1
  insecureSkipTLSVerify: true
  groupPriorityMinimum: 100
  versionPriority: 100
```

如上,APIService提供了一个名为v1beta1.metrics.k8s.io的API，并绑定至一个名为metrics-server的Service资源对象.可以通过kubectl get apiservices命令查询集群中的APIService.因此,访问Metrics-Server的方式如下：
```
发起请求 -> API Server (/apis/metrics.k8s.io/v1beta1) -> Service: metrics-server (metrics-server.kube-system.svc) -> Pod：metrics-server-xxx-xxx 
```
有了访问Metrics-Server的方式,HPA kubectl top 等对象就可以正常工作了.

#### 部署前准备
> Prometheus可以采集其它各种指标,但是prometheus采集到的metrics并不能直接给k8s用,因为两者数据格式不兼容,因此还需要另外一个组件(kube-state-metrics),将prometheus的metrics数据格式转换成k8s API接口能识别的格式,转换以后,因为是自定义API,所以还需要用Kubernetes aggregator在主API服务器中注册,以便直接通过/apis/来访问.  

1.  相关组件
- cAdvisor
> cAdvisor可以对节点机器上的资源及容器进行实时监控和性能数据采集,包括CPU使用情况、内存使用情况、网络吞吐量及文件系统使用情况,在K8S中集成在Kubelet里作为默认启动项,官方标配.  
- node-exporter
> prometheus的export,收集Node级别的监控数据  
- prometheus
> 监控服务端,从node-exporter拉数据并存储为时序数据.  
- kube-state-metrics
> prometheus中可以用PromQL查询到的指标数据转换成k8s对应的数据格式,即转换成Custerom Metrics API接口格式的数据,但是它不能聚合进apiserver中的功能  
- k8s-prometheus-adpater
> 即提供了一个apiserver, cuester-metrics-api,自定义APIServer通常都要通过Kubernetes aggregator聚合到apiserver  
- grafana
> 展示prometheus获取到的metrics  


