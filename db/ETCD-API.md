# ETCD API
## 核心API
### KV操作
```
type KV interface {
    // 存放.
    Put(ctx context.Context, key, val string, opts ...OpOption) (*PutResponse, error)
    // 获取.
    Get(ctx context.Context, key string, opts ...OpOption) (*GetResponse, error)
    // 删除.
    Delete(ctx context.Context, key string, opts ...OpOption) (*DeleteResponse, error)
    // 压缩rev指定版本之前的历史数据.
    Compact(ctx context.Context, rev int64, opts ...CompactOption) (*CompactResponse, error)
    // 通用的操作执行命令，可用于操作集合的遍历。Put/Get/Delete也是基于Do.
    Do(ctx context.Context, op Op) (OpResponse, error)
    // 创建一个事务，只支持If/Then/Else/Commit操作.
    Txn(ctx context.Context) Txn
}
```


### Watch观察者模式，监听数据变化
```
type Watcher interface {
    // 监视key的变化，返回变化的结果
    Watch(ctx context.Context, key string, opts ...OpOption) WatchChan
    // 关闭所有监视器
    Close() error
}
```

### Lease租约操作
```
type Lease interface {
    // 分配一个租约.
    Grant(ctx context.Context, ttl int64) (*LeaseGrantResponse, error)
    // 释放一个租约.
    Revoke(ctx context.Context, id LeaseID) (*LeaseRevokeResponse, error)
    // 获取剩余TTL时间.
    TimeToLive(ctx context.Context, id LeaseID, opts ...LeaseOption) (*LeaseTimeToLiveResponse, error)
    // 获取所有租约.
    Leases(ctx context.Context) (*LeaseLeasesResponse, error)
    // 续约保持激活状态.
    KeepAlive(ctx context.Context, id LeaseID) (<-chan *LeaseKeepAliveResponse, error)
    // 仅续约激活一次.
    KeepAliveOnce(ctx context.Context, id LeaseID) (*LeaseKeepAliveResponse, error)
    // 关闭续约激活的功能.
    Close() error
}
```

### Cluster集群管理相关操作
```
type Cluster interface {
    // 列出集群所有成员.
    MemberList(ctx context.Context) (*MemberListResponse, error)
    // 添加新成员到集群中.
    MemberAdd(ctx context.Context, peerAddrs []string) (*MemberAddResponse, error)
    // 移除一个集群中的成员.
    MemberRemove(ctx context.Context, id uint64) (*MemberRemoveResponse, error)
    // 更新一个集群成员的地址.
    MemberUpdate(ctx context.Context, id uint64, peerAddrs []string) (*MemberUpdateResponse, error)        
}
```

## 并发API
### Lock分布式锁
```
// proto定义的接口
type LockServer interface {
    // 获取一个锁.
    Lock(context.Context, *LockRequest) (*LockResponse, error)
    // 释放当前持有的锁.
    Unlock(context.Context, *UnlockRequest) (*UnlockResponse, error)
}

// 已提供的实现
type lockServer struct {
    c *clientv3.Client
}
```


### Election选举
```
// proto定义的接口
type ElectionClient interface {
    // 竞选.
    Campaign(ctx context.Context, in *CampaignRequest, opts ...grpc.CallOption) (*CampaignResponse, error)
    // 公告更新新的领导值.
    Proclaim(ctx context.Context, in *ProclaimRequest, opts ...grpc.CallOption) (*ProclaimResponse, error)
    // 返回最新的公告.
    Leader(ctx context.Context, in *LeaderRequest, opts ...grpc.CallOption) (*LeaderResponse, error)
    // 流式返回多个公告.
    Observe(ctx context.Context, in *LeaderRequest, opts ...grpc.CallOption) (Election_ObserveClient, error)
    // 退选领导地位.
    Resign(ctx context.Context, in *ResignRequest, opts ...grpc.CallOption) (*ResignResponse, error)
}

// 已提供的实现
type electionServer struct {
    c *clientv3.Client
}
```