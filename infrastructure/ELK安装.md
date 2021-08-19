# ELK安装
> ELK是开源日志界的三大剑客，本文主要讲怎么在docker里头跑起来这一套东东。  

## 镜像
这里采用https://github.com/deviantony/docker-elk的镜像。

## 安装
> cd docker-elk  
> docker-compose up -d  

## 查看kibana
http://localhost:5601/

## 查看sense
http://localhost:5601/app/sense

## 默认端口
> 5000: Logstash TCP input.  
> 9200: Elasticsearch HTTP  
> 9300: Elasticsearch TCP transport  
> 5601: Kibana  

## 测试
> nc localhost 5000 < /path/to/logfile.log  