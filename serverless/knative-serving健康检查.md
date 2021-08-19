# knative-serving(健康检查)

![](knative-serving%E5%81%A5%E5%BA%B7%E6%A3%80%E6%9F%A5/079AB471-9D32-46E5-81D1-FCC66C1F30BC.png)
> 咱们以http1. 为例进行说明。业务流量首先进入Istio Gateway，然后会转发到Queue-Proxy的8012端口，Queue-Proxy8012再把请求转发到user-con- tainer的监听端口，至此一个业务请求的服务就算完成了。   
粗略的介绍原理基本就是上面这样，现在咱们对几个细节进行深入的剖析看看其 内部机制: 
* 为什么要引入Queue-Proxy ? 
* Pod缩容到零的时候流量会转发到Activator上面，那么Activator是怎么处
理这些请求的?
* Knative中的业务Pod有Queue-Proxy和user-container,那 么Pod的readinessProber和LivenessProber分别是怎么做的? Pod的readinessProber、LivenessProber和业务的健康状态是什么样的关系? 
* IstioGateway向Pod转发流量的时候是怎么选择Pod进行转发的?

## 为什么要引入Queue-Proxy ?
Serverless的一个核心诉求就是把业务的复杂度下沉到基础平台，让业务代码 快速的迭代并且按需使用资源。不过现在更多的还是聚焦在按需使用资源层面。 
如果想要按需使用资源我们就需要收集相关的Metrics，并根据这些Metrics信息来指导资源的伸缩。Knative首先实现的就是KPA策略，这个策略是根据请求数来判断是否需要扩容的。所以Knative需要有一个机制收集业务请求数量。除了业务请求数还有如下信息也是需要统一处理:
* 访问日志的管理 
* Tracing
* Pod健康检查机制 
* 需要实现Pod和Activator的交互，当Pod缩容到零的时候如何接收
Activator转发过来的流量
* 其他诸如判断Ingress是否Ready的逻辑也是基于Queue-Proxy实现的 
为了保持和业务的低耦合关系，还需要实现上述这些功能所以就引入了Queue- Proxy负责这些事情。这样可以在业务无感知的情况下把Serverless的功能实现。 

## 从零到一的过程
当Pod缩容到零的时候流量会指到Activator上面，Activator接收到流量以后会主动“通知”Autoscaler做一个扩容的操作。扩容完成以后Activator会探测Pod 的健康状态，需要等待第一个Pod ready之后才能把流量转发过来。所以这里就出现了第一个健康检查的逻辑:Activator检查第一个Pod是否ready。 
这个健康检查是调用的Pod 8012端口完成的，Activator会发起HTTP的健康检查，并且设置K-Network-Probe=queue Header，所以QueueContainer中会根据K-Network-Probe=queue来判断这是来自Activator的检查，然后执行相 应的逻辑。 

## VirtualService的健康检查
Knative Revision部署完成以后就会自动创建一个Ingress(以前叫做 ClusterIngress),这个Ingress最终会被IngressController解析成Istio的VirtualService配置，然后IstioGateway才能把相应的流量转发给相关的Revision。 
所以每添加一个新的Revision都需要同步创建Ingress和Istio的VirtualService，而VirtualService是没有状态表示Istio的管理的Envoy是否配置生效的能力的。所以IngressController需要发起一个http请求来监测VirtualService是否 ready。这个http的检查最终也会打到Pod的8012端口上。标识Header是K-Network-Probe=probe。Queue-Proxy需要基于此来判断，然后执行相应的 逻辑。 
相关代码如下所示: 
![](knative-serving%E5%81%A5%E5%BA%B7%E6%A3%80%E6%9F%A5/F1B1FD7B-47D2-4765-9713-080B4826257F.png)
![](knative-serving%E5%81%A5%E5%BA%B7%E6%A3%80%E6%9F%A5/4069F46E-011D-4FD7-82F7-51CC00D627EC.png)

## Kubelet的健康检查
Knative最终生成的Pod是需要落实到Kubernetes集群的，Kubernetes中Pod有两个健康检查的机制ReadinessProber和LivenessProber。 其中LivenessProber是判断Pod是否活着，如果检查失败Kubelet就会尝试重启 Container，ReadinessProber是来判断业务是否Ready，只有业务Ready的情 况下才会把Pod挂载到Kubernetes Service的EndPoint中，这样可以保证Pod故障时对业务无损。 
那么问题来了，Knative的Pod中默认会有两个Container Queue-Proxy和 user-container。前面两个健康检查机制你应该也发现了，流量的“前半路径”需要通过Queue-Proxy来判断是否可以转发流量到当前Pod，而在Kubernetes 的机制中Pod是否加入Kubernetes Service EndPoint中完全是由ReadinessProber的结果决定的。而这两个机制是独立的，所以我们需要有一种方案来把这两 个机制协调一致。这也是Knative作为一个Serverless编排引擎是需要对流量做更精细的控制要解决的问题。所以Knative最终是把user-container的ReadinessProber收敛到Queue-Proxy中，通过Queue-Proxy的结果来决定Pod的状态。 
另外https://github.com/knative/serving/issues/2912 这个Issue中也提到在启动istio的情况下，kubelet发起的tcp检查可能会被Envoy拦截，所以给user-container配置TCP探测器判断user-container是否ready也是不准的。 这也是需要把Readiness收敛到Queue-Proxy的一个动机。
Knative收敛user-container健康检查能力的方法是:
* 置空user-container的ReadinessProber
* 把user-container的ReadinessProber配置的json String配置到Queue-Proxy的env中
* Queue-Proxy的Readinessprober命令里面解析user-container的ReadinessProber的jsonString然后实现健康检查逻辑。并且这个检查的机制和前面提到的Activator的健康检查机制合并到了一起。这样做也保证了Activator向Pod转发流量时user-container一定是Ready状态 
### 使用方法
如下所示可以在KnativeService中定义Readiness 
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: readiness-prober 
spec:
  template:
    metadata:
	  labels: 
        app: helloworld-go 
  spec:
    containers:				
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/helloworld-go:160e4db7 
        readinessProbe:
          httpGet:
           path: /
           initialDelaySeconds: 3 
```
但是需要说明两点:
* 和原生的KubernetesPodReadiness配置相比，Knative中timeout-Seconds、failureThreshold、periodSeconds和successThreshold 如果要配置就要一起配置，并且不能为零，否则Knative webhook校验无法通过。并且如果设置了periodSeconds那么一旦出现一次Success，就再也不会去探测user-container( 不建议设置periodSeconds，应该让系统自动处理) 
* 如果periodSeconds没有配置那么就会使用默认的探测策略，默认配置如下:

```
timeoutSeconds: 60
failureThreshold: 3
periodSeconds: 10
successThreshold: 1 
```
从这个使用方式上来看其实Knative是在逐渐收敛user-container配置，因为 在Serverless模式中需要系统自动化处理很多逻辑，这些“系统行为”就不需要麻 烦用户了。 

## 总结
### 前面提到的三种健康检查机制的对比关系:
![](knative-serving%E5%81%A5%E5%BA%B7%E6%A3%80%E6%9F%A5/2144B752-6671-4EF3-8F0C-8FD33A385E91.png)