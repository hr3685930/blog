# Docker Swarm

> Swarm可以用来管理Docker集群，它将一群Docker主机虚拟为单一主机。 新版Swarm被集成进了Docker Engine，并通过Docker CLI更方便的创建、管理集群。  

## 基本概念
swarm集群包含以下两种节点：  
* Manager Node —— 负责调度Task
* Worker Node —— 接受Manager Node调度并指派的Task
每个node的availability包括：
* Active：接受task
* Pause：不接受task，已有task仍运转
* Drain：不接受task，已有task被交接

## 集群构建

```
# 创建集群的Manager node
docker swarm init --advertise-addr Manager节点ip

# 管理join token（去中心化设计）
docker swarm join-token worker|manager

# worker node加入集群，成为一个工作副本
docker swarm join --token 令牌 Manager节点ip:2377

# 查看集群节点
docker node ls

# 检查指定节点
docker node inspect self|NODE

# 节点状态变更
docker node update --availability drain manager # 设置Manager Node只具有管理功能

# 节点打标签
docker node update --label-add 键名称=值

# 节点提权/降权
docker node promote 节点 # 升级为manager node
docker node demote 节点 # 降级为worker node

# 节点退出集群
docker swarm node leave [--force]
服务管理
Service 包含两种模式：
* replicated：可指定服务驻留节点数量（默认）
* global：服务驻留在全部节点
# 在集群中部署service
docker service create --replicas 副本数 --name 服务名称 镜像名称 command

# 滚动更新服务
docker service update --update-parallelism 并行更新限额数 --update-delay 更新间隔秒数 --image 镜像名称 服务名称

# 查看集群服务列表
docker service ls

# 查询指定服务的部署状况
docker service ps 服务名称

# 检查指定服务
docker service inspect --pretty 服务名称

# 集群服务扩容缩容
docker service scale 服务名称=节点数

# 删除集群指定服务及相关的副本
docker service rm 服务名称
网络管理
容器加入到同一网络中可以实现互访
# 创建网络
docker network create --driver overlay 网络名称

# 创建服务时指定网络
docker service create --network 网络名称 --replicas 副本数 --name 服务名称 镜像名称 command
```