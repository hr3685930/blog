# mysql 高可用集群galera cluster
> 集成了Galera插件的MySQL集群，是一种新型的，数据不共享的，高度冗余的高可用方案，目前Galera Cluster有两个版本，分别是Percona Xtradb Cluster及MariaDB Cluster，Galera本身是具有多主特性的，即采用multi-master的集群架构，是一个既稳健，又在数据一致性、完整性及高性能方面有出色表现的高可用解决方案。  
![](mysql%E9%AB%98%E5%8F%AF%E7%94%A8%E9%9B%86%E7%BE%A4galeracluster/289fa0a255559098dae7fb4354d52417.png)

## 特点
* 多主架构：真正的多点读写的集群，在任何时候读写数据，都是最新的。
* 同步复制：集群不同节点之间数据同步，没有延迟，在数据库挂掉之后，数据不会丢失
* 并发复制：从节点APPLY数据时，支持并行执行，更好的性能
* 故障切换：在出现数据库故障时，因支持多点写入，切换容易
* 热插拔：在服务期间，如果数据库挂了，只要监控程序发现的够快，不可服务时间就会非常少。在节点故障期间，节点本身对集群的影响非常小
* 自动节点克隆：在新增节点，或者停机维护时，增量数据或者基础数据不需要人工手动备份提供，Galera Cluster会自动拉取在线节点数据，最终集群会变为一致
* 对应用透明：集群的维护，对应用程序是透明的


## 原理
> 当一个事务在当前写入的节点提交后，通过wsrep API（write set replication API）将这个事务变成写集(write set)广播到同集群的其他节点中，其他节点收到写集事务后，对这个事务进行可行性检查，并返回结果给wsrep API。若大多数节点都预估自己可以成功执行这个事务，则wsrep API会做出仲裁，通知所有可以成功执行这个事务的节点提交这个事务，并将事务成功提交的消息返回给客户端，同时根据需要剔除没有成功执行事务的节点  

## 优势
- 与异步复制相比：
数据一致性强，传统异步复制并不能保证主从数据一致性，这是由于一般情况下，主库多线程并发执行事务，但从库却只有一个线程重做事务，在高压力情况下必然会导致主从延迟。
- 与使用半同步复制或分布式锁实现的同步复制相比：
性能高，扩展性好，半同步复制在高负载甚至从库性能较差的情况下，难以保证其性能。即使自动的从半同步复制切换到异步复制，也会牺牲其最大的优点：一致性。其扩展友好度也较差
- galera集群的独特优势：
	1. 同步复制，主备无延迟
	2. 支持多主同时读写，保证数据一致性
	3. 集群中各节点保存全量数据
	4. 节点添加/删除，自动检测和配置
	5. 行级别并行复制
	6. 不需要写binlog
## 缺点
DDL操作会严重阻塞同步线程，线上大动作DDL会导致有可能导致节点堵塞无响应，更进一步会导致部分节点下线。实际使用中需要搭配pt-osc或者gh-osc等在线DDL工具来进行操作DDL。



## 安装(docker-compose)
```
version: “2”
services:

  node1:
    image: percona/percona-xtradb-cluster
    container_name: node1
    restart: always
    networks:
      net1:
        ipv4_address: 10.5.0.10
    ports:
      - “3305:3306”
    volumes:
      - v1:/mysql-cluster
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - CLUSTER_NAME=mysql-cluster
      - XTRABACKUP_PASSWORD=123456
  node2:
    image: percona/percona-xtradb-cluster
    container_name: node2
    restart: always
    networks:
      net1:
        ipv4_address: 10.5.0.11
    ports:
      - “3304:3306”
    volumes:
      - v2:/mysql-cluster
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - CLUSTER_NAME=mysql-cluster
      - XTRABACKUP_PASSWORD=123456
      - CLUSTER_JOIN=node1

  node3:
    image: percona/percona-xtradb-cluster
    container_name: node3
    restart: always
    networks:
      net1:
        ipv4_address: 10.5.0.12
    ports:
      - “3303:3306”
    volumes:
      - v3:/mysql-cluster
    environment:
      - MYSQL_ROOT_PASSWORD=123456
      - CLUSTER_NAME=mysql-cluster
      - XTRABACKUP_PASSWORD=123456
      - CLUSTER_JOIN=node1

volumes:
  v1: {}
  v2: {}
  v3: {}

networks:
  net1:
    driver: bridge
    ipam:
      config:
        - subnet: 10.5.0.0/16

```

### 安装注意事项
> 先注释掉node2和node3服务等待node1起来后在运行  
> 当node1节点下线后重启需要配置  
```
vim /etc/my.cnf
wsrep_cluster_address=‘gcomm://xxx.xxx.xxx.xxx;xxx.xx.xx.xx'
```
> 当停止所有集群后 最后停止的要最先启动否则可能导致数据丢失  