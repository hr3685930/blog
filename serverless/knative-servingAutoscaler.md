# knative-serving Autoscaler
> Knative.Serving默认情况下，提供了开箱即用的快速、基于请求的自动扩缩 容功能-Knative.Pod.Autoscaler(KPA)。下面带你体验如何在Knative中配置Autoscaler

## Autoscaler机制

KnativeServing为每个POD注入QUEUE代理容器 (queue-proxy)，该容 器负责向Autoscaler报告用户容器并发指标。Autoscaler接收到这些指标之后，会 根据并发请求数及相应的算法，调整Deployment的POD数量，从而实现自动扩缩容。

![](knative-servingAutoscaler/DC43B318-4D47-47F5-BDA0-7811787B54F6.png)
### 算法
Autoscaler基于每个POD的平均请求数(并发数)。默认并发数为100。
POD数 = 并发请求总数 / 容器并发数。
如果服务中并发数设置了10，这时候如果加载了50个并发请求的服务，
Autoscaler就会创建了5个POD(50个并发请求/10=POD)。
Autoscaler实现了两种操作模式的缩放算法:Stable/稳定模式和Panic/恐慌模式。

1. 稳定模式
在稳定模式下，Autoscaler调整Deployment的大小，以实现每个POD所需的平均并发数。POD的并发数是根据60秒窗口内接收所有数据请求的平均数来计算得出。
2. 恐慌模式
Autoscaler计算60秒窗口内的平均并发数，系统需要1分钟稳定在所需的并发级别。但是，Autoscaler也会计算6秒的恐慌窗口，如果该窗口达到目标并发的2倍，则会进入恐慌模式。在恐慌模式下，Autoscaler在更短、更敏感的紧急窗口上工作。一旦紧急情况持续60秒后，Autoscaler将返回初始的60秒稳定窗口。
![](knative-servingAutoscaler/A3180EE7-76B1-4267-9B40-AAFCF5C8283F.png)

### 配置KPA
通过上面的介绍，我们对Knative Pod Autoscaler工作机制有了初步的了解， 那么接下来介绍如何配置KPA。在Knative中配置KPA信息，需要修改k8s中的 ConfigMap:config-autoscaler，该ConfigMap在knative-serving命名空间下。
```
kubectl -n knative-serving get cm config-autoscaler
apiVersion: v1
kind: ConfigMap
metadata:
    name: config-autoscaler
    namespace: knative-serving
data:
    container-concurrency-target-percentage: “70”
    container-concurrency-target-default: “100”
    requests-per-second-target-default: “200”
    target-burst-capacity: “200”
    stable-window: “60s”
    panic-window-percentage: “10.0”
    panic-threshold-percentage: “200.0”
    max-scale-up-rate: “1000.0”
    max-scale-down-rate: “2.0”
    enable-scale-to-zero: “true”
    tick-interval: “2s”
    scale-to-zero-grace-period: “30s”
```

* container-concurrency-target-percentage:容器并发请求数比例。容器实际最大并发数=容器最大并发请求数*容器并发请求数比例。例如，Revision设置的容器最大并发请求数为10，容器并发请求数比例为70%，那么在稳定状态下，实际容器的最大并发请求数为7。

* container-concurrency-target-default: 容器并发请求默认值。 当Revision中未设置容器最大并发请求数时，使用该默认值作为容器最大并发请求数

* requests-per-second-target-default:每秒请求并发(RPS)默认值。当使用 RPS 进行度量时，autoscaler会依据此值进行扩缩容判断。

* target-burst-capacity:突发请求容量。在突发流量场景下，切换到Activator模式进行流量控制。取值范围为 [-1,+ ∞ )。-1 表示一直使用Activator模式;0表示不使用突发流量功能。

* stable-window:稳定窗口期。稳定模式窗口期。

* panic-window-percentage:恐慌窗口比例。通过恐慌窗口比例，计算恐慌窗口期。恐慌窗口期=恐慌窗口比例*稳定窗口期 /100

* panic-threshold-percentage:恐慌模式比例阈值。当前并发请求数大于容器最大并发请求数*恐慌比例阈值，并且达到恐慌窗口期，则进入恐慌模式。

* max-scale-up-rate:最大扩容比例。每次扩容允许的最大速率。当前最大扩容数=最大扩容比例*Ready 的 Pod 数量。

* max-scale-down-rate:最大缩容比例。

* enable-scale-to-zero:允许缩容至0。

* tick-interval:扩缩容计算间隔。

* scale-to-zero-grace-period:缩容至 0 优雅下线时间。

### 为KPA配置缩容至0
为了正确配置使Revision缩容为0，需要修改ConfigMap中的如下参数
- scale-to-zero-grace-period
scale-to-zero-grace-period 表示在缩为 0 之前，inactive revison保留的运行时间（最小是30s）。
scale-to-zero-grace-period: 30s
- stable-window
当在 stable mode 模式运行中，autoscaler 在稳定窗口期下平均并发数下的操作。
stable-window: 60s
stable-window 同样可以配置在 Revision 注释中。
autoscaling.knative.dev/window: 60s
- enable-scale-to-zero
保证 enable-scale-to-zero 参数设置为 true
- Termination period
Termination period（终止时间）是 POD 在最后一个请求完成后关闭的时间。POD 的终止周期等于稳定窗口值和缩放至零宽限期参数的总和。在本例中，Termination period 为 90 秒

