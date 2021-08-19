# 容器-kubernetes篇

### kubelet
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/B7641FBB-5C59-4E46-828B-9104732900AF.png)
在Kubernetes项目中，kubelet 主要负责同容器运行时（比如 Docker 项目）打交道是一个称作 CRI（Container Runtime Interface）的远程调用接口，这个接口定义了容器运行时的各项核心操作，比如：启动一个容器需要的所有参数。

此外，kubelet 还通过 gRPC 协议同一个叫作 Device Plugin 的插件进行交互
这个插件是 Kubernetes 项目用来管理 GPU 等宿主机物理设备的主要组件，也是基于Kubernetes 项目进行机器学习训练、高性能作业支持等工作必须关注的功能。
Kubelet 的另一个重要功能，则是调用网络插件和存储插件为容器配置网络和持久化存储,分别是 CNI（Container Networking Interface）和CSI（Container Storage Interface）
从一开始，Kubernetes 项目就没有像同时期的各种“容器云”项目那样，把 Docker 作为整个架构的核心，而仅仅把它作为最底层的一个容器运行时实现。

在master节点上需要打污点(不允许pod调度到master节点上)
一旦某个节点被加上了一个 Taint，即被“打上了污点”，那么所有Pod 就都不能在这个节点上运行，因为 Kubernetes 的 Pod 都有“洁癖”。除非有个别的 Pod 声明自己能“容忍”这个“污点”，即声明了 Toleration，它才可以在这个节点上运行。


### rook
K8s容器存储插件-Rook
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/operator.yaml
kubectl apply -f https://raw.githubusercontent.com/rook/rook/master/cluster/examples/kubernetes/ceph/cluster.yaml

为什么我要选择 Rook?
如果你去研究一下 Rook 项目的实现，就会发现它巧妙地依赖了 Kubernetes 提供的编排能力，合理的使用了很多诸如 Operator、CRD 等重要的扩展特性这使得 Rook 项目，成为了目前社区中基于 Kubernetes API 构建的最完善也最成熟的容器存储插件。我相信，这样的发展路线，很快就会得到整个社区的
推崇。


### k8s.gcr.io/pause的作用
重要概念：Pod内的容器都是平等的关系，共享Network Namespace、共享文件

Pause容器的最主要的作用：创建共享的网络名称空间，以便于其它容器以平等的关系加入此网络名称空间
Pause进程是pod中所有容器的父进程（即第一个进程）
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/6757ECCF-B31E-4317-BC55-D193FA11FC29.png)
Infra 容器永远都是第一个被创建的容器，而其他用户定义的容器，则通过 Join Network Namespace 的方式，与 Infra 容器关联在一起。
Infra 容器一定要占用极少的资源，所以它使用的是一个非常特殊的镜像，叫作：k8s.gcr.io/pause。
这个镜像是一个用汇编语言编写的、永远处于“暂停”状态的容器，解压后的大小也只有 100~200 KB 左右。
而在 Infra 容器“Hold 住(创建)”Network Namespace 后，用户容器就可以加入到 Infra 容器的 Network Namespace 当中了。

### Pod
凡是调度、网络、存储，以及安全相关的属性，基本上是 Pod 级别的。
NodeSelector：是一个供用户将 Pod 与 Node 进行绑定的字段
HostAliases：定义了 Pod 的 hosts 文件（比如 /etc/hosts）里的内容

### Service
- 第一种方式，是以 Service 的 VIP（Virtual IP，即：虚拟 IP）方式
- 第二种方式，就是以 Service 的 DNS 方式,而在第二种 Service DNS 的方式下，具体还可以分为两种处理方法：
	- 第一种处理方法，是 Normal Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，正是 my-svc 这个 Service 的 VIP，后面的流程就跟 VIP 方式一致了。
	- 而第二种处理方法，正是 Headless Service。这种情况下，你访问“my-svc.my-namespace.svc.cluster.local”解析到的，直接就是 my-svc 代理的某一个 Pod 的 IP 地址。(  clusterIP: None)
可以看到，这里的区别在于，Headless Service 不需要分配一个 VIP，而是可以直接以 DNS 记录的方式解析出被代理 Pod 的 IP 地址。


### StatefulSet
首先，StatefulSet 的控制器直接管理的是 Pod。
其次，Kubernetes 通过 Headless Service，为这些有编号的 Pod，在 DNS 服务器中生成带有同样编号的 DNS 记录。
最后，StatefulSet 还为每一个 Pod 分配并创建一个同样编号的 PVC。

