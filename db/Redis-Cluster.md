# Redis-Cluster
## Docker Swarm
服务器
192.168.1.88        manager  管理
192.168.1.91         node1       节点1
192.168.1.92         node2      节点2

- swarm初始化 (192.168.1.88)
```
[root@manager ~]# docker swarm init --listen-addr 192.168.1.88:2377
Swarm initialized: current node (brz1h2hl97k6xon4gosj36add) is now a manager.
 
To add a worker to this swarm, run the following command:
 
docker swarm join \
--token SWMTKN-1-3uged1s7vnfvwhhqdlsvh639w4ve25br8hmp6cygijnw35ntd4-2jbs7f7osr1kvmwvq7s1jgero \
192.168.1.88:2377
 
To add a manager to this swarm, run 'docker swarm join-token manager' and follow the instructions.
```

- 两个节点复制并运行上面的命令(docker swarm join \.......)
```
[root@node1 ~]# docker swarm join \
> --token SWMTKN-1-3uged1s7vnfvwhhqdlsvh639w4ve25br8hmp6cygijnw35ntd4-2jbs7f7osr1kvmwvq7s1jgero \
> 192.168.1.88:2377
This node joined a swarm as a worker.
 
  
 
[root@node2 ~]# docker swarm join \
> --token SWMTKN-1-3uged1s7vnfvwhhqdlsvh639w4ve25br8hmp6cygijnw35ntd4-2jbs7f7osr1kvmwvq7s1jgero \
> 192.168.1.88:2377
This node joined a swarm as a worker.
```

- Manager 查看节点列表
```
[root@manager ~]# docker node ls
ID HOSTNAME STATUS AVAILABILITY MANAGER STATUS
4worlu3j8bobphbxcghioq0bb localhost.localdomain Ready Active
ajszugb3lsnnrrel8ae25elww localhost.localdomain Ready Active
brz1h2hl97k6xon4gosj36add * localhost.localdomain Ready Active Leader
```

- Manager 创建overlay 网络
```
[root@manager ~]# docker network create --driver overlay net1
7x023eqvxtr606i4hwfc6shek 
[root@manager ~]# docker network ls
NETWORK ID NAME DRIVER SCOPE
a90ef5a0aee2 bridge bridge local
aaf7b93e4018 docker_gwbridge bridge local
8bd60b3a381e host host local
2okhwlbxv046 ingress overlay swarm
7x023eqvxtr6 net1 overlay swarm
f54d83608389 none null local
```
  
 
### 注意： 由于自身设计的原因, 在没有跑起应用前, 节点上不会存在net1 网络.
 

## Redis Cluster
- 各主机 Pull 镜像
```
[root@manager ~]# docker pull zeffee/redis-cluster
Using default tag: latest
``` 

- manager创建redis cluster应用
创建3台redis主服务器, swarm将自动平均分配到三台主机上
```
[root@manager ~]# docker service create --replicas 3 --network net1 --name master -e auth=123456 zeffee/redis-cluster
cw7f9763grrt0vdu7ag37b05r
```
 
- 同理, 创建三台从服务器
```
[root@manager ~]# docker service create --replicas 3 --network net1 --name slave -e auth=123456 zeffee/redis-cluster
9e7pke4bax90v4d1d577jpfvm
```
 
- 查看实例列表
> manager  
```
[root@manager ~]# docker ps
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
c850fa618054 zeffee/redis-cluster:latest "/start.sh" 10 seconds ago Up 8 seconds slave.2.3a8gvy69v58ym4na6xyhh9sqe
9d57ac8653b4 zeffee/redis-cluster:latest "/start.sh" 8 minutes ago Up 8 minutes master.1.dkbgmb6h7qkdian9ayflu702t
```
 
> node1  
```
[root@node2 ~]# docker ps -a
CONTAINER ID IMAGE COMMAND CREATED STATUS PORTS NAMES
f9f617b0277b zeffee/redis-cluster:latest "/start.sh" 4 seconds ago Up 2 seconds slave.3.4xs6e0x9m6umbnul0oxhz1iix
33034c660f2d zeffee/redis-cluster:latest "/start.sh" 8 minutes ago Up 8 minutes master.2.3cids3db5c980zzcpv1087luw

......
```

- 获取各容器的net1网络的ip地址
```
# Manager (192.168.1.88)
 
[root@manager ~]# docker network inspect net1
[
{
"Name": "net1",
......
"Containers": {
"9d57ac8653b409299a6bed473ff3ad25d09e64d95d3a7e4d78ff3a8fcb540244": {
"Name": "master.1.dkbgmb6h7qkdian9ayflu702t",
"EndpointID": "ade78de5ad95dcee2aa4c484b79b5e137d2a097020c769e398eb5f0b355f904f",
"MacAddress": "02:42:0a:00:00:03",
"IPv4Address": "10.0.0.3/24",
"IPv6Address": ""
},
"c850fa6180540157bce466af4d565128481a18463614017a7eefd0d3b2dcd04c": {
"Name": "slave.2.3a8gvy69v58ym4na6xyhh9sqe",
"EndpointID": "08f3ae8cdca516de9e460e45673bd52b418ecfc7e7045ff1c58334506a980b15",
"MacAddress": "02:42:0a:00:00:08",
"IPv4Address": "10.0.0.8/24",
"IPv6Address": ""
}
 
......
}
]
```
提取出 10.0.0.3 以及 10.0.0.8 。
 
