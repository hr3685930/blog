# Knative Eventing原理
## 概念理解
> 由于整个knative都是建立在k8s之上的，knative的实现完全基于k8s原生的编程框架。所以，要理解eventing的整个原理，我们先来分别了解其定义的CRD资源，以及它们之间的联动关系。  


## CRD资源
> 通过在k8s上过滤eventing的crd资源，可以看到knative提供了一下的资源定义。他们都有什么作用，以及彼此之间存在何种依赖关系？接下来将做分析。  
```
kc get crd | grep eventing
apiserversources.sources.eventing.knative.dev        2019-07-06T14:34:04Z
brokers.eventing.knative.dev                         2019-07-06T14:34:04Z
channels.eventing.knative.dev                        2019-07-06T14:34:04Z
clusterchannelprovisioners.eventing.knative.dev      2019-07-06T14:34:04Z
containersources.sources.eventing.knative.dev        2019-07-06T14:34:04Z
cronjobsources.sources.eventing.knative.dev          2019-07-06T14:34:04Z
eventtypes.eventing.knative.dev                      2019-07-06T14:34:04Z
subscriptions.eventing.knative.dev                   2019-07-06T14:34:04Z
triggers.eventing.knative.dev                        2019-07-06T14:34:04Z

```


可以按照功能，将其分为两大类：
* source相关
**抽象事件类型，适用于每一种source**
eventtypes.eventing.knative.dev

**系统支持的三种source**
apiserversources.sources.eventing.knative.dev
containersources.sources.eventing.knative.dev
cronjobsources.sources.eventing.knative.dev

* broker-trigger相关
**顶层抽象**
brokers.eventing.knative.dev
triggers.eventing.knative.dev
**逻辑层**
channels.eventing.knative.dev
subscriptions.eventing.knative.dev
**物理实现**
clusterchannelprovisioners.eventing.knative.dev



## 关联关系
> 上面列出了所有的CRD，他们的作用和关联关系如下：  
1. 

![](Knative%20Eventing%E5%8E%9F%E7%90%86/knative-eventing-crds.png)
> 三个source CRD创建之后，其controller会主动部署对应的deployment; deployment实现了从producer收集event，并转发到下一跳的逻辑，相当于event进入knative-eventing系统的入口；  
2.  当创建namespace的时候，如果指定了label
knative-eventing-injection=enabled### ，knative会在该namespace自动创建default broker；
3. broker和trigger抽象了event转发的逻辑，为了实现broker，controller会分别创建该broker的ingress-channel和trigger-channel以及用于租户业务namespace与eventing system namespace之间event转发的deployment和service；
4. trigger是依赖于broker的，如果trigger不指定broker，会自动使用default broker，trigger中明确定义了subscription；
5. subscription在这里涉及到两部分，一部分是用户创建trigger时指定的业务相关的订阅和SINK；另一部分是系统内置的，trigger channel在fanout了消息之后，需要reply通道，ingress-channel和ingress-subscription就是为了实现该通道的转发逻辑；
6. clusterchannelprovisioners在当前版本中还保留，但是后续会被废弃掉，转而采用各个provisioner对应的CRD。这里可以理解为channel之下对应的物理实现，通过解析channel中的subscription信息来fanout事件。

