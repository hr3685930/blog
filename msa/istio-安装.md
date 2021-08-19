# istio-安装
## 先决条件
1.  [安装Kubernetes](https://links.jianshu.com/go?to=https%3A%2F%2Fkubernetes.io%2Fdocs%2Fsetup%2Fproduction-environment%2Ftools%2Fkubeadm%2Fcreate-cluster-kubeadm%2F) 
2.  [安装 Helm （ 版本高于 2.10）](https://links.jianshu.com/go?to=https%3A%2F%2Fdocs.helm.sh%2Fusing_helm) 。

## 部署步骤
使用 Helm 部署 Istio主要有以下两个步骤：
1. 下载 Istio 发布包
2. 部署 Istio

### 1. 下载 Istio 发布包
主要步骤：
> 1. 下载和自动解压缩 Istio 发布包  
> 2. 配置 Istio 环境变量  
1. 下载和自动解压缩 Istio 发布包
> curl -L https://git.io/getLatestIstio | ISTIO_VERSION=1.2.3 sh -  
2. 版本选择可以参考：  [Istio release](https://links.jianshu.com/go?to=https%3A%2F%2Fgithub.com%2Fistio%2Fistio%2Freleases)  页面
3. 配置 Istio 环境变量
> cd istio-1.2.3  
> export PATH=$PWD/bin:$PATH  

### 2. 部署 Istio
主要步骤：
> 1. 改造 Tiller 客户端（为 Tiller 配置 Service account）  
> 2. 安装 istio-init（用于启动 Istio CRD 的安装过程）  
> 3. 部署 Istio （采用 demo 配置文件）  
1. 改造 Tiller 客户端（为 Tiller 配置 Service account）
* 卸载原有的 Tiller 客户端
> helm reset  
* 创建一个 Tiller 的 Service account
> cd istio-1.2.3  
* kubectl apply -f install/kubernetes/helm/helm-service-account.yaml

* 安装 Tiller 并为其配置 Service account ：
> helm init —upgrade \  
> -i registry.cn-hangzhou.aliyuncs.com/google_containers/tiller:v2.14.3 \  
> —stable-repo-url https://kubernetes.oss-cn-hangzhou.aliyuncs.com/charts \  
> —service-account tiller  
注：这里修改了另外两点：
	1. 设置 Tiller 的安装源为阿里云源
	2. 设置 Helm 的稳定存储库为阿里云的仓库

2. 安装 istio-init（用于启动 Istio CRD 的安装过程） 
> helm install install/kubernetes/helm/istio-init —name istio-init —namespace istio-system —set certmanager.enabled=true  
注：这里启用了 cert-manager
* 验证 CRD  
> kubectl get crds | grep ‘istio.io\|certmanager.k8s.io’ | wc -l  
注：启用了 cert-manager 时，CRD 的个数为 28 个，否则为 26 个

3. 部署 Istio （采用 **demo** 配置文件）
> helm install install/kubernetes/helm/istio —name istio —namespace istio-system \  
> —values install/kubernetes/helm/istio/values-istio-demo.yaml  
注：虽然官方推荐的是使用 **default** 配置文件，但从官方的详细说明来看 **demo** 配置文件提供的功能更加完善，详细参考： [https://istio.io/docs/setup/kubernetes/additional-setup/config-profiles/](https://links.jianshu.com/go?to=https%3A%2F%2Fistio.io%2Fdocs%2Fsetup%2Fkubernetes%2Fadditional-setup%2Fconfig-profiles%2F) 

4. 卸载 Istio
> helm delete —purge istio  
> helm delete —purge istio-init  
到这里，Istio 就已经安装完成了
