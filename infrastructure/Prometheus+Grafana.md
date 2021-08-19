# Prometheus+Grafana
> Prometheus 是一个开源的服务监控系统和时间序列数据库。  

特性：

- 高维度数据模型
- 自定义查询语言
- 可视化数据展示
- 高效的存储策略
- 易于运维
- 提供各种客户端开发库
- 警告和报警
- 数据导出

首先，在创造上的主机文件系统的最小Prometheus配置文件prometheus.yml （替换你要监控的IP地址）：

```
global:
  scrape_interval:     60s
  evaluation_interval: 60s

scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ['localhost:9090']
        labels:
          instance: prometheus

  - job_name: linux
    static_configs:
      - targets: ['10.10.0.186:9100']
        labels:
          instance: db1

  - job_name: mysql
    static_configs:
      - targets: ['10.10.0.186:9104']
        labels:
          instance: db1
```
10.10.0.186是我们数据库主机的IP，端口则是对应的exporter的监听端口。

## 启动Prometheus

```
docker run -d \
  -p 9090:9090 \
  --name prometheus \
  -v ~/prometheus.yml:/etc/prometheus/prometheus.yml \
  quay.io/prometheus/prometheus \
```

Prometheus内置了一个web界面，我们可通过http://monitor_host:9090进行访问：


> 在Status->Targets页面下，我们可以看到我们配置的两个Target，它们的State为UNKNOW（测试下来这个Docker镜像有问题）。  


下一步我们需要安装并运行exporter，在被监控端服务器安装Docker。
- 安装运行node_exporter
```
docker run -d \
  -p 9100:9100 \
  --name node-exporter \
  -v "/proc:/host/proc" \
  -v "/sys:/host/sys" \
  -v "/:/rootfs" \
  --net="host" \
  quay.io/prometheus/node-exporter
```
 
- 安装运行mysqld_exporter
mysqld_exporter需要连接到MySQL，所以需要MySQL的权限，我们先为它创建用户并赋予所需的权限。
```
mysql> CREATE USER 'mysql_monitor'@'localhost' IDENTIFIED BY 'mysql_monitor';
mysql> GRANT PROCESS, REPLICATION CLIENT, SELECT ON *.* TO 'mysql_monitor'@'localhost';
mysql> FLUSH PRIVILEGES;
```
启动Docker并传入用户名和密码以及主机IP和端口。
```
docker run -d \
  -p 9104:9104 \
  -e DATA_SOURCE_NAME="mysql_monitor:mysql_monitor@(10.10.0.186:3306)/" prom/mysqld-exporter
```
我们再次回到Status->Targets页面，可以看到两个Target的状态已经变成UP了。

接下来就是加入Grafana作为Prometheus的Dashboard

## 安装运行Grafana
```
docker run -d \
  -p 3000:3000 \
  -e "GF_SECURITY_ADMIN_PASSWORD=admin" \
  -v ~/grafana_db:/var/lib/grafana grafana/grafana
```
我们可通过http://monitor_host:3000访问Grafana网页界面（缺省的帐号/密码为admin/admin）

