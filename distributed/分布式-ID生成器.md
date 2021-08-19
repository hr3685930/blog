# 分布式-ID生成器
> 有时我们需要能够生成类似MySQL自增ID这样不断增大，同时又不会重复的id。以支持业务中的高并发场景。比较典型的，电商促销时，短时间内会有大量的订单涌入到系统，比如每秒10w+。明星出轨时，会有大量热情的粉丝发微博以表心意，同样会在短时间内产生大量的消息。  


## Twitter的snowflake
![](%E5%88%86%E5%B8%83%E5%BC%8F-ID%E7%94%9F%E6%88%90%E5%99%A8/ch6-snowflake-easy.png)
```
package main

import (
	“fmt”
	“os”

	“github.com/bwmarrin/snowflake”
)

func main() {
	n, err := snowflake.NewNode(1)
	if err != nil {
		println(err)
		os.Exit(1)
	}

	for i := 0; i < 3; i++ {
		id := n.Generate()
		fmt.Println(“id”, id)
	}
}
```
可定制字段
```
   // Epoch is set to the twitter snowflake epoch of Nov 04 2010 01:42:54 UTC
    // You may customize this to set a different epoch for your application.
    Epoch int64 = 1288834974657

    // Number of bits to use for Node
    // Remember, you have a total 22 bits to share between Node/Step
    NodeBits uint8 = 10

    // Number of bits to use for Step
    // Remember, you have a total 22 bits to share between Node/Step
    StepBits uint8 = 12
```
Epoch就是本节开头讲的起始时间，NodeBits指的是机器编号的位长，StepBits指的是自增序列的位长。

## Sony的sonyflake
![](%E5%88%86%E5%B8%83%E5%BC%8F-ID%E7%94%9F%E6%88%90%E5%99%A8/ch6-snoyflake.png)
> 这里的时间只用了39个bit，但时间的单位变成了10ms，所以理论上比41位表示的时间还要久(174年)。  
```
Sequence ID ## 和之前的定义一致，
Machine ID ## 其实就是节点id。
Sonyflake ## 与众不同的地方在于其在启动阶段的配置参数：
func NewSonyflake(st Settings) *Sonyflake
```
Settings ## 数据结构如下：
```
type Settings struct {
    StartTime      time.Time
    MachineID      func() (uint16, error)
    CheckMachineID func(uint16) bool
}

StartTime  ## 选项和我们之前的
Epoch  ## 差不多，如果不设置的话，默认是从2014-09-01 00:00:00 +0000 UTC开始。
MachineID  ## 可以由用户自定义的函数，如果用户不定义的话，会默认将本机IP的低16位作为
machine id  ## 。
CheckMachineID  ## 是由用户提供的检查
MachineID  ## 是否冲突的函数。这里的设计还是比较巧妙的，如果有另外的中心化存储并支持检查重复的存储，那我们就可以按照自己的想法随意定制这个检查
MachineID  ## 是否冲突的逻辑。如果公司有现成的Redis集群，那么我们可以很轻松地用Redis的集合类型来检查冲突。
redis 127.0.0.1:6379> SADD base64_encoding_of_last16bits MzI0Mgo=
(integer) 1
redis 127.0.0.1:6379> SADD base64_encoding_of_last16bits MzI0Mgo=
(integer) 0
```
使用起来也比较简单，有一些逻辑简单的函数就略去实现了
```
package main

import (
	“fmt”
	“os”
	“time”

	“github.com/sony/sonyflake”
)

func getMachineID() (uint16, error) {
	var machineID uint16
	var err error
	machineID = readMachineIDFromLocalFile()
	if machineID == 0 {
		machineID, err = generateMachineID()
		if err != nil {
			return 0, err
		}
	}

	return machineID, nil
}

func checkMachineID(machineID uint16) bool {
	saddResult, err := saddMachineIDToRedisSet()
	if err != nil || saddResult == 0 {
		return true
	}

	err := saveMachineIDToLocalFile(machineID)
	if err != nil {
		return true
	}

	return false
}

func main() {
	t, _ := time.Parse("2006-01-02", "2020-01-01")
	settings := sonyflake.Settings{
		StartTime:      t,
		MachineID:      getMachineID,
		CheckMachineID: checkMachineID,
	}

	sf := sonyflake.NewSonyflake(settings)
	id, err := sf.NextID()
	if err != nil {
		fmt.Println(err)
		os.Exit(1)
	}

	fmt.Println(id)
}
```
