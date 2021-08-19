# knative-serving(流量灰度和版本管理)
> 本篇主要介绍KnativeServing的流量灰度，通过一个rest-api的例子演示如何创建多个Revision、并在不同的Revision之间按照流量比例灰度。

## 部署rest-api.v1
```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
    name: traffic-example
    namespace: default
spec:
    template:
        metadata:
            name: traffic-example-v1
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
> 首次安装会创建出一个叫做traffic-example-v1的Revision，并且是把100%的流量都打到traffic-example-v1上。   
验证Serving的各个资源 
如下图所示，我们先回顾一下Serving涉及到的各种资源。接下来我们分别看 一下刚才部署的revision-v1yaml各个资源配置。 

![](knative-serving%E6%B5%81%E9%87%8F%E7%81%B0%E5%BA%A6%E5%92%8C%E7%89%88%E6%9C%AC%E7%AE%A1%E7%90%86/9BB8798B-50BD-41EF-8B1C-FF0929C7F484.png)


灰度20%的流量到v2
```
apiVersion: serving.knative.dev/v1
kind: Service
metadata:
    name: traffic-example
    namespace: default
spec:
    template:
        metadata:
             name: traffic-example-v2
        spec:
        containers:
        - image: registry.cn-hangzhou.aliyuncs.com/knative-sample/rest-api-go:v1
            env:
            - name: RESOURCE
              value: v2
        readinessProbe:
        httpGet:
            path: /
            traffic:
            - tag: v1
        revisionName: traffic-example-v1
        percent: 80
        - tag: v2
        revisionName: traffic-example-v2
            percent: 20
        -  tag: latest
            latestRevision: true
            percent: 0
```

## 总结
KnativeService的灰度、回滚都是基于流量的。Workload(Pod)是根据过来的流量自动创建出来的。所以在KnativeServing模型中流量是核心驱动。这和传统的应用发布、灰度模型是有区别的。假设有一个应用app1，传统的做法首先是设置应用的实例个数 (Kubernetes 体系中就是Pod)，我们假设实例个数是10个。如果要进行灰度发布，那么传统 的做法就是先发布一个Pod，此时v1和v2的分布方式是:v1的Pod9 个，v2的Pod1个。如果要继续扩大灰度范围的话那就是v2的Pod数量变多，v1的Pod数 量变少，但总的Pod数量维持10个不变。在KnativeServing模型中Pod数量永远都是根据流量自适应的，不需要提前指定。在灰度的时候只需要指定流量在不同版本之间的灰度比例即可。每一个Revision的实例数都是根据流量的大小自适应，不需要提前指定。从上面的对比中可以发现KnativeServing模型是可以精准的控制灰度影响的范 围的，保证只灰度一部分流量。而传统的模型中Pod灰度的比例并不能真实的代表流量的比例，是一个间接的灰度方法。

