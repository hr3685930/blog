# Apache Flink

> Flink是Apache下的一款分布式的计算引擎，它的亮点在于处理实时数据流（无界数据流），实时地产生数据的结果；当然，通过划分窗口（时间窗口等）同样适用于批处理（有界数据流）。想想Spark streaming也可以处理实时数据呀，那为什么会诞生flink呢？flink与spark相比有哪些特色？下文将逐个介绍这些内容。  

## 安装
- 使用brew安装
```
brew install apache-flink
```

- 使用docker compose安装
```
version: “2.1”
services:
jobmanager:
    image: ${FLINK_DOCKER_IMAGE_NAME:-flink}
    expose:
    - “6123”
    ports:
    - “8081:8081”
    command: jobmanager
    environment:
    - JOB_MANAGER_RPC_ADDRESS=jobmanager

taskmanager:
    image: ${FLINK_DOCKER_IMAGE_NAME:-flink}
    expose:
    - "6121"
    - "6122"
    depends_on:
    - jobmanager
    command: taskmanager
    links:
    - "jobmanager:jobmanager"
    environment:
    - JOB_MANAGER_RPC_ADDRESS=jobmanager

```