- 同理， 获取node1  以及node2 的容器的ip地址
```
[root@node1  ~]# docker network inspect net1
```
10.0.0.7  和 10.0.0.5
  
```
[root@node2  ~]# docker network inspect net1
```
10.0.0.9  和 10.0.0.4
 
## 创建 redis-cluster 集群
- 随便进入一个容器
```
[root@localhost ~]# docker exec -ti 772 /bin/bash
```
 
- 执行集群命令
```
[root@772b766adefb /]# redis-trib.rb create --replicas 1 10.0.0.3:6379 10.0.0.4:6379 10.0.0.5:6379 10.0.0.7:6379 10.0.0.8:6379 10.0.0.9:6379
>>> Creating cluster
>>> Performing hash slots allocation on 6 nodes...
Using 3 masters:
10.0.0.3:6379
10.0.0.4:6379
10.0.0.5:6379
Adding replica 10.0.0.7:6379 to 10.0.0.3:6379
Adding replica 10.0.0.8:6379 to 10.0.0.4:6379
Adding replica 10.0.0.9:6379 to 10.0.0.5:6379
M: 1977c4d47bfbed850375b2c40d798e4b37e2c070 10.0.0.3:6379
slots:0-5460 (5461 slots) master
M: 69a0bef4416aca326334766413d852afa96c112c 10.0.0.4:6379
slots:5461-10922 (5462 slots) master
M: 4744ffb2731aa838bca97927ee0575cb785f7ee6 10.0.0.5:6379
slots:10923-16383 (5461 slots) master
S: 708fed786ff88bdd9ec75b2711e5e664f64e5ac8 10.0.0.7:6379
replicates 1977c4d47bfbed850375b2c40d798e4b37e2c070
S: 080d52329f736c96f7a7d37883ae5f482859d1f6 10.0.0.8:6379
replicates 69a0bef4416aca326334766413d852afa96c112c
S: 105f108d1bf695166ba6d48a0d0fd060b0cb782d 10.0.0.9:6379
replicates 4744ffb2731aa838bca97927ee0575cb785f7ee6
Can I set the above configuration? (type 'yes' to accept): yes
>>> Nodes configuration updated
>>> Assign a different config epoch to each node
>>> Sending CLUSTER MEET messages to join the cluster
Waiting for the cluster to join...
>>> Performing Cluster Check (using node 10.0.0.3:6379)
M: 1977c4d47bfbed850375b2c40d798e4b37e2c070 10.0.0.3:6379
slots:0-5460 (5461 slots) master
1 additional replica(s)
S: 080d52329f736c96f7a7d37883ae5f482859d1f6 10.0.0.8:6379
slots: (0 slots) slave
replicates 69a0bef4416aca326334766413d852afa96c112c
S: 708fed786ff88bdd9ec75b2711e5e664f64e5ac8 10.0.0.7:6379
slots: (0 slots) slave
replicates 1977c4d47bfbed850375b2c40d798e4b37e2c070
M: 69a0bef4416aca326334766413d852afa96c112c 10.0.0.4:6379
slots:5461-10922 (5462 slots) master
1 additional replica(s)
S: 105f108d1bf695166ba6d48a0d0fd060b0cb782d 10.0.0.9:6379
slots: (0 slots) slave
replicates 4744ffb2731aa838bca97927ee0575cb785f7ee6
M: 4744ffb2731aa838bca97927ee0575cb785f7ee6 10.0.0.5:6379
slots:10923-16383 (5461 slots) master
1 additional replica(s)
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
```
  
- 集群创建成功
测试可用
```
[root@772b766adefb /]# redis-cli -c -h 192.168.1.91 -p 7001
192.168.1.91:7001> auth 123456
OK
192.168.1.91:7001>
192.168.1.91:7001>
192.168.1.91:7001> set a dfdf
-> Redirected to slot [15495] located at 10.0.0.5:6379
(error) NOAUTH Authentication required.
10.0.0.5:6379> auth 123456
OK
 
10.0.0.5:6379> set name zeffee
OK
```
  
