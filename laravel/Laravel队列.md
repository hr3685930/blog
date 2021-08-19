# Laravel队列

### laravel 队列服务的任务调度
队列服务的任务调度过程如下：

Laravel 的队列服务由两个进程控制，一个是生产者，一个是消费者。这两个进程操纵了 redis 三个队列，其中一个 List，负责即时任务，两个 Zset，负责延时任务与待处理任务。
生产者负责向 redis 推送任务，如果是即时任务，默认就会向 queue:default 推送；如果是延时任务，就会向 queue:default:delayed 推送。
消费者轮询两个队列，不断的从队列中取出任务，先把任务放入 queue:default:reserved 中，再执行相关任务。如果任务执行成功，就会删除 queue:default:reserved 中的任务，否则会被重新放入 queue:default:delayed 队列中。
