# Knative-serving
> knative是一个大熔炉，将DevOps构建到服务的自动弹性伸缩，流量管控，事件驱动等都整合到一起。  


## 安装
```
kubectl apply --selector knative.dev/crd-install=true \
   --filename https://github.com/knative/serving/releases/download/v0.8.0/serving.yaml \
   --filename https://github.com/knative/eventing/releases/download/v0.8.0/release.yaml \
   --filename https://github.com/knative/serving/releases/download/v0.8.0/monitoring.yaml


kubectl apply --filename https://github.com/knative/serving/releases/download/v0.8.0/serving.yaml \
   --filename https://github.com/knative/eventing/releases/download/v0.8.0/release.yaml \
   --filename https://github.com/knative/serving/releases/download/v0.8.0/monitoring.yaml
```

## Autoscaler 机制
> Knative Serving 为每个 POD 注入 QUEUE 代理容器 (queue-proxy)，该容器负责向 Autoscaler 报告用户容器并发指标。Autoscaler 接收到这些指标之后，会根据并发请求数及相应的算法，调整 Deployment 的 POD 数量，从而实现自动扩缩容。  

## 配置 KPA
> kubectl -n knative-serving get cm config-autoscaler  

默认的 ConfigMap 如下：
```
apiVersion: v1
kind: ConfigMap
metadata:
 name: config-autoscaler
 namespace: knative-serving
data:
 container-concurrency-target-default: 100  //默认配置的并发 target 为 100。
 container-concurrency-target-percentage: 1.0
 enable-scale-to-zero: true  //保证 enable-scale-to-zero 参数设置为 true
 enable-vertical-pod-autoscaling: false
 max-scale-up-rate: 10
 panic-window: 6s
 scale-to-zero-grace-period: 30s   //scale-to-zero-grace-period 表示在缩为 0 之前，inactive revison 保留的运行时间(最小是30s)
 stable-window: 60s  //当在 stable mode 模式运行中，autoscaler 在稳定窗口期下平均并发数下的操作 
 tick-interval: 2s
```

### service对应的参数
```
stable-window 同样可以配置在 Revision 注释中。
autoscaling.knative.dev/window: 60s
这个值可以通过 Revision 中的 autoscaling.knative.dev/target 注释进行修改：
autoscaling.knative.dev/target: 50
通过 minScale 和 maxScale 可以配置应用程序提供服务的最小和最大 Pod 数量。通过这两个参数配置可以控制服务冷启动或者控制计算成本。
minScale 和 maxScale 可以在 Revision 模板中按照以下方式进行配置：
spec:
  template:
    metadata:
      autoscaling.knative.dev/minScale: "2"
      autoscaling.knative.dev/maxScale: "10"
```


### 扩缩容边界示例
```
修改一下 servcie.yaml 配置如下：
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
        autoscaling.knative.dev/target: "10"
        autoscaling.knative.dev/minScale: "1"
        autoscaling.knative.dev/maxScale: "3"		
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/autoscale-go:0.1
```

### 服务路由管理
1. 首先需要部署一个 Knative Service
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: helloworld

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: hello
  namespace: helloworld
spec:
  template:
    metadata:
      labels:
        app: hello
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/simple-app:132e07c14c49
          env:
            - name: TARGET
              value: "World!"

```


```
# kubectl -n helloworld get ksvc
NAME    URL                                   LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.helloworld.example.com   hello-wsnvc     hello-wsnvc   True
# kubectl get svc istio-ingressgateway --namespace istio-system --output jsonpath="{.status.loadBalancer.ingress[*]['ip']}"
47.95.191.136
curl -H "Host: hello.helloworld.example.com" http://47.95.191.136/
Hello World!!

```

### 配置自定义主域名
> kubectl edit cm config-domain --namespace knative-serving ，如下图所示，添加 serverless.kuberun.com 到 ConfigMap 中，然后保存退出就完成了自定义主域名的配置。  
￼
```
# kubectl -n helloworld get ksvc
NAME    URL                                              LATESTCREATED   LATESTREADY   READY   REASON
hello   http://hello.helloworld.serverless.kuberun.com   hello-wsnvc     hello-wsnvc   True
```

### 泛域名解析
> *.serverless.kuberun.com  解析到 Istio Gateway 47.95.191.136 上面去。  

### 自定义服务域名
- 先在万网上面修改域名解析，把 hello.kuberun.com  的 A 记录指向  Istio Gateway 47.95.191.136；
- hello.kuberun.com 解析到 Istio Gateway 以后 Istio Gateway 并不知道此时应该转发到哪个服务，所以还需要配置 VirtualService 告知 Istio 如何转发。
```
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
 name: hello-ingress-route
 namespace: knative-serving