- 模拟故障转移
查看现在的集群情况
```
[root@772b766adefb /]# redis-cli -a 123456 cluster nodes
1977c4d47bfbed850375b2c40d798e4b37e2c070 10.0.0.3:6379 slave 708fed786ff88bdd9ec75b2711e5e664f64e5ac8 0 1478511905710 8 connected
4744ffb2731aa838bca97927ee0575cb785f7ee6 10.0.0.5:6379 master - 0 1478511907215 3 connected 10923-16383
080d52329f736c96f7a7d37883ae5f482859d1f6 10.0.0.8:6379 slave 69a0bef4416aca326334766413d852afa96c112c 0 1478511906714 5 connected
708fed786ff88bdd9ec75b2711e5e664f64e5ac8 10.0.0.7:6379 myself,master - 0 0 8 connected 0-5460
69a0bef4416aca326334766413d852afa96c112c 10.0.0.4:6379 master - 0 1478511907216 2 connected 5461-10922
105f108d1bf695166ba6d48a0d0fd060b0cb782d 10.0.0.9:6379 slave 4744ffb2731aa838bca97927ee0575cb785f7ee6 0 1478511907216 6 connected
```
- 关闭某个10.0.0.4节点 ， 模拟shutdown
- docker stop f85
```
[root@772b766adefb /]# redis-cli -a 123456 cluster nodes
......
4744ffb2731aa838bca97927ee0575cb785f7ee6 10.0.0.5:6379 master - 0 1478512172623 3 connected 10923-16383
080d52329f736c96f7a7d37883ae5f482859d1f6 10.0.0.8:6379 master - 0 1478512171112 9 connected 5461-10922
708fed786ff88bdd9ec75b2711e5e664f64e5ac8 10.0.0.7:6379 myself,master - 0 0 8 connected 0-5460
69a0bef4416aca326334766413d852afa96c112c 10.0.0.4:6379 master,fail - 1478512160631 1478512158518 2 connected
.....
```
可以看出10.0.0.4 显示fail , 而集群选举出了 10.0.0.8 作为新的master

- 节点增加
```
[root@manager ~]# docker service create --name redis007 --network=net1 zeffee/redis-cluster
66y4jkj5qpgt4fkhzlt5glipv

[root@772b766adefb /]# redis-trib.rb add-node 10.0.0.13:6379 10.0.0.3:6379

```
- 重新分片
```
[root@772b766adefb /]# redis-trib.rb reshard 10.0.0.3:6379
```

- 节点删除
副节点可以直接删除

- 以刚才新增的副节点为例子 ， 10.0.0.15 的id 为 05ad918a90f580aa6a8b465e2d0ec570d4bbe2eb
```
[root@772b766adefb /]# redis-trib.rb del-node 10.0.0.3:6379 05ad918a90f580aa6a8b465e2d0ec570d4bbe2eb
>>> Removing node 05ad918a90f580aa6a8b465e2d0ec570d4bbe2eb from cluster 10.0.0.3:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
```
主节点的删除需要先删除从节点，并且将本来的哈希槽迁移出去才可以删除

- 从节点的删除参照上面的，现在将哈希槽迁移至其他节点
 以刚才新增的10.0.0.13 主节点为例子， 拥有4096 个哈希槽， 平均分配给剩下的三台主节点将是每台分配 4096/3=1365 个哈希槽
```
[root@772b766adefb /]# redis-trib.rb reshard 10.0.0.3:6379
>>> Performing Cluster Check (using node 10.0.0.3:6379)
S: 1977c4d47bfbed850375b2c40d798e4b37e2c070 10.0.0.3:6379
slots: (0 slots) slave
replicates 708fed786ff88bdd9ec75b2711e5e664f64e5ac8
M: 708fed786ff88bdd9ec75b2711e5e664f64e5ac8 10.0.0.7:6379
slots:1365-5460 (4096 slots) master
1 additional replica(s)
M: 8fabdaedcc475069d78f31bb05ee0c954b3daf7f 10.0.0.13:6379
slots:0-1364,5461-6826,10923-12287 (4096 slots) master
0 additional replica(s)
M: 080d52329f736c96f7a7d37883ae5f482859d1f6 10.0.0.8:6379
slots:6827-10922 (4096 slots) master
0 additional replica(s)
M: 4744ffb2731aa838bca97927ee0575cb785f7ee6 10.0.0.5:6379
slots:12288-16383 (4096 slots) master
1 additional replica(s)
S: 105f108d1bf695166ba6d48a0d0fd060b0cb782d 10.0.0.9:6379
slots: (0 slots) slave
replicates 4744ffb2731aa838bca97927ee0575cb785f7ee6
[OK] All nodes agree about slots configuration.
>>> Check for open slots...
>>> Check slots coverage...
[OK] All 16384 slots covered.
How many slots do you want to move (from 1 to 16384)? 1365                  # 分配 1365个槽
What is the receiving node ID? 4744ffb2731aa838bca97927ee0575cb785f7ee6   # 接收的master是 10.0.0.5 的id
Please enter all the source node IDs.
Type 'all' to use all the nodes as source nodes for the hash slots.
Type 'done' once you entered all the source nodes IDs.
Source node #1:8fabdaedcc475069d78f31bb05ee0c954b3daf7f               #  从10.0.0.13中分配出去
Source node #2:done
 
```
 
- 现在已经转移出去 1365 个槽，剩下的继续上面的操作，直至全部迁移出去，再执行删除操作
```
[root@772b766adefb /]# redis-trib.rb del-node 10.0.0.3:6379 8fabdaedcc475069d78f31bb05ee0c954b3daf7f
>>> Removing node 8fabdaedcc475069d78f31bb05ee0c954b3daf7f from cluster 10.0.0.3:6379
>>> Sending CLUSTER FORGET messages to the cluster...
>>> SHUTDOWN the node.
```