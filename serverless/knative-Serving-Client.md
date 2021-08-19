# knative-serving(Serving Client)
> 通过前面的一系列文章你已经知道如何基于kubectl来操作Knative的各种资源。但是如果想要在项目中集成Knative仅仅使用kubectl这种命令的方式是不够的。是需要在代码中基于Knative Serving SDK进行集成开发。本文就从 Knative Serving SDK入手，介绍如何基于KnativeSDK进行serverless开发。   

## Golang Context
在正式开始介绍Knative Serving SDK之前我们先简单的介绍一下Golang Context的机理，因为在Knative Serving中client、Informer的初始化和信息传 递完全是基于Golang Context实现的。 
Golang是从1.7版本开始引入的Context，Golang的Context可以很好的简化多个goroutine之间以及请求域间的数据传递、取消信号和截至时间等相关操 作。Context主要有两个作用:
1. 传输必要的数据
2. 进行协调控制，比如终止goroutine、设置超时时间等

###Context定义
Context本身是一个接口
```
type Context interface {
    Deadline() (deadline time.Time, ok bool) 
    Done() <-chan struct{}
    Err() error
    Value(key interface{}) interface{} 
}
```
- Deadline方法是获取设置的截止时间的意思，到了这个时间点，Context会自动发起取消请求 

- Done方法返回一个只读的chan，如果该方法返回的chan可以读取，则意味着parent Context已经发起了取消请求 ,此时应该应该做清理操作，然后退出goroutine并释放资源 

- Err方法返回取消的错误原因 

- Value方法获取该Context上绑定的值，是一个键值对。所以要通过一个Key才可以获取对应的值，这个值是线程安全的 
> 关于Context主要记住一点:可以通过Value传递数据，Value是一个键值对结构。   

