# redis-分布式锁

## 为什么需要分布式锁
> 随着互联网的兴起，现代软件发生了翻天覆地的变化，以前单机的程序，已经支撑不了现代的业务。无论是在抗压，还是在高可用等方面都需要多台计算机协同工作来解决问题。现代的互联网系统都是分布式部署的，分布式部署确实能带来性能和效率上的提升，但为此，我们就需要多解决一个分布式环境下，数据一致性的问题。  
> 当某个资源在多系统之间共享的时候，为了保证大家访问这个资源数据是一致的，那么就必须要求在同一时刻只能被一个客户端处理，不能并发的执行，否者就会出现同一时刻有人写有人读，大家访问到的数据就不一致了。  
> 在分布式系统的时代，传统线程之间的锁机制，就没作用了，系统会有多份并且部署在不同的机器上，这些资源已经不是在线程之间共享了，而是属于进程（服务器）之间共享的资源。  
> 因此，为了解决这个问题，我们就必须引入「分布式锁」。分布式锁，是指在分布式的部署环境下，通过锁机制来让多客户端互斥的对共享资源进行访问。分布式锁的特点如下：  
1. 互斥性
和我们本地锁一样互斥性是最基本，但是分布式锁需要保证在不同节点的不同线程的互斥。
2. 可重入性
同一个节点上的同一个线程如果获取了锁之后那么也可以再次获取这个锁。
3. 锁超时
和本地锁一样支持锁超时，防止死锁。
4. 高效，高可用
加锁和解锁需要高效，同时也需要保证高可用防止分布式锁失效，可以增加降级。
5. 支持阻塞和非阻塞
和 ReentrantLock 一样支持 lock 和 trylock 以及 tryLock(long timeOut)。
## 基于redis分布式锁
> 如果你通过网络搜索分布式锁，最多的就是基于redis的了。基于redis的分布式锁得益于redis的单线程执行机制，单线程在执行上就保证了指令的顺序化，所以很大程度上降低了开发人员的思考设计成本。但是，基于redis做分布式锁难道真的这么容易吗？  
1. 原子操作
基于redis的分布式锁常用命令是
SETNX key value
只在键 key 不存在的情况下，将键 key的值设置为value 。若键key 已经存在， 则SETNX 命令不做任何动作。SETNX 是『SET if Not eXists』(如果不存在，则 SET)的简写。代码示例：
![](redis-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81/dc54564e9258d1095fcf2526e8270bbb6d814dfe.jpeg)
成功获取到锁之后，然后设置一个过期时间(这里避免了客户端down掉，锁得不到释放的问题)
redis> expire redislock 5
成功拿到锁的客户端顺利进行自己的业务，业务代码执行完，然后再删除该key
redis> DEL redislock
如果一切都像想象的那么顺利，程序员TMD就不用996了。假如客户端拿到锁之后，执行设置超时指令之前down掉了（现实总是那么悲剧），那这个锁就永远都释放不了.也许你会想到用 Redis 事务来解决。但是这里不行，因为 expire 是依赖于 setnx 的执行结果的，如果 setnx 没抢到锁，expire 是不应该执行的。事务里没有 if-else 分支逻辑，事务的特点是一口气执行，要么全部执行要么一个都不执行。公司几个亿的业务又被你耽误了…
以上情况的出现是因为两个命令并非一个原子性操作，所以在redis 2.8 版本之后出现了新的命令
SETEX key seconds value
所以现在可以利用一条原子性操作的命令来获取锁
![](redis-%E5%88%86%E5%B8%83%E5%BC%8F%E9%94%81/0823dd54564e9258d6442ebca5fd165ccdbf4ef7.jpeg)
2. 超时问题
在正常的业务当中，当一个线程获取到锁并且设置了锁的过期时间之后，会出现由于业务代码执行时间过长，锁由于到达超时时间自动释放的情况。自动释放之后，其他的线程就会获取到分布式锁，导致业务代码不会串行执行。如果业务上允许这样的情况偶尔发生，那程序员就开干吧，最后顶多人工干预一下，update 一下数据库。
为了避免这类情况发生，在使用redis分布式锁的时候，业务方应尽量避免长时间执行的代码任务。
如果设置锁的超时时间比较长，在一定程度上可以缓解业务代码执行时间长锁自动到期的问题，但是一旦业务代码down掉，其他等待锁的线程等待的时间会比较长，这种情况下，确保获取到锁的程序不会down 成为了主要问题。
3. 获取锁失败
当锁被一个调用方获取之后，其他调用方在获取锁失败之后，是继续轮询还是直接业务失败呢？如果是继续轮询的话，同步情况下当前线程会一直处于阻塞状态，所以这里轮询的情况还是建议使用异步。
4. 可重入性
可重入性是指已经拥有锁的客户端再次请求加锁，如果锁支持同一个客户端重复加锁，那么这个锁就是可重入的。如果基于redis的分布式锁要想支持可重入性，需要客户端封装，可以使用threadlocal存储持有锁的信息。这个封装过程会增加代码的复杂度，所以菜菜不推荐这样做。
5. redis挂了
	- 在Redis的master节点上拿到了锁；
	- 但是这个加锁的key还没有同步到slave节点；
	- master故障，发生故障转移，slave节点升级为master节点；
	- 导致锁丢失。
> 当且仅当从大多数（N/2+1，这里是3个节点）的Redis节点都取到锁，并且使用的时间小于锁失效时间时，锁才算获取成功。  
6. 时钟跳跃问题
在某些时候，redis的服务器时间发生的跳跃，由于锁的过期时间依赖于服务器时间，所以也会出现两个客户端同时获取到锁的情况发生。

7. 主从切换可能丢失锁信息
考虑一下这样的场景：在分布式环境中，很多并发需要锁来同步，当使用redis分布式锁，通用的做法是使用redis的setnx key value px 这样的命令，设置一个字段，当设置成功说明获取锁，设置不成功说明锁被占用，当获取锁之后需要删除锁，也就是删除设置的锁字段，这是锁可以被其他占用。

这里在主从切换回出现问题，当第一个线程在主服务器上设置了锁，但是这时候从服务器并没有及时同步主服务器的状态，也就是没有同步主服务器中的锁字段，而此时，主服务器挂了，redis的哨兵模式升级从服务器为主服务器，如果在并发量大的情况下，虽然第一个线程获取了锁，其他线程会在当前的主服务器（之前的从服务器，但是并没有同步已经设置的锁字段）上设置锁字段，这样并不能保证锁的互斥性。

8. 缓存易失性
假如第一个线程设置了锁，但是之后触发内存淘汰机制很不幸淘汰了设置的锁字段，接下来的线程在第一个线程没有释放锁的情况下，也是重新设置锁字段的，这样并不能保证锁的安全性。
