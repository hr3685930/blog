# Knative-eventing

> Eventing 主要由事件源（Event Source）、事件处理（Flow）以及事件消费者（Event Consumer）三部分构成。  

## 事件源（Event Source）
当前支持以下几种类型的事件源：
- ApiserverSource：每次创建或更新 Kubernetes 资源时，ApiserverSource 都会触发一个新事件
- GitHubSource：GitHub 操作时，GitHubSource 会触发一个新事件
- GcpPubSubSource： GCP 云平台 Pub/Sub 服务会触发一个新事件
- AwsSqsSource：Aws 云平台 SQS 服务会触发一个新事件
- ContainerSource：ContainerSource 将实例化一个容器，通过该容器产生事件
- CronJobSource：通过 CronJob 产生事件
- KafkaSource：接收 Kafka 事件并触发一个新事件
- CamelSource：接收 Camel 相关组件事件并触发一个新事件

## 事件接收/转发（Flow）
当前 Knative 支持如下事件接收处理：

### 直接事件接收
> 通过事件源直接转发到单一事件消费者。支持直接调用 Knative Service 或者 Kubernetes Service 进行消费处理。这样的场景下，如果调用的服务不可用，事件源负责重试机制处理通过事件通道（Channel）以及事件订阅（Subscriptions）转发事件处理这样的情况下，可以通过 Channel 保证事件不丢失并进行缓冲处理，通过 Subscriptions 订阅事件以满足多个消费端处理  

### 通过 brokers 和 triggers 支持事件消费及过滤机制
> 从 v0.5 开始，Knative Eventing 定义 Broker 和 Trigger 对象，实现了对事件进行过滤（亦如通过 ingress 和 ingress controller 对网络流量的过滤一样）通过定义 Broker 创建 Channel，通过 Trigger 创建 Channel 的订阅（subscription），并产生事件过滤规则。  

### 事件消费者（Event Consumer）
> 为了满足将事件发送到不同类型的服务进行消费， Knative Eventing 通过多个 Kubernetes 资源定义了两个通用的接口：  
- Addressable 接口提供可用于事件接收和发送的 HTTP 请求地址，并通过status.address.hostname字段定义。作为一种特殊情况，Kubernetes Service 对象也可以实现 Addressable 接口
- Callable 接口接收通过 HTTP 传递的事件并转换事件。可以按照处理来自外部事件源事件的相同方式，对这些返回的事件做进一步处理

当前 Knative 支持通过 Knative Service 或者 Kubernetes Service 进行消费事件。

另外针对事件消费者，如何事先知道哪些事件可以被消费？ Knative Eventing 在最新的 0.6 版本中提供 Registry 事件注册机制, 这样事件消费者就可以事先通过 Registry 获得哪些 Broker 中的事件类型可以被消费。


### 创建和配置Eventing名称空间

> kubectl create namespace default  
> kubectl label namespace default knative-eventing-injection=enabled  

### 验证Broker正在运行
> kubectl get broker  
当Broker具有READY=True状态时，它可以开始管理它接收的任何事件。
```
NAME      READY   REASON   HOSTNAME    AGE
default   True    default-Broker.event-example.svc.cluster.local   1m
```

kubectl get triggers
### 创建时间使用者booker (hello-display和goodbye-display)
```
kubectl --namespace event-example apply --filename - << END
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-display
spec:
  replicas: 1
  selector:
    matchLabels: &labels
      app: hello-display
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: event-display
          # Source code: https://github.com/knative/eventing-contrib/blob/release-0.6/cmd/event_display/main.go
          image: gcr.io/knative-releases/github.com/knative/eventing-sources/cmd/event_display@sha256:37ace92b63fc516ad4c8331b6b3b2d84e4ab2d8ba898e387c0b6f68f0e3081c4

 ---

# Service pointing at the previous Deployment. This will be the target for event
# consumption.
  kind: Service
  apiVersion: v1
  metadata:
    name: hello-display
  spec:
    selector:
      app: hello-display
    ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
—
apiVersion: apps/v1
kind: Deployment
metadata:
  name: goodbye-display
spec:
  replicas: 1
  selector:
    matchLabels: &labels
      app: goodbye-display
  template:
    metadata:
      labels: *labels
    spec:
      containers:
        - name: event-display
          # Source code: https://github.com/knative/eventing-contrib/blob/release-0.6/cmd/event_display/main.go
          image: gcr.io/knative-releases/github.com/knative/eventing-sources/cmd/event_display@sha256:37ace92b63fc516ad4c8331b6b3b2d84e4ab2d8ba898e387c0b6f68f0e3081c4

---

# Service pointing at the previous Deployment. This will be the target for event
# consumption.
kind: Service
apiVersion: v1
metadata:
  name: goodbye-display
spec:
  selector:
    app: goodbye-display
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
END

```


### 创建一个触发器
```
kubectl --namespace event-example apply --filename - << END
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: hello-display
spec:
  filter:
    attributes:
      type: greeting
  subscriber:
    ref:
     apiVersion: v1
     kind: Service
     name: hello-display
END

#该命令创建一个触发器，将所有类型greeting的事件发送到您指定的事件使用者hello-display。
```
### 创建二个触发器
```
kubectl --namespace event-example apply --filename - << END
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: goodbye-display
spec:
  filter:
    attributes:
      source: sendoff
  subscriber:
    ref:
     apiVersion: v1
     kind: Service
     name: goodbye-display
END
该命令创建一个触发器，将所有源事件发送sendoff到名为的事件使用者goodbye-display。
```

### 创建一个测试Pod
```

kubectl --namespace event-example apply --filename - << END
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: curl
  name: curl
spec:
  containers:
    # This could be any image that we can SSH into and has curl.
  - image: radial/busyboxplus:curl
    imagePullPolicy: IfNotPresent
    name: curl
    resources: {}
    stdin: true
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    tty: true
END

```

> kubectl --namespace event-example attach curl -it  

```
curl -v "default-broker.event-example.svc.cluster.local" \
  -X POST \
  -H "Ce-Id: say-hello" \    #触发器名
  -H "Ce-Specversion: 0.2" \  
  -H "Ce-Type: greeting" \    #类型
  -H "Ce-Source: not-sendoff" \
  -H "Content-Type: application/json" \
  -d '{"msg":"Hello Knative!"}'
```
### 查看时间日志
> kubectl logs -l app=hello-display --tail=100  

### Broker创建
```
cat << EOF | kubectl apply -f -
apiVersion: eventing.knative.dev/v1alpha1
kind: Broker
metadata:
  namespace: default
  name: default
EOF

```

### 创建Trigger时默认为default
```
apiVersion: eventing.knative.dev/v1alpha1
kind: Trigger
metadata:
  name: my-service-trigger
  namespace: default
spec:
  broker: default # Defaulted by the Webhook.
  filter:
    attributes:
      type: dev.knative.foo.bar
  subscriber:
    ref:
      apiVersion: serving.knative.dev/v1alpha1
      kind: Service
      name: my-service

```

### event register
```
apiVersion: eventing.knative.dev/v1alpha1
kind: EventType
metadata:
  name: com.github.pullrequest
  namespace: event-example
spec:
  type: com.github.pull_request
  source: github.com
  schema: //github.com/schemas/pull_request
  description: "GitHub pull request"
  broker: default
```

> kubectl get eventtypes -n event-example  