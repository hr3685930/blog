# mysql慢查询
## 查看当前 慢查日志 状态
```
show variables like '%slow%';
#主要关注两个变量：
slow_query_log  #是否启用
slow_query_log_file  #日志文件

#慢查的时间阈值
show variables like 'long_query_time';
```

## 配置启用 慢查日志
```
## vim my.cnf：
slow_query_log=1
slow_query_log_file=/var/log/mysql/slow.log  #需要新建该文件，并注意该文件的读写权限
long_query_time = 2

## 可以在当前连接切换配置， Mysql Shell下执行：
set long_query_time = 5  #临时调整当前连接慢查询的阈值，不影响其他连接阈值
set global slow_query_log=0  #全局关闭 慢查日志。(mysql强制此为全局配置)
                             #关闭状态持续至手动切回开启
```

## 查看 慢查日志

直接查看日志文件的方式会比较费力， 好在mysql提供了简单的日志分析工具 mysqldumpslow
```
mysqldumpslow 【选项】 log_file

选项：
-s order  #以什么排序 (al, at, ar, ae, c, l, r, e, t)
          #at(平均查询时间，默认)、 al(平均上锁时间)、 ar(平均传送行数)
          #c(记录统计)、 l(上锁时间)、 r(传送行数)、 t(查询时间)
-r  #倒序
-t  #显示几条慢查询
-g pattern  #搜索关键字
```