## DaemonSet
DaemonSet 的“过人之处”，其实就是依靠Toleration(污点) 实现的

## Job cronJob
我再和你分享三种常用的、使用 Job 对象的方法。
- 第一种用法，也是最简单粗暴的用法：外部管理器 +Job 模板。
这种模式的特定用法是：把 Job 的 YAML 文件定义为一个“模板变量”，然后用一个外部工具控制这些“模板变量”来生成 Job。

- 第二种用法：拥有固定任务数目的并行 Job
可以使用一个工作队列（Work Queue）进行任务分发。

- 第三种用法，也是很常用的一个用法：指定并行度（parallelism），但不设置固定的completions 的值。


## APIServer
自定义控制器(CRD) 


## Operator
Operator 的工作原理，实际上是利用了 Kubernetes 的自定义 API 资源（CRD），来描述我们想要部署的“有状态应用”；然后在自定义控制器里，根据自定义 API 对象的变化，来完成具体的部署和运维工作。

Operator是开发CRD的一个脚手架项目，目的是帮我们实现CRD，那么一定就有一个疑问，为什么要实现CRD呢？我举2个例子：
* 假设公司的PHP项目都是deployment+service+ingress的统一玩法，那么我们就可以定义一个CRD叫做php-deployment，创建一个这样的资源对象，就可以全自动的拉起deployment、service、ingress，并且全程监控它们的生命期，这还不够强大嘛。
* 比如我们想给研发提供一键创建mysql的服务，我们就可以实现一个CRD，它可以帮我们创建stateful的mysql实例，挂载PVC持久卷，并且发一封邮件通知我们mysql POD的启停事件，这就很强大了嘛。


## Volume
挂载两阶段
Attach和Mount
为虚拟机挂载远程磁盘的操作，对应的正是“两阶段处理”的第一阶段。在
Kubernetes 中，我们把这个阶段称为 Attach。attach是加载，是把磁盘加载到主机

将磁盘设备格式化并挂载到 Volume 宿主机目录的操作，对应的正是“两阶段处理”的第二个阶段，我们一般称为：Mount。mount叫做挂载，包括格式化磁盘和mount到本地plugin目录下面。

自动创建PV  StorageClass

本地存储
hostPath volume和 Local Persistent Volume (推荐)  

Local volumes的最佳实践
* 为了更好的IO隔离效果，建议将一整块磁盘作为一个存储卷使用；
* 为了得到存储空间的隔离，建议为每个存储卷使用一个独立的磁盘分区；
* 避免重新创建同名的 Node，否则会导致新 Node 无法识别已绑定旧 Node 的 PV 否则，系统可能会认为新Node节点包含旧的PV。
* 对于具有文件系统的存储卷，建议在fstab条目和该卷的mount安装点的目录名中使用它们的UUID（例如ls -l /dev/disk/by-uuid的输出）。 这种做法可确保不会安装错误的本地卷，即使其设备路径发生了更改（例如，如果/dev/sda1在添加新磁盘时变为/dev/sdb1）。 此外，这种做法将确保如果创建了具有相同名称的另一个节点时，该节点上的任何卷仍然都会是唯一的，而不会被误认为是具有相同名称的另一个节点上的卷。

开发存储插件
FlexVolume 和 CSI

CSI
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/748DB4E7-A57D-463B-A9F0-5BB2EA666E07.png)

这套存储插件体系多了三个独立的外部组件（External Components），即：
Driver Registrar、External Provisioner 和 External Attacher，对应的正是从
Kubernetes 项目里面剥离出来的那部分存储管理功能。

- Driver Registrar 组件，负责请求 Identity Service 来获取插件信息并且注册到 kubelet 

- External Provisioner 组件，负责的正是 Provision 阶段
观察 APIServer 的 PVC 对象，PVC 被创建时，就会调用 CSI Controller 的 CreateVolume 方法，创建对应 PV

- External Attacher 组件，负责的正是“Attach 阶段
观察APIServer 的 VolumeAttachment 对象，就会调用 CSI Controller 服务的 ControllerPublish 方法，完成它所对应的 Volume 的 Attach 阶段


CSI Identity、CSI Controller 和 CSINode。
- CSI Identity 服务，负责对外暴露这个插件本身的信息
- CSI Controller 服务，创建以及管理volume管理卷
- CSI Node ,将volume存储卷挂载到指定目录中