## Knative Serving client源码浅析 
- 在Context的这些特性中，Knative Serving中重度依赖的是Value功能。 以Service的Informer初始化为例进行说明,[serving/service.go at v0.10.0 · knative/serving · GitHub](https://github.com/knative/serving/blob/v0.10.0/pkg/client/injection/informers/serving/v1alpha1/service/service.go#L31) 可以看到源码。 
Informer“构造函数”是在init函数中自动注册到injection Default中的。当Informer“构造函数”被调用之后会自动把生成的Informer注入到Context中 context.WithValue(ctx,Key{},inf),inf.Informer()。 
![](knative-servingServing%20Client/C6BB76D2-74A6-436A-88CF-DF0E0C2C49FC.png)
从上图中可以看到，Informer初始化的时候需要调用factory，而factory本身 是从Context中获取的。下面我发再看看factory是怎么初始化的。 
- factory的初始化 
(https://github.com/knative/serving/blob/v0.8.0/pkg/client/injection/informers/serving/factory/servingfactory.go#L31)
![](knative-servingServing%20Client/9E1CFC07-4159-4445-95CF-2559A3497E7B.png)

可以发现factory也是把“构造函数”注册到injectionDefault中，并且会把生成的SharedInformerFactory注入到Context中。 而且factory中使用的client(链接kube-apiserver使用的对象)也是从Context获取到的。 
可以说KnativeServingSDK初始化的过程是面向Context编程的。关键对象是自动注入到Context，在使用的时候从Context中取出。 
顺带提一点，KnativeServing的日志对象也是在Context保存的，当需要打印日志的时候先通过logger:=logging.FromContext(ctx) 从Context中拿到logger，然后就可以使用了。这样做的好处是可以通过管理logger对象，比如做trace功能。如下所示是基于logger打印出来的日志，可以看到对于同一个请求的处理是可以通过traceID进行追踪的。下面这段日志都是对577f8de5-cec9-4c17- 84f7-f08d39f40127这个trace的处理。 
```
{“level”:”info”,”ts”:”2019-08-28T20:24:39.871+0800”,”caller”:”controll er/service.go:67”,”msg”:”Reconcile: default/helloworld-go”,”knative.dev/ traceid”:”be5ec711-6ca3-493c-80ed-dddfa21fd576”,”knative.dev/key”:”default/ helloworld-go”} {“level”:”info”,”ts”:”2019-08-28T20:24:39.871+0800”,”caller”:” controller/controller.go:339”,”msg”:”Reconcile succeeded. Time 
taken: 487.347μs.”,”knative.dev/traceid”:”90653eda-644b-4b1e-8bdb- 4a1a7a7ff0d8”,”knative.dev/key”:”eci-test/helloworld-go”} {“level”:”info”,”ts”:”2019-08-28T20:24:39.871+0800”,”caller”:”contr oller/service.go:106”,”msg”:”service: default/helloworld-go route: 
default/helloworld-go “,”knative.dev/traceid”:”be5ec711-6ca3-493c-80ed- dddfa21fd576”,”knative.dev/key”:”default/helloworld-go”} {“level”:”info”,”ts”:”2019-08-28T20:24:39.872+0800”,”caller”:”controller/ service.go:67”,”msg”:”Reconcile: eci-test/helloworld-go”,”knative.dev/ traceid”:”22f6c77d-8365-4773-bd78-e011ccb2fa3d”,”knative.dev/key”:”eci-test/ helloworld-go”} {“level”:”info”,”ts”:”2019-08-28T20:24:39.872+0800”,”caller”:”controll er/service.go:116”,”msg”:”service: default/helloworld-go revisions: 1 “,”knative.dev/traceid”:”be5ec711-6ca3-493c-80ed-dddfa21fd576","knative.dev/ key":"default/helloworld-go"} {"level":"info","ts":"2019-08-28T20:24:39.872+0800","caller":"controller/ service.go:118”,”msg”:”service: default/helloworld-go revision: default/ helloworld-go-cgt65 “,”knative.dev/traceid”:”be5ec711-6ca3-493c-80ed- dddfa21fd576”,”knative.dev/key”:”default/helloworld-go”} {“level”:”info”,”ts”:”2019-08-28T20:24:39.872+0800”,”caller”:” controller/controller.go:339”,”msg”:”Reconcile succeeded. Time 
taken: 416.527μs.”,”knative.dev/traceid”:”be5ec711-6ca3-493c-80ed- dddfa21fd576”,”knative.dev/key”:”default/helloworld-go”} {“level”:”info”,”ts”:”2019-08-28T20:24:39.872+0800”,”caller”:”controller/ service.go:106”,”msg 

```

## 使用Knative Serving SDK 
介绍完Knative Serving client的初始化过程，下面我们看一下应该如何在代码 中用Knative Serving SDK进行编码。 
示例参见:(https://github.com/knative-sample/serving-controller/blob/b1.0/cmd/app/app.go) 
这个示例中首先使用配置初始化*zap.SugaredLogger 对象，然后基于ctx:= signals.NewContext()生成一个Context。signals.NewContext()作用是监听SIGINT信号，也就是处理CTRL+c指令。这里用到了Context接口的Done函数机制。 
### 在 Reconcile 中调用 Knative API
代码示例:(https://github.com/knative-sample/serving-controller/blob/v0.1/pkg/controller/service.go) 
另外除了文章中提到的示例代码，提供另外一个例子(https://github.com/knative-sample/serving-sdk-demo/tree/b1.0)这个例子是直接使用Knative SDK创建一个ksvc，这两个例子的侧重点有所不同。可以参考着看。 

## 总结
本文从KnativeServing client的初始化过程开始展开， 介绍了Knative informer的设计以及使用方法。通过本文你可以了解到: 
- Knative Servingclient的设计思路
- 如何基于Knative Serving SDK进行二次开发
- 通过Knative Serving学习到如何面向Context编程
- KnativeServing集成开发示例:(https://github.com/knative-sample/serving-controller)