### 配置并发数
可以使用以下方法配置 Autoscaler 的并发数：
- target
Target 定义在给定时间（软限制）需要多少并发请求，是 Knative 中 Autoscaler 的推荐配置。
在 ConfigMap 中默认配置的并发 target 为 100。
container-concurrency-target-default: 100
这个值可以通过 Revision 中的 autoscaling.knative.dev/target 注释进行修改：
autoscaling.knative.dev/target: 50
/
- containerConcurrency
**注意**：只有在明确需要限制在给定时间有多少请求到达应用程序时，才应该使用 containerConcurrency (容器并发)。只有当应用程序需要强制的并发约束时，才建议使用 containerConcurrency。
containerConcurrency 限制在给定时间允许并发请求的数量（硬限制），并在 Revision 模板中配置。
containerConcurrency: 0 | 1 | 2-N
1: 将确保一次只有一个请求由 Revision 给定的容器实例处理；
2-N: 请求的并发值限制为2或更多；
0: 表示不作限制，有系统自身决定。

### 配置扩缩容边界（minScale 和 maxScale）
通过 minScale 和 maxScale 可以配置应用程序提供服务的最小和最大 Pod 数量。通过这两个参数配置可以控制服务冷启动或者控制计算成本。
minScale 和 maxScale 可以在 Revision 模板中按照以下方式进行配置：
```
spec:
   template:
      metadata:
         autoscaling.knative.dev/minScale: “2”
         autoscaling.knative.dev/maxScale: “10”
```
通过在 Revision 模板中修改这些参数，将会影响到 PodAutoscaler 对象，这也表明在无需修改 Knative Serving 系统配置的情况下，PodAutoscaler 对象是可被修改的。
edit podautoscaler <revision-name>
注意：这些注释适用于 Revision 的整个生命周期。即使 Revision 没有被任何 route 引用，minscale 指定的最小 POD 计数仍将提供。请记住，不可路由的 Revision 可能被垃圾收集掉。

### 默认情况
如果未设置 minscale 注释，pods 将缩放为零（如果根据上面提到的 configmap，enable-scale-to-zero 为 false，则缩放为 1）。
如果未设置 maxscale 注释，则创建的 Pod 数量将没有上限。

## 基于KPA配置的示例


```
service.yaml
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
name: autoscale-go
    namespace: default
spec:
    template:
      metadata:
      labels:
        app: autoscale-go
      annotations:
        autoscaling.knative.dev/target: “10”
      spec:
        containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/autoscale-go:0.1
```


### 场景1:并发请求示例
如上配置，当前最大并发请求数10。我们执行30s内保持50个并发请求，看一下执行情况: 
```
hey -z 30s -c 50   -host “autoscale-go.default.example.com”   “url”
```
![](knative-servingAutoscaler/abb463e97c01cb0f4aef99c72c22797d394.jpg.gif)
结果正如我们所预期的:扩容出来了5个POD。 
### 场景2:扩缩容边界示例
```
apiVersion:
serving.knative.dev/v1alpha1
kind: Service
metadata:
name: autoscale-go
  namespace: default
spec:
  template:
    metadata:
      labels:
        app: autoscale-go
      annotations:
        autoscaling.knative.dev/target: “10”
        autoscaling.knative.dev/minScale: “1”
        autoscaling.knative.dev/maxScale: “3”
    spec:
  	  containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/autoscale-go:0.1
```
当前最大并发请求数10，minScale最小保留实例数为1，maxScale最大扩容实例数为3。
我们依然执行30s内保持50个并发请求，看一下执行情况:
```
hey -z 30s -c 50   -host “autoscale-go.default.example.com”   "url"
```
![](knative-servingAutoscaler/ac54ff2f5edca3079c0ebcc24d12670f489.jpg.gif)
结果如我们所预期：最多扩容出来了3个POD，并且即使在无访问请求流量的情况下，保持了1个运行的POD。
## 总结
看了上面的介绍，是不是感觉在 Knative 中配置应用扩缩容是如此简单？其实 Knative 中除了支持 KPA 之外，也支持K8s HPA。你可以通过如下配置基于 CPU 的 Horizontal POD Autoscaler（HPA）。
通过在修订模板中添加或修改 autoscaling.knative.dev/class 和 autoscaling.knative.dev/metric 值作为注释，可以将Knative 配置为使用基于 CPU 的自动缩放，而不是默认的基于请求的度量。配置如下:
```
spec:
  template:
    metadata:
      autoscaling.knative.dev/metric: concurrency
      autoscaling.knative.dev/class: hpa.autoscaling.knative.dev
```
你可以自由的将 Knative Autoscaling 配置为使用默认的 KPA 或 Horizontal POD Autoscaler（HPA）。