相比于 FlexVolume，CSI 的设计思想，把插件的职责从“两阶段处理”，扩展
成了 Provision、Attach 和 Mount 三个阶段。其中，Provision 等价于“创建磁盘”，Attach 等价于“挂载磁盘到虚拟机”，Mount 等价于“将该磁盘格式化后，挂载在Volume 的宿主机目录上”。


部署 CSI 插件的常用原则:
- 第一，通过 DaemonSet 在每个节点上都启动一个 CSI 插件，来为 kubelet 提供 CSINode 服务。

- 第二，通过 StatefulSet 在任意一个节点上再启动一个 CSI 插件，为 External
Components 提供 CSI Controller 服务。


## 网络
docker
> 被限制在 Network Namespace 里的容器进程，实际上是通过 Veth Pair 设备 + 宿主机网桥的方式，实现了  
> 跟同其他容器的数据交换。  

![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/3B2A8072-B1FC-4822-927D-9818FEFD861F.png)

Flannel 项目是 CoreOS 公司主推的容器网络方案。事实上，Flannel 项目本身只是一个框架，真正为我们提供容器网络功能的，是 Flannel 的后端实现。目前，Flannel 支持三种后端实现，分别是：
1  VXLAN
2  host-gw
3  UDP


### UDP (已弃用)
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/C1A1E996-1146-496A-8F11-E41966249EED.png)
可以看到，Flannel UDP 模式提供的其实是一个三层的 Overlay 网络，即：它首先对发出端的 IP 包进行 UDP 封装，然后在接收端进行解封装拿到原始的 IP 包，进而把这个 IP 包转发给目标容器。这就好比，Flannel 在不同宿主机上的两个容器之间打通了一条“隧道”，使得这两个容器可以直接使用 IP 地址进行通信，而无需关心容器和宿主机的分布情况。

为什么弃用?
经过三次用户态与内核态之间的数据拷贝
第一次：用户态的容器进程发出的 IP 包经过 docker0 网桥进入内核态；
第二次：IP 包根据路由表进入 TUN（flannel0）设备，从而回到用户态的 flanneld 进程；
第三次：flanneld 进行 UDP 封包之后重新进入内核态，将 UDP 包通过宿主机的 eth0 发出去。

### vxlan
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/B6AC9DEF-65CE-4594-AC3D-C6489B39A109.png)
VTEP 
VTEP 设备的作用，其实跟前面的 flanneld 进程非常相似。
只不过，它进行封装和解封装的对象，是二层数据帧（Ethernet frame）；
而且这个工作的执行流程，全部是在内核里完成的（因为 VXLAN 本身就是 Linux 内核中的一个模块）。



CNI接口包含几个接口

type CNI interface {
    AddNetworkList(net *NetworkConfigList, rt *RuntimeConf) (types.Result, error)
    DelNetworkList(net *NetworkConfigList, rt *RuntimeConf) error

    AddNetwork(net *NetworkConfig, rt *RuntimeConf) (types.Result, error)
    DelNetwork(net *NetworkConfig, rt *RuntimeConf) error
}

该接口只有四个方法，添加网络、删除网络、添加网络列表、删除网络列表。

CNI插件
flannel的网络划分和如何与docker交互，如何通过iptables访问service？
UDP 模式创建的是 TUN 设备，VXLAN 模式创建的则是 VTEP 设备
第一次：用户态的容器进程发出的 IP 包经过 docker0 网桥进入内核态；
第二次：IP 包根据路由表进入 TUN（flannel0）设备，从而回到用户态的 flanneld 进程；
第三次：flanneld 进行 UDP 封包之后重新进入内核态，将 UDP 包通过宿主机的 eth0 发出去。


Kubernetes 是通过一个叫作 CNI 的接口，维护了一个单独的网桥来代替 docker0。
这个网桥的名字就叫作：CNI 网桥，它在宿主机上的设备名称默认是：cni0。

网络创建步骤
1 kubelet runtime创建network namespace
kubelet先创建pause容器，并为这个pause容器生成一个network namespace，然后把这个network namespace关联到pause容器上

2 Kubelet触发CNI插件
在触发cni插件的时候会将cni的配置load给cni插件
```
添加或者删除网卡
CNI_COMMAND=ADD/DEL
# 容器的ID
CNI_CONTAINERID=xxxxxxxxxxxxxxxxxxx
#  容器网络空间主机映射路径
CNI_NETNS=/proc/4390/ns/net
# CNI参数，使用分号分开，主要是POD的一些相关信息
CNI_ARGS=IgnoreUnknown=1;K8S_POD_NAMESPACE=default;K8S_POD_NAME=22-my-nginx-2523304718-7stgs;K8S_POD_INFRA_CONTAINER_ID=xxxxxxxxxxxxxxxx
# 容器内网卡的名称
CNI_IFNAME=eth0
# CNI二进制文件路径
CNI_PATH=/opt/cni/bin.
```
3 kubelet调用相应CNI插件
CNI插件创建veth pair
通过IPAM插件获取空闲的ip地址
将ip配置到容器network namespace的网络设备


