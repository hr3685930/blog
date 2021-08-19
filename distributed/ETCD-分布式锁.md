# ETCD-分布式锁
> 我们希望同一时间只有一个线程能够访问到资源，但是分布式资源点之间的协调会非常麻烦，这个时候我们就需要一个分布式锁。线程先要获取到分布式锁，然后才能去操作资源。  

1. 利用租约在etcd集群中创建一个key，这个key有两种形态，存在和不存在，而这两种形态就是互斥量。
2. 如果这个key不存在，那么线程创建key，成功则获取到锁，该key就为存在状态。
3. 如果该key已经存在，那么线程就不能创建key，则获取锁失败。
4. 形象的解释一下，将key的存在和不存在两种状态比作一个int变量，存在为1，不存在为0. 如果线程去操作资源点时，先要去访问该变量。如果该变量为0,则线程有资格去访问资源，还要将int变为1，结束时将变量改变为0；如果变量为1，则没有资格去访问资源只能等待变量为0.

## 安装ETCD
```
version: ‘2’

networks:
  byfn:

services:
  etcd1:
    image: registry.cn-hangzhou.aliyuncs.com/coreos_etcd/etcd:v3
    container_name: etcd1
    command: etcd -name etcd1 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    networks:
      - byfn

  etcd2:
    image: registry.cn-hangzhou.aliyuncs.com/coreos_etcd/etcd:v3
    container_name: etcd2
    command: etcd -name etcd2 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    networks:
      - byfn

  etcd3:
    image: registry.cn-hangzhou.aliyuncs.com/coreos_etcd/etcd:v3
    container_name: etcd3
    command: etcd -name etcd3 -advertise-client-urls http://0.0.0.0:2379 -listen-client-urls http://0.0.0.0:2379 -listen-peer-urls http://0.0.0.0:2380 -initial-cluster-token etcd-cluster -initial-cluster "etcd1=http://etcd1:2380,etcd2=http://etcd2:2380,etcd3=http://etcd3:2380" -initial-cluster-state new
    ports:
      - 2379
      - 2380
    networks:
      - byfn

```


```
package main

import (
	“context”
	“fmt”
	“time”

	“github.com/coreos/etcd/clientv3”
	“github.com/coreos/etcd/clientv3/concurrency”
)

var (
	defaultTimeout = 10 * time.Second
	Client         *clientv3.Client
)

func init() {
	endpoints := []string{
		“127.0.0.1:32773”,
	}

	cfg := clientv3.Config{
		Endpoints:   endpoints,
		DialTimeout: time.Second * 5,
	}

	Client, _ = clientv3.New(cfg)
	// defer Client.Close() // make sure to close the client
}

func main() {
	m, err := GetLockSession("hello", "boy")
	if err != nil {
		fmt.Println("m get etcd lock failed:", err)
		return
	}

	// 添加带超时时间的ctx
	ctx, cancel := context.WithTimeout(context.Background(), 15*time.Second)
	defer cancel()

	err = m.Lock(ctx)
	if err != nil {
		fmt.Println(“m lock failed:”, err)
		return
	}

	fmt.Println(“m lock …”)

	go func() {
		select {
		case <-ctx.Done():
			err = m.Unlock(context.TODO())
			if err != nil {
				fmt.Println(“unlock:”, err)
				return
			}
			fmt.Println("m unlock ...")
		}
	}()

	for i := 1; i < 8; i++ {
		time.Sleep(1 * time.Second)
		fmt.Println("sleep ", i)
	}
}

func GetLockSession(key string, id string) (*concurrency.Mutex, error) {
	def := "/com/lock"
	lockKey := fmt.Sprintf("%s/%s-%s", def, key, id)
	session, err := concurrency.NewSession(Client)
	if err != nil {
		return nil, fmt.Errorf("get lock session %v", err)
	}

	return concurrency.NewMutex(session, lockKey), nil
}


```


