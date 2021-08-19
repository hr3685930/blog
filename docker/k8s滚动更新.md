# k8s 滚动更新
> 滚动更新是一次只更新一小部分副本，成功后，再更新更多的副本，最终完成所有副本的更新。滚动更新的最大的好处是零停机，整个更新过程始终有副本在运行，从而保证了业务的连续性。  
下面我们部署三副本应用，初始镜像为 httpd:2.2.31，然后将其更新到 httpd:2.2.32。
- Httpd:2.2.31 的配置文件如下：
![](k8s%E6%BB%9A%E5%8A%A8%E6%9B%B4%E6%96%B0/775365-20180311063542455-145915125.png)
- 通过 kubectl apply 部署。
![](k8s%E6%BB%9A%E5%8A%A8%E6%9B%B4%E6%96%B0/775365-20180311063557557-887173559.png)
- 部署过程如下：
	1. 创建 Deployment httpd
	2. 创建 ReplicaSet httpd-551879778
	3. 创建三个 Pod
	4. 当前镜像为 httpd:2.2.31
- 将配置文件中 httpd:2.2.31 替换为 httpd:2.2.32，再次执行 kubectl apply。
![](k8s%E6%BB%9A%E5%8A%A8%E6%9B%B4%E6%96%B0/775365-20180311063613409-1357523286.png)
我们发现了如下变化：
	1. Deployment httpd 的镜像更新为 httpd:2.2.32
	2. 新创建了 ReplicaSet httpd-1276601241，镜像为 httpd:2.2.32，并且管理了三个新的 Pod。
	3. 之前的 ReplicaSet httpd-551879778 里面已经没有任何 Pod。

- 结论是：ReplicaSet httpd-551879778 的三个 httpd:2.2.31 Pod 已经被 ReplicaSet httpd-1276601241 的三个 httpd:2.2.32 Pod 替换了。
具体过程可以通过 kubectl describe deployment httpd 查看。
![](k8s%E6%BB%9A%E5%8A%A8%E6%9B%B4%E6%96%B0/775365-20180311063631945-183144683.png)
每次只更新替换一个 Pod：
	1. ReplicaSet httpd-1276601241 增加一个 Pod，总数为 1。
	2. ReplicaSet httpd-551879778 减少一个 Pod，总数为 2。
	3. ReplicaSet httpd-1276601241 增加一个 Pod，总数为 2。
	4. ReplicaSet httpd-551879778 减少一个 Pod，总数为 1。
	5. ReplicaSet httpd-1276601241 增加一个 Pod，总数为 3。
	6. ReplicaSet httpd-551879778 减少一个 Pod，总数为 0。

每次替换的 Pod 数量是可以定制的。Kubernetes 提供了两个参数 maxSurge 和 maxUnavailable 来精细控制 Pod 的替换数量