上面都是二层网络

接下来介绍三层网络
Flannel 的 host-gw 模式和Calico

Host-gw 模式的工作原理，其实就是将每个 Flannel 子网（FlannelSubnet，比如：10.244.1.0/24）的“下一跳”，设置成了该子网对应的宿主机的 IP 地址。
也就是说，这台“主机”（Host）会充当这条容器通信路径里的“网关”（Gateway）。这也正是“host-gw”的含义

### Calico
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/820B9331-C79C-4506-BDF5-9CD74A4068D8.png)

不同于 Flannel 通过 Etcd 和宿主机上的 flanneld 来维护路由信息的做法，Calico项目使用了一个“重型武器”来自动地在整个集群中分发路由信息
BGP 的全称是 Border Gateway Protocol，即：边界网关协议
它跟普通路由器的不同之处在于，它的路由表里拥有其他自治系统里的主机路由信息。
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/0C582849-0991-4192-94C6-5DF19256418D.png)
所谓 BGP，就是在大规模网络中实现节点路由信息共享的一种协议。
除了对路由信息的维护方式之外，Calico 项目与 Flannel 的 host-gw 模式的另一个不同之处，就是它不会在宿主机上创建任何网桥设备。

三层和隧道的异同：相同之处是都实现了跨主机容器的三层互通，而且都是通过对目的 MAC 地址的操作来实现的；
不同之处是三层通过配置下一条主机的路由规则来实现互通，隧道则是通过通过在IP 包外再封装一层 MAC 包头来实现。
三层的优点：少了封包和解包的过程，性能肯定是更高的。


### 网络隔离能力
NetworkPolicy 
Kubernetes 网络插件对 Pod 进行隔离，其实是靠在宿主机上生成NetworkPolicy 对应的 iptable 规则来实现的。


### Service
Service 是由 kube-proxy 组件，加上 iptables 来共同实现的。
一直以来，基于 iptables 的 Service 实现，都是制约 Kubernetes 项目承载更多量级的 Pod 的主要障碍。
IPVS 并不需要在宿主机上为每个 Pod 设置 iptables 规则，而是把对这些“规则”的处理放到了内核态，从而极大地降低了维护这些规则的代价。
在大规模集群里，我非常建议你为 kube-proxy 设置Cproxy-mode=ipvs 来开启这个功能。它为 Kubernetes 集群规模带来的提升，还是非常巨大的。

Service 机制，以及Kubernetes 里的 DNS 插件，都是在帮助你解决同样一个问题
即：如何找到我的某一个容器？这个问题在平台级项目中，往往就被称作服务发现

ClusterIP 模式的 Service 为你提供的，就是一个 Pod 的稳定的 IP 地址，即 VIP。并且，这里 Pod 和 Service 的关系是可以通过 Label 确定的。
Headless Service 为你提供的，则是一个 Pod 的稳定的 DNS 名字，并且，这个名字是可以通过 Pod 名字和 Service 名字拼接出来的。

所谓 Service，其实就是 Kubernetes 为 Pod 分配的、固定的、基于 iptables（或者 IPVS）的访问入口。
而这些访问入口代理的 Pod 信息，则来自于Etcd，由 kube-proxy 通过控制循环来维护。

所谓 Ingress 对象，其实就是 Kubernetes 项目对“反向代理”的一种抽象


## 资源
CPU 这样的资源被称作“可压缩资源”
当可压缩资源不足时，Pod 只会“饥饿”，但不会退出

内存这样的资源，则被称作“不可压缩资源（incompressible resources）。
当不可压缩资源不足时，Pod 就会因为 OOM（Out-Of-Memory）被内核杀掉。

你也可以直接把这个配置写成 cpu=0.5。但在实际使用时，我还是推荐你使用500m 的写法，毕竟这才是 Kubernetes 内部通用的 CPU 表示方式。
Kubernetes 里 Pod 的 CPU 和内存资源，实际上还要分为 limits 和requests 两种情况

