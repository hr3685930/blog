# Docker Stack
- yaml
```
version: ‘3’

services:
  MyService:
    image: my_image:tag
    container_name: xxx
    hostname: yyy
    restart: always
    env_file: .env
    environment:
	  - key=val
    deploy:
      replicas: 5 #service启用副本数
      resources:
        limits:
          Cpus: 0.1 #每个节点cpu资源占比
          memory: 50M #每个节点内存资源限额
      restart_policy:
        condition: on-failure
      placement:
        constraints: [node.role == manager]
    ports:
      - ‘xxx:yyy’
    networks:
      - MyNetwork #service使用的负载均衡网络（要求overlay网络）
    volumes:
      - xxx:yyy
    depends_on:
      - OtherService

Networks: #网络自动负载均衡
  MyNetwork:

```


```
docker stack deploy -c docker-compose.yml StackName #发布Service Stack（每次执行都会自动热更新stack）

docker stack services StackName #罗列stack下的service
docker stack ps StackName #罗列stack下的task
docker service ls #服务列表
docker service ps 服务名称 #罗列服务下的task（服务包含的容器节点称作task）
docker stack rm StackName #下线Service Stack
```


Docker更改为OverlayFS存储驱动