## broker-trigger实例
> 按照官网的步骤来部署一个  
ApiServerSource### 类型的source采集k8s的event，并经过一些列的channel之后，最终达到ksvc来展示出来。 具体步骤详见这里 。忽略一些非关键的步骤，我们来重点看看几个核心概念都是如何定义的，以及他们之间的关联关系。
[具体步骤详见这里](https://knative.dev/docs/eventing/samples/kubernetes-event-source/)

## consumer
先准备好最终展示所接收event的ksvc，该资源受autoscale的控制，在没有请求的情况下，pod实例数会自动缩容到0值。查看yaml配置文件的最后一段是knative service containers的定义，这里的镜像的功能是显示接收到的event信息 (为了墙内拉取方便，镜像已经被替换地址)。
```
apiVersion: serving.knative.dev/v1alpha1
kind: Service
metadata:
  name: event-display
  namespace: default
spec:
  template:
    spec:
      containers:
        - image: ljchen/knative_eventing-sources_cmd_event_display:v0.7.0
```
部署好之后，可以看到对应的pod和service信息如下：
```
kc get deploy,svc,pod
NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/event-display-xnztn-deployment     1/1     1            1           15h

NAME                                       TYPE           CLUSTER-IP       EXTERNAL-IP                                               PORT(S)             AGE
service/event-display                      ExternalName   <none>           istio-ingressgateway.istio-system.svc.cluster.local       <none>              15h
service/event-display-xnztn                ClusterIP      10.111.128.248   <none>                                                    80/TCP              15h
service/event-display-xnztn-m5j2x          ClusterIP      10.110.215.179   <none>                                                    80/TCP              15h
service/event-display-xnztn-xqtmr          ClusterIP      10.105.98.248    <none>                                                    9090/TCP,9091/TCP   15h

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/event-display-xnztn-deployment-797c9bbcd8-hd8l4    2/2     Running   0          96s
```
## source
首先是event的来源，这里由于是ApiServerSource，只需直接指定对应的配置文件。
```
apiVersion: sources.eventing.knative.dev/v1alpha1
kind: ApiServerSource
metadata:
  name: testevents
  namespace: default
spec:
  serviceAccountName: events-sa
  mode: Resource
  resources:
    - apiVersion: v1
      kind: Event
  sink:
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Broker
    name: default
```
注意，这里配置的sink 指定了使用default broker。当该yaml被应用到k8s之后，在对应的namespace下可以看到创建了一个deployment。

## apiserversource
```
kc get apiserversource
NAME         AGE
testevents   4h27m
```

## deployment, pod
```
kc get deploy,pod
NAME                                                     READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/apiserversource-testevents-57knn   1/1     1            1           4h26m                                           9090/TCP            37d

NAME                                                   READY   STATUS    RESTARTS   AGE
pod/apiserversource-testevents-57knn-95dfb87bd-s48p4   1/1     Running   0          4h26m
```
如果查看service，会发现找不到该deploy对应的service。原因是其只watch k8s api-server，然后将k8s event信息收集到之后发送到对应的channel；因此，该服务并不对外提供其他服务。另外，该服务貌似并没有sidecar，我们知道对于knative的service都是有sidecar的，为啥呢？还是同样的原因，这是一个系统服务，不是业务的ksvc。

## broker
default broker是自动创建的，创建的时候需要为namespace配置对应的label（如果不配置的话，会导致broker无法被自动创建，整个event链路不通），具体如下：
```
kubectl label namespace default knative-eventing-injection=enabled
```
在执行完命令之后，最直观的是可以查看到已经创建的broker default （通过查看该broker的status可以看到address，即外部的访问地址）；在default namespace下可以看到，已经自动部署了两个deployment。但是如果我们通过查看knative的eventing CRD可以发现更多的资源被创建了出来。


## broker详情
```
kc get broker default -o yaml
apiVersion: eventing.knative.dev/v1alpha1
kind: Broker
metadata:
  ......
status:
  IngressChannel: # ingress channel
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: default-kn-ingress
    namespace: default
  address:  # 对外访问的地址信息(并不是指向channel，而是deployment)
    hostname: default-broker.default.svc.cluster.local
    url: http://default-broker.default.svc.cluster.local
  ......
  triggerChannel: # trigger channel
    apiVersion: eventing.knative.dev/v1alpha1
    kind: Channel
    name: default-kn-trigger
    namespace: default
```
## channels
```
 kc get channels
NAME                 READY   REASON   AGE
default-kn-ingress   True             10h # 用于reply的channel
default-kn-trigger   True             10h # 主channel
```

## subscriptions 
（属于ingress侧的订阅, 将ingressChannel事件订阅到default-broker-ingress上，用于reply）
```
kc get subscriptions.eventing.knative.dev
NAME                             READY   REASON   AGE
internal-ingress-default-fv8bg   True             10h
```

## deployment, service and pod （将流量引入in-memory channel的业务端组件）
```
kc get deploy,svc,pod
NAME                                                   READY   UP-TO-DATE   AVAILABLE   AGE
deployment.extensions/default-broker-filter            1/1     1            1           96s
deployment.extensions/default-broker-ingress           1/1     1            1           96s

NAME                                       TYPE           CLUSTER-IP       EXTERNAL-IP                                               PORT(S)             AGE
```

## 业务端组件ingress和filter的service

```
service/default-broker                     ClusterIP      10.101.17.164    <none>                                                    80/TCP,9090/TCP     10h
service/default-broker-filter              ClusterIP      10.97.228.135    <none>                                                    80/TCP,9090/TCP     10h
```

## 真正channel的service（都映射到dispatcher上）
```
service/default-kn-ingress-channel-4xxz2   ExternalName   <none>           in-memory-dispatcher.knative-eventing.svc.cluster.local   <none>              10h
service/default-kn-trigger-channel-zbrlw   ExternalName   <none>           in-memory-dispatcher.knative-eventing.svc.cluster.local   <none>              10h

NAME                                         READY   STATUS    RESTARTS   AGE
pod/default-broker-filter-744ff96759-x4nbt   1/1     Running   0          97s
pod/default-broker-ingress-96cd4b769-qswm2   1/1     Running   0          97s
```

## trigger
```
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: testevents-trigger
  namespace: default
spec: # 未指定broker，默认使用default broker
  subscriber: # 指定trigger侧订阅到consumer
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: event-display
```
在subscriber.ref可以看到，其将event内容发送到了event-display这一个knative service。
```
 kc get trigger
NAME                 READY   REASON   BROKER    SUBSCRIBER_URI                                   AGE
testevents-trigger   True             default   http://event-display.default.svc.cluster.local   12s
```
subscriptions 多出来了一条记录 default-testevents-trigger-rrhzn
（属于trigger侧的订阅, 固定将triggerChannel事件订阅到default-broker-filter上，event订阅的URI中会带上详细trigger信息，用于反查trigger中用户指定的subscription内容）
```
kc get subscriptions
NAME                               READY   REASON   AGE
default-testevents-trigger-rrhzn   True             7m31s
#internal-ingress-default-fv8bg     True             10h
```

## 数据平面
前面是整个创建流程，以及对应的资源属性，接下来分析一下event转发的数据面流程，先来看官方的一张控制面与转发面的图。
由于该图较老，只体现了channel与scription这一层的概念，且数据面较抽象。我重新基于broker&trigger以及in-memory-channel整理了该图，具体如下。
其中实现部分为真正event的转发流程，broker-ingress、broker-filter与上面的channel之间通过虚线连接的部分是逻辑链路。黑色的连线为event转发，红色为反向的reply链路。in-memory-dispatcher是imc的底层实现，如果是使用kafka就应该替换为kafaka deployment。这里先不做详细描述，具体见下文逐步分析。

![](Knative%20Eventing%E5%8E%9F%E7%90%86/knative-eventing-cd-plane.png)

![](Knative%20Eventing%E5%8E%9F%E7%90%86/knative-eventing-imc-dataplane.png)

## source源
前面讲到，当我们将apiserversource配置下发后，controller会在namespace下部署出一个deployment，该deployment的作用是收集k8s的evnet并发送到指定的SINK。由于我们指定的是default broker，经controller处理之后，其参数变为了default-broker这个service的访问地址（即，broker的总入口）；具体见下面操作中对应的注释信息。
```
 kc get deployment.extensions/apiserversource-testevents-57knn -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  ……
spec:
  ……
  template:
    ……
    spec:
      containers:
      - env:
        - name: SINK_URI  # 重点关注该value，为default-broker这个service
          value: http://default-broker.default.svc.cluster.local
        - name: MODE
          value: Resource
        - name: API_VERSION
          value: v1
        - name: KIND
          value: Event
        - name: CONTROLLER
          value: "false"
        - name: SYSTEM_NAMESPACE
          valueFrom:
            fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
        image: ljchen/knative_eventing_cmd_apiserver_receive_adapter:v0.7.0
        ......
status:
  ......
```
当event接收到后，直接转发到SINK_URL指定的default-broker地址。自此，source部分的工作已经完结，接下来在看看default-broker。

## broker & triger
broker在逻辑层面包含ingress-channel和trigger-channel，对应在数据面位于eventing的system namespace下有对应的dispatcher。同时，位于业务的namespace中会有broker-ingress和broker-filter两个deployment用来负责dispatcher与业务层之间event的转发。
## broker-ingress
既然已经知道流量是转发给default-broker这个service的，直接查看该service以及对应的endpoint的地址，确定其对应pod和deployment。

## service, endpoint
```
kc get svc,ep default-broker
NAME                     TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
service/default-broker   ClusterIP   10.101.17.164   <none>        80/TCP,9090/TCP   15h

NAME                       ENDPOINTS                           AGE
endpoints/default-broker   10.244.0.69:9090,10.244.0.69:8080   15h
```
根据endpoint的IP地址，查找对应pod，deployment。

## pod
```
kc get pod -o wide | grep 10.244.0.69
default-broker-ingress-96cd4b769-qswm2             1/1     Running   0          4h46m   10.244.0.69    k8s-master   <none>           <none>
```
## deployment
```
kc get deploy default-broker-ingress
NAME                     READY   UP-TO-DATE   AVAILABLE   AGE
default-broker-ingress   1/1     1            1           4h48m
```
该deployment的配置信息如下，这里面关键信息已经在注释中标明。
```
kc get deploy default-broker-ingress -o yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  ......
spec:
  ......
  selector:
    matchLabels:
      eventing.knative.dev/broker: default  # broker的名称为default
      eventing.knative.dev/brokerRole: ingress  # 这里指定了broker的角色为ingress，难道还有其他的角色？是的，就是trigger role
  ......
  template:
    ......
    spec:
      containers:
      - env:
        ......
        - name: FILTER
        - name: CHANNEL  # 该deployment接受到event之后，发送到下一个channel的名称 
          value: default-kn-trigger-channel-zbrlw.default.svc.cluster.local
        - name: BROKER   # 该deploy所属的broker
          value: default  
        image: ljchen/knative_eventing_cmd_broker_ingress:v0.7.0  # 镜像已经替换
        ......
status:
  ......
```
下一跳channel的service表明发送到knative-eventing ns，ingress-channel和trigger-channel都发到同一个external-ip，即imc的底层in-memory-dispatcher
```
kc get svc
NAME                               TYPE           CLUSTER-IP       EXTERNAL-IP                                               PORT(S)             AGE
#default-kn-ingress-channel-4xxz2   ExternalName   <none>           in-memory-dispatcher.knative-eventing.svc.cluster.local   <none>              16h
default-kn-trigger-channel-zbrlw   ExternalName   <none>           in-memory-dispatcher.knative-eventing.svc.cluster.local   <none>              16h
```
knative-eventing system namespace中查看服务
```
ke get svc
NAME                   TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)    AGE
in-memory-dispatcher   ClusterIP   10.98.22.109    <none>        80/TCP     17h
```
## endpoint
```
ke get ep in-memory-dispatcher
NAME                   ENDPOINTS          AGE
in-memory-dispatcher   10.244.0.43:8080   17h
```
## pod
```
ke get pod -o wide | grep 10.244.0.43
in-memory-channel-dispatcher-ffd969cd9-vhblf   1/1     Running   0          17h   10.244.0.43   k8s-master   <none>           <none>
```

## deployment
```
ke get deploy
NAME                           READY   UP-TO-DATE   AVAILABLE   AGE
in-memory-channel-dispatcher   1/1     1            1           17h
```
## dispatcher
自此，我们似乎再也没有线索，不知道event到了in-memory-dispatcher之后是如何转发的。但是通过查看dispatcher代码了解到，该deployment转发event的逻辑是基于channel的subscription配置信息。由于当前对应为triggerChannel，通过查询其channel
default-kn-trigger可知：channel description “default-kn-trigger”
```
kc describe channels default-kn-trigger
Name:         default-kn-trigger
Kind:         Channel
….
Spec:
  Provisioner:
    API Version:  eventing.knative.dev/v1alpha1
    Kind:         ClusterChannelProvisioner
    Name:         in-memory
  Subscribable:
    Subscribers:
      Generation:  1
      Ref:
        Name:          default-testevents-trigger-rrhzn
        Namespace:     default
        UID:           9985ef4e-a05a-11e9-84ef-525400ff729a
      Reply URI:       http://default-kn-ingress-channel-4xxz2.default.svc.cluster.local
      Subscriber URI:  http://default-broker-filter.default.svc.cluster.local/triggers/default/testevents-trigger/99849cd0-a05a-11e9-84ef-525400ff729a   # 找到其fanout的下一跳URL了。注意，这个URI除了host之外，还有一串信息
      UID:             9985ef4e-a05a-11e9-84ef-525400ff729a
Status:
  Address: # 对外通过该地址来接收发往该channel的event
    Hostname:  default-kn-trigger-channel-zbrlw.default.svc.cluster.local
    URL:       http://default-kn-trigger-channel-zbrlw.default.svc.cluster.local
  ….
```
报文会被dispacher fanout到subscriber URI，也就是default-broker-filter这个服务。

## broker-filter
```
kc get svc default-broker-filter
NAME                    TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)           AGE
default-broker-filter   ClusterIP   10.97.228.135   <none>        80/TCP,9090/TCP   17h

kc get ep default-broker-filter
NAME                    ENDPOINTS                           AGE
default-broker-filter   10.244.0.68:9090,10.244.0.68:8080   17h

kc get pod -o wide | grep 10.244.0.68
default-broker-filter-744ff96759-x4nbt             1/1     Running   0          7h      10.244.0.68    k8s-master   <none>           <none>

kc get deploy default-broker-filter
NAME                               READY   UP-TO-DATE   AVAILABLE   AGE
default-broker-filter              1/1     1            1           7h1m

```
到这里似乎又卡壳了，因为通过查看default-broker-filter的配置，得不到任何有关其下一跳的信息。通过分析代码，发现代码中通过报文URI中的路径信息（http://default-broker-filter.default.svc.cluster.local/triggers/default/testevents-trigger/99849cd0-a05a-11e9-84ef-525400ff729a### ）来获取到trigger名称（default/testevents-trigger），然后再提取trigger的subscriber来定位到下一跳，最终将event发送给knative-service即event-display。
```
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: testevents-trigger
  namespace: default
spec:
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: event-display

```