我强烈建议你将 DaemonSet 的 Pod 都设置为Guaranteed（Pod 里的每个容器都必须有内存/CPU 限制和请求，而且值必须相等） 的 QoS 类型。
否则，一旦 DaemonSet 的 Pod 被回收，它又会立即在原宿主机上被重建出来，这就使得前面资源回收的动作，完全没有意义了。


## 调度器
Kubernetes 的调度器的核心，实际上就是两个相互独立的控制循环。
第一个控制循环，我们可以称之为 Informer Path。
它的主要目的，是启动一系列Informer，用来监听（Watch）Etcd 中 Pod、Node、Service 等与调度相关的 API 对象的变化。
第二个控制循环，是调度器负责 Pod 调度的主循环，我们可以称之为 Scheduling Path
就是不断地从调度队列里出队一个 Pod。然后，调用Predicates 算法进行“过滤”。这一步“过滤”得到的一组 Node，就是所有可以运行这个 Pod 的宿主机列表。

未来会将默认调度器进一步做轻做薄，并且插件化，才是 kube-scheduler 正确的演进方向。

调度流程
预选（Predicates）：输入是所有节点，输出是满足预选条件的节点。kube-scheduler根据预选策略过滤掉不满足策略的Nodes。
例如，如果某节点的资源不足或者不满足预选策略的条件如“Node的label必须与Pod的Selector一致”时则无法通过预选。

优选（Priorities）：输入是预选阶段筛选出的节点，优选会根据优先策略为通过预选的Nodes进行打分排名，选择得分最高的Node。
例如，资源越富裕、负载越小的Node可能具有越高的排名。

优先级（Priority ）和抢占（Preemption）机制。
优先级和抢占机制，解决的是 Pod 调度失败时该怎么办的问题。




## SIG-Node与CRI
Kubelet 调用下层容器运行时的执行过程，并不会直接调用 Docker的 API，而是通过一组叫作 CRI（Container Runtime Interface，容器运行时接口）的gRPC 接口来间接执行的。

CRI是由SIG-Node来维护的。

### kubelet
Kubelet 的主要功能就是定时从某个地方获取节点上 pod/container 的期望状态（运行什么容器、运行的副本数量、网络或者存储如何配置等等），并调用对应的容器平台接口达到这个状态。
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/27B0E6D2-A4A6-4670-8F29-4BC1E982716C.png)
#### Pod 管理
#### 容器健康检查
创建了容器之后，kubelet 还要查看容器是否正常运行，如果容器运行出错，就要根据设置的重启策略进行处理。检查容器是否健康主要有三种方式：执行命令，http Get，和tcp连接。不管用什么方式，如果检测到容器不健康，kubelet 会删除该容器，并根据容器的重启策略进行处理（比如重启，或者什么都不做）。

#### 容器监控
监控所在节点的资源使用情况，并定时向 master 报告。kubelet 使用cAdvisor进行资源使用率的监控

工作流程
当Kubernetes通过编排能力创建了一个Pod之后，调度器会为这个Pod选择一个具体的节点来运行。这个时候，kubelet当然就会通过SyncLoop来判断需要进行的操作，比如创建一个Pod。kubelet之后就会调用一个叫做GenericRuntime的通用组件来发起一个创建一个Pod的CRI grpc请求。如果使用的容器项目是Docker的话，会有一个叫做dockershim的组件，接收到CRI请求并解析后，组装成一个Docker API请求发送给Docker Daemon。
![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/0FDF8DFE-94B7-4B39-B1E6-548053B769E7.png)


#### CRI-shim
CRI-shim 作为CRI中定义的接口的实现，本质上其实就是一个gRPC server。对于docker来说这个shim就是docker-shim，对于cri-containerd来说，cri-shim就是cri-containerd

![](%E5%AE%B9%E5%99%A8-kubernetes%E7%AF%87/52085C72-8DB2-4A55-BD6E-4DB57603A457.png)


## 日志与监控
第一种 Metrics，是宿主机的监控数据。
第二种 Metrics，是来自于 Kubernetes 的 API Server、kubelet 等组件的 metrics API。
第三种 Metrics，是 Kubernetes 相关的监控数据。这其中包括了 Pod、Node、容器、Service 等主要Kubernetes 核心概念的 Metrics。容器相关的 Metrics 主要来自于 kubelet 内置的 cAdvisor 服务

Metrics Server 并不是 kube-apiserver 的一部分，而是通过 Aggregator
这种插件机制，在独立部署的情况下同 kube-apiserver 一起统一对外提供API监控数据服务的。


对于一个多实例应用来说，通过 Service 来采集 Pod 的 Custom Metrics 其实才是合理的做法



