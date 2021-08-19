# istio-简介
> Istio 是一个由 IBM、Google 以及 Lyft 联合推出的开源软件，以无痛方式为运行在 Kubernetes 上的微服务提供流量管理，访问策略管理以及监控等功能。这一软件目前仅在 Kubernetes 上运行，今后可能会扩展到其他平台。  


## Istio核心组件及其功能
Istio总体被分为两部分:
1. 数据面
> 数据面被称为Sidecar,通过注入方式和业务容器共存于同一个Pod里,会劫持业务容器的流量,并接受控制面组件的控制,同时会向控制面输出日志,跟踪及监控数据  
2. 控制面
> 控制面试Istio的核心,管理Istio的所有功能  

### Pilot
> Pilot 是用户和 Isito 之间的桥梁，负责接收各种配置，并发送给各个组件。  
#### 架构图
![](istio-%E7%AE%80%E4%BB%8B/71840022-3C8B-4E26-BE30-45C39E2D9D72.png)
上面是 [官方关于pilot的架构图](https://github.com/istio/old_pilot_repo/blob/master/doc/design.md) ，因为是old_pilot_repo目录下，可能与最新架构有出入，仅供参考。所谓的pilot包含两个组件：pilot-agent和pilot-discovery。图里的agent对应pilot-agent二进制，proxy对应Envoy二进制，它们两个在同一个容器中，discovery service对应pilot-discovery二进制，在另外一个跟应用分开部署的单独的deployment中。
1. discovery service：从Kubernetes apiserver list/watch service、endpoint、pod、node等资源信息，监听istio控制平面配置信息（如VirtualService、DestinationRule等）， 翻译为Envoy可以直接理解的配置格式。
2. proxy：也就是Envoy，直接连接discovery service，间接地从Kubernetes等服务注册中心获取集群中微服务的注册情况
3. agent：生成Envoy配置文件，管理Envoy生命周期
4. service A/B：使用了istio的应用，如Service A/B，的进出网络流量会被proxy接管

#### 相关组件
> Pilot组件核心Pod, 对接平台适配层, 抽象服务注册信息、流量控制模型等, 封装统一的 API，供 Envoy 调用获取.  
包含以下容器:
1. sidecar container istio-proxy
2. container discovery: 主要进程为pilot-discovery discovery
3. 主要监听端口:
	* 15010: 通过grpc 提供的 xds 获取接口
	* 15011: 通过https 提供的 xds 获取接口
	* 8080: 通过http 提供的 xds 获取接口, 兼容v1版本, 另外 http readiness 探针 /ready也在该端口
	* —monitoringPort http self-monitoring 端口, 默认 15014
4. 以上端口通过k8s serviceistio-pilot对外提供服务




### Mixer
> Mixer是Istio的核心组件，提供了遥测数据收集的功能，能够实时采集服务的请求状态等信息，以达到监控服务状态目的。  

#### 架构图
![](istio-%E7%AE%80%E4%BB%8B/90BE8391-7AA1-4428-9AE8-AACB4E0E662D.png)
>  在1.3版本中将大多数常见的安全策略相关的功能（如 RBAC）直接迁移到了 Envoy 中，同时也将大部分遥测功能迁移到了 Envoy 中。现在 Istio proxy 可以直接将收集到的 HTTP 指标暴露给 Prometheus，无需通过 istio-telemetry 服务来中转并丰富指标信息。如果你只关心 HTTP 服务的遥测，可以试试这个新功能，具体步骤参考无 Mixer 的 HTTP 遥测。该功能接下来几个月将会逐渐完善，以便在启用双向 TLS 认证时支持 TCP 服务的遥测。  
#### 核心功能
	* 前置检查（Check）：某服务接收并响应外部请求前，先通过Envoy向Mixer（Policy组件）发送Check请求，做一些access检查，同时确认adaptor所需cache字段，供之后Report接口使用；
	* 配额管理（Quota）：通过配额管理机制，处理多请求时发生的资源竞争；
	* 遥测数据上报（Report）：该服务请求处理结束后，将请求相关的日志，监控等数据，通过Envoy上报给Mixer（telemetry）

#### Mixer工作流程
1. 外部请求服务,请求被envoy拦截,envoy根据请求生成属性,属性作为参数向mix发起check请求.
2. mix进行前置条件检查和配额检查,调用相应的adapter处理,并返回结果.
3. envoy根据结果,执行请求或拒绝请求.
4. 执行请求后向mix服务发起report请求,上报遥测数据.
5. mix的adapter基于上报的数据做进一步处理

#### 相关组件
> mixer 组件包含2个pod, istio-telemetry 和 istio-policy, istio-telemetry负责遥测功能, istio-policy 负责策略控制, 它们分别包含2个容器:  
1. sidecar container istio-proxy
2. mixer: 主要进程为 mixs server 
3. 主要监听端口:
	* 9091: grpc-mixer
	* 15004: grpc-mixer-mtls
	* —monitoring-port: http self-monitoring 端口, 默认 15014, liveness 探针/version


### Citadel
> 安全相关，服务之间访问鉴权等  
#### 相关组件
1. istio-citadel
	* 用于安全相关功能，为服务和用户提供认证和鉴权、管理凭据和 RBAC，挂掉则会导致认证，安全相关功能失效
	* —set security.enabled=false如果要禁能则通过设置security，它对应的镜像就是citadel
	* 如果前期不使用安全相关的功能可以禁能，不会影响整体使用
	* 如果部署了 istio-citadel，则 Envoy 每 15 分钟会进行一次重新启动来刷新证书

### Sidecar(Envoy)
> 一个 C++ 编写的高性能代理服务器，这里做了扩展，在 Istio 中会以 Sidecar 方式跟应用运行在同一 Pod 内，一方面可以接收并执行关于规则、流量拆分等方面的指令，另一方面能够产生各种指标用于监控和跟踪。  


## Istio的Pod
1. istio-citadel-xxx
	- 用于安全相关功能，为服务和用户提供认证和鉴权、管理凭据和 RBAC，挂掉则会导致认证，安全相关功能失效
	- —set security.enabled=false如果要禁能则通过设置security，它对应的镜像就是citadel
	- 如果前期不使用安全相关的功能可以禁能，不会影响整体使用
	- 如果部署了 istio-citadel，则 Envoy 每 15 分钟会进行一次重新启动来刷新证书

2. istio-egressgateway-xxx
	- 出口网关，可选的，根据自身业务形态决定

3. istio-galley—xxx
	- istio API配置的校验、各种配置之间统筹，为 Istio 提供配置管理服务，包含有Kubernetes CRD资源的listener，通过用Kubernetes的Webhook机制对Pilot 和 Mixer 的配置进行验证
	- 这个服务挂掉会导致配置校验异常，是一个必须的组件
	- 比如创建gateway、virtualService等资源，就会需要校验
	- 如果不想校验，可以通过设置helm 的选项参数—set global.configValidation=false来关闭校验

4. istio-ingressgateway-xxx
	- 入口网关，必须的
	- 对外流量入口，所有从外部访问集群内部的服务都需要经过入口网关ingressgateway。需要多实例、防止单点；同时要保证多实例进行负载均衡；
	- 如果异常则导致整个流量入口异常
	- 承担相对较大的并发和高峰流量

5. istio-pilot-xxx
	* 控制sidecar中envoy的启动与参数配置
	* 如果异常则envoy无法正常启动，应用服务的流量无法进行拦截和代理
	* 所有配置、流量规则、策略无法生效
	* 必要组件

6. istio-policy-xxx
	- Mixer相关组件，用于与envoy交互，check需要上报的数据，确定缓存内容，挂掉会影响check相关功能，除非设置为不进行check
	- 不能直接关闭或者说禁能这个策略组件，因为默认请求都是要去policy pods进行check检测的，如果失败则会导致请求失败， [详见](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fistio%2Fistio%2Fissues%2F7663)  
	- 通过安装参数—set mixer.enabled=false禁能

7. istio-sidecar-injector—xxx
	- 现sidecar自动注入功能组件
	- 对应的设置是选项是sidecarInjectorWebhook.enabled，sidecarInjectorWebhook.image对应就是sidecar_injector
	- 这个只是对自动注入Sidecar有影响，如果是通过手动注入kubectl kube-inject命令参数执行的没有影响，不管使能这个前还是后，都对手动注入Sidecar没有影响
	- 如果异常则会导致sidecar无法自动注入；如果注入策略设置为必须注入（policy为Fail），则会导致新创建的应用服务（Pod）无法启动，因为无法注入 Sidecar

8. istio-statsd-prom-bridge—xxx
	- 暴露9102、9125端口
		* 9125是statsdUdpAddress配置的地址
		* 9102是prometheus的Metric接口
			* 这个是istio-statsd-prom-bridge组件提供的服务
	- 这个 statsd是一个转换为prometheus的组件，用来统计envoy 生成的数据，这个是必须的
	- 通过安装参数—set mixer.enabled=false就禁能了这个组件，禁能这个组件会导致ingressgateway的envoy初始化失败，报错日志error initializing configuration ‘/etc/istio/proxy/envoy-rev0.json’: malformed IP address: istio-statsd-prom-bridge，因为statsdUdpAddress这个参数指定了地址为istio-statsd-prom-bridge:9125，因此还需要修改istio这个configmap中的statsdUdpAddress地址
		* 这就需要外部的statsd_exporter支持，关于 [statsd exporter的更多信息查看这里](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fprometheus%2Fstatsd_exporter)  
	- 安装选项global.proxy.envoyStatsd.enabled可以控制envoy是否直接通过statsd上报，global.proxy.envoyStatsd.host和global.proxy.envoyStatsd.port可以设置statsd exporter的地址

9. istio-telemetry—xxx
	- Mixer相关组件的Service，用于采集envoy上报的遥测数据
	- 高并发下会有性能影响，会间接导致整体性能下降，业界针对这个有较多探讨；可以禁能
	- 为提高性能，需要设置为多实例，防止单点；均衡流量
	- 暴露9091、9093、15004、42422端口
		* 9093端口是Mixer组件本身的prometheus暴露的端口，这个是istio-policy 组件提供的
		* 42422是所有 Mixer 生成的网格指标，这个是istio-telemetry 组件提供的
	- 通过安装参数—set mixer.enabled=false禁能
	- 如果异常，则通过Mixer进行上报的一些监控采集数据无法采集到，并不影响整体流程

10. prometheus—xxx
	* 暴露9090端口
	* prometheus组件的Service
	* 如果采用外部的prometheus则不用
	* 其他组件如jaeger、grafana则同样采用外部系统，因此可以不用和istio一起安装

## 最后
> istio 的最大特性,它不拘泥于某种语言.  
