# mysql二进制日志-binlog 

## 配置 my.cnf
```
log_bin = /var/log/mysql-bin.log

## 可选配置
binlog_format = MIXED  #默认statement， 推荐mixed
expire_logs_days = 7
max_binlog_size = 100m
```

## Binlog三种格式

1. 基于语句 statement  

    记录数据变化的sql语句。一般日志量较小，但是语句执行的可靠性较低。

2. 基于行 row  

    记录被修改的数据行。数据细节清晰可靠，但一般日志量较大。

3. 混合模式mixed  

    以上两种格式的合并。

## Binlog查看
* mysql-bin.index     索引文件， 内容为所有日志文件名， 一行一个
* mysql-bin.000001  具体日志文件， 每次mysql服务重启都会新增一个日志文件， 序列号增一。

```
Mysql Shell下查看：
show binary logs  #查看binlog列表
show master status   #查看当前写入的binlog文件
                     #（每次服务重启都会新建一个binlog，具体运行时往最近创建的binlog写入）
show binlog evnets 【in 'log_name'】  #查看指定binlog， 没有指定则默认第一个。注意有单引号。


Bash下查看， 更细致的分析：
##如果binlog格式是行模式的,请加 -vv参数
mysqlbinlog --start-datetime='2015-01-01 00:00:00' --stop-datetime='2015-06-01 01:01:01' 【-d 库名】 二进制文件   #时间端内查看
mysqlbinlog --start-postion=1                      --stop-position=1000                  【-d 库名】 二进制文件   #Event Pos 区间内查看
```

## Binlog恢复mysql的更新
如果数据库被各种原因误删， 则我们可以结合 binlog 及 上次数据库的备份 来恢复
```
#先导入备份的数据库，然后执行以下命令
#注意：如果是从上次备份来恢复， 则 mysqlbinlog 还需指定上次备份的时间 --start-date
mysqlbinlog -d DatabaseName binlog-file|mysql -u user -p
```