spec:
 gateways:
 - knative-ingress-gateway
 hosts:
 - hello.helloworld.serverless.kuberun.com
 - hello.kuberun.com
 http:
 - match:
   - uri:
       prefix: "/"
   rewrite:
     authority: hello.helloworld.svc.cluster.local
   retries:
     attempts: 3
     perTryTimeout: 10m0s
   route:
   - destination:
       host: istio-ingressgateway.istio-system.svc.cluster.local
       port:
         number: 80
     weight: 100
   timeout: 10m0s
   websocketUpgrade: true
```

### 基于路径的服务转发
```
---
apiVersion: v1
kind: Namespace
metadata:
  name: blog

---
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: hello-blog
  namespace: blog
spec:
  template:
    metadata:
      labels:
        app: hello
      annotations:
        autoscaling.knative.dev/target: "10"
    spec:
      containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/simple-app:132e07c14c49
          env:
            - name: TARGET
              value: "Blog!"
---
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: hello-ingress-route
  namespace: knative-serving
spec:
  gateways:
  - knative-ingress-gateway
  hosts:
  - hello.helloworld.serverless.kuberun.com
  - hello.kuberun.com
  http:
  - match:
    - uri:
        prefix: "/blog"
    rewrite:
      authority: hello-blog.blog.svc.cluster.local
    retries:
      attempts: 3
      perTryTimeout: 10m0s
    route:
    - destination:
        host: istio-ingressgateway.istio-system.svc.cluster.local
        port:
          number: 80
      weight: 100
  - match:
    - uri:
        prefix: "/"
    rewrite:
      authority: hello.helloworld.svc.cluster.local
    retries:
      attempts: 3
      perTryTimeout: 10m0s
    route:
    - destination:
        host: istio-ingressgateway.istio-system.svc.cluster.local
        port:
          number: 80
      weight: 100
    timeout: 10m0s
    websocketUpgrade: true

```

### 路由运作原理
当主机外部请求example.com或你自己的域名到达 knative-ingress-gateway网关时，entry-routeVirtualService会检查是否有/search或/loginURI。如果URI匹配，则请求主机将相应地重写到Search服务主机或Login服务中。这会重置请求的最终目的地。更新主机的请求将knative-ingress-gateway再次转发给Gateway。网关代理检查更新的主机，并根据其主机设置将其转发Search或 Login服务。
￼
### 流量灰度和版本管理
> 部署一个service  
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
 name: stock-service-example
 namespace: default
spec:
 template:
   metadata:
     name: stock-service-example-v1
   spec:
     containers:
     - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
       env:
         - name: RESOURCE
           value: v1
       readinessProbe:
         httpGet:
           path: /

```

- Knative Service
> kubectl get ksvc helloworld-go --output yaml  
- Knative Configuration
> kubectl get configuration -l "serving.knative.dev/service=helloworld-go" --output yaml  
- Knative Revision
> kubectl get revision -l "serving.knative.dev/service=helloworld-go" --output yaml  
- Knative Route
> kubectl get route -l "serving.knative.dev/service=helloworld-go" --output yaml  

### 50%流量到V2
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: stock-service-example
  namespace: default
spec:
  template:
    metadata:
      name: stock-service-example-v2
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
        env:
          - name: RESOURCE
            value: v2
        readinessProbe:
          httpGet:
            path: /
  traffic:    #增加了traffic配置
  - tag: v1
    revisionName: stock-service-example-v1
    percent: 50
  - tag: v2
    revisionName: stock-service-example-v2
    percent: 50
  - tag: latest
    latestRevision: true
    percent: 0

```


### 提前验证 Revision
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: stock-service-example
  namespace: default
spec:
  template:
    metadata:
      name: stock-service-example-v3
    spec:
      containers:
      - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
        env:
          - name: RESOURCE
            value: v3
        readinessProbe:
          httpGet:
            path: /
  traffic:
  - tag: v1
    revisionName: stock-service-example-v1
    percent: 50
  - tag: v2
    revisionName: stock-service-example-v2
    percent: 50
  - tag: latest
    latestRevision: true
    percent: 0

```
可以看到 v3 Revision 虽然创建出来了，但是因为没有设置 traffic，所以并不会有流量转发。此时你执行多少次 ./run-test.sh 都不会得到 v3 的输出。因为traffic没设置…


> Serving到这结束了,下次继续eventing  
