# istio-日志采集
## 安装之前
> 按照安装指南中的说明安装 Istio。  
## Envoy 访问日志
```
helm template install/kubernetes/helm/istio --namespace=istio-system -x templates/configmap.yaml --set global.proxy.accessLogFile="/dev/stdout" | kubectl replace -f -
```

## 使用Fluentd 记录日志
```
# Logging Namespace。下列内容都在此 namespace 中.
apiVersion: v1
kind: Namespace
metadata:
  name: logging
---
# Elasticsearch Service
apiVersion: v1
kind: Service
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
spec:
  ports:
    - port: 9200
      protocol: TCP
      targetPort: 9200
  selector:
    app: elasticsearch
---
# Elasticsearch Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: elasticsearch
  namespace: logging
  labels:
    app: elasticsearch
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  template:
    metadata:
      labels:
        app: elasticsearch
    spec:
      containers:
        - image: docker.elastic.co/elasticsearch/elasticsearch-oss:6.1.1
          name: elasticsearch
          resources:
            # 在初始化时需要更多的 cpu，因此使用 burstable 级别。
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
          env:
            - name: discovery.type
              value: single-node
          ports:
            - containerPort: 9200
              name: db
              protocol: TCP
            - containerPort: 9300
              name: transport
              protocol: TCP
          volumeMounts:
            - name: elasticsearch
              mountPath: /data
      volumes:
        - name: elasticsearch
          emptyDir: {}
---
# Fluentd Service
apiVersion: v1
kind: Service
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    app: fluentd-es
spec:
  ports:
    - name: fluentd-tcp
      port: 24224
      protocol: TCP
      targetPort: 24224
    - name: fluentd-udp
      port: 24224
      protocol: UDP
      targetPort: 24224
  selector:
    app: fluentd-es
---
# Fluentd Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: fluentd-es
  namespace: logging
  labels:
    app: fluentd-es
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  template:
    metadata:
      labels:
        app: fluentd-es
    spec:
      containers:
        - name: fluentd-es
          image: ist0ne/fluentd-elasticsearch:v2.0.1
          env:
            - name: FLUENTD_ARGS
              value: --no-supervisor -q
          resources:
            limits:
              memory: 500Mi
            requests:
              cpu: 100m
              memory: 200Mi
          volumeMounts:
            - name: config-volume
              mountPath: /etc/fluent/config.d
      terminationGracePeriodSeconds: 30
      volumes:
        - name: config-volume
          configMap:
            name: fluentd-es-config
---
# Fluentd ConfigMap，包含配置文件。
kind: ConfigMap
apiVersion: v1
data:
  forward.input.conf: |-
    # 采用 TCP 发送的消息
    <source>
      type forward
    </source>
  output.conf: |-
    <match **>
       type elasticsearch
       log_level info
       include_tag_key true
       host elasticsearch
       port 9200
       logstash_format true
       # 设置 chunk limits.
       buffer_chunk_limit 2M
       buffer_queue_limit 8
       flush_interval 5s
       # 重试间隔绝对不要超过 5 分钟。
       max_retry_wait 30
       # 禁用重试次数限制（永远重试）。
       disable_retry_limit
       # 使用多线程。
       num_threads 2
    </match>
metadata:
  name: fluentd-es-config
  namespace: logging
---
# Kibana Service
apiVersion: v1
kind: Service
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
spec:
  ports:
    - port: 5601
      protocol: TCP
      targetPort: ui
  selector:
    app: kibana
---
# Kibana Deployment
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: kibana
  namespace: logging
  labels:
    app: kibana
  annotations:
    sidecar.istio.io/inject: "false"
spec:
  template:
    metadata:
      labels:
        app: kibana
    spec:
      containers:
        - name: kibana
          image: docker.elastic.co/kibana/kibana-oss:6.1.1
          resources:
            # need more cpu upon initialization, therefore burstable class
            limits:
              cpu: 1000m
            requests:
              cpu: 100m
          env:
            - name: ELASTICSEARCH_URL
              value: http://elasticsearch:9200
          ports:
            - containerPort: 5601
              name: ui
              protocol: TCP
---
```

## istio-mixer(logentry,fluentd,rule)
```
# logentry 实例配置
apiVersion: "config.istio.io/v1alpha2"
kind: logentry
metadata:
  name: newlog
  namespace: istio-system
spec:
  severity: '"info"'
  timestamp: request.time
  variables:
    source: source.labels["app"] | source.workload.name | "unknown"
    user: source.user | "unknown"
    destination: destination.labels["app"] | destination.workload.name | "unknown"
    responseCode: response.code | 0
    responseSize: response.size | 0
    latency: response.duration | "0ms"
  monitored_resource_type: '"UNSPECIFIED"'
---
# Fluentd handler 配置
apiVersion: "config.istio.io/v1alpha2"
kind: fluentd
metadata:
  name: handler
  namespace: istio-system
spec:
  address: "fluentd-es.logging:24224"
---
# 将 logentry 示例发送到 Fluentd handler 的 rule
apiVersion: "config.istio.io/v1alpha2"
kind: rule
metadata:
  name: newlogtofluentd
  namespace: istio-system
spec:
  match: "true" # 匹配所有请求
  actions:
    - handler: handler.fluentd
      instances:
        - newlog.logentry
---

```

> 启动后记得配置网关  