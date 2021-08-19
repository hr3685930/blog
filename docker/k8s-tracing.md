# k8s-链路(tracing)
## 链路
> Tracing - 用于记录请求范围内的信息.例如,一次远程方法调用的执行过程和耗时.它是我们排查系统性能问题的利器  

### OpenTracing
#### OpenTracing是什么？
1. 后台无关的一套接口:
> 被跟踪的服务只需要调用这套接口,就可以被任何实现这套接口的跟踪后台（比如Zipkin, Jaeger等等）支持,而作为一个跟踪后台,只要实现了个这套接口,就可以跟踪到任何调用这套接口的服务  

2. 标准化了对跟踪最小单位Span的管理:
> 定义了开始Span,结束Span和记录Span耗时的API.Span的定义可以参照 [开源分布式跟踪系统Zipkin介绍（架构篇）](https://zhuanlan.zhihu.com/p/41478682)   

3. 标准化了进程间跟踪数据传递的方式:
> 定义了一套API方便跟踪数据的传递  

4. 标准化了进程内当前Span的管理:
> 定义了存储和获取当前Span的API  

#### 相关组件
1. Span:
> 表示分布式调用链条中的一个调用单元,比方说某个http调用的服务提供方,他的边界包含一个请求进到服务内部再由某种途径(http等)从当前服务出去.一个span一般会记录这个调用单元内部的一些信息,例如:  
- 日志信息
- 标签信息
- 开始/结束时间

2. SpanContext
> 表示一个span对应的上下文,span和spanContext基本上是一一对应的关系,上下文存储的是一些需要跨越边界的一些信息,例如:  
- spanId 当前这个span的id
- traceId 这个span所属的traceId(也就是这次调用链的唯一id)
- baggage 其他的能过跨越多个调用单元的信息 
这个SpanContext可以通过某些媒介和方式传递给调用链的下游来做一些处理（例如子Span的id生成、信息的继承打印日志等等）

3. Tracer
> tracer表示的是一个通用的接口,它相当于是opentracing标准的枢纽,它有以下的职责:  
-  建立和开启一个span
- 从某种媒介中提取和注入一个spanContext

4. Operation Names
> 每一个span都有一个操作名称,这个名称简单,并具有可读性高.（例如：一个RPC方法的名称,一个函数名,或者一个大型计算过程中的子任务或阶段）.span的操作名应该是一个抽象、通用的标识,能够明确的、具有统计意义的名称  

5. Inter-Span References
> 一个span可以和一个或者多个span间存在因果关系.OpenTracing定义了两种关系：ChildOf 和 FollowsFrom.这两种引用类型代表了子节点和父节点间的直接因果关系.未来,OpenTracing将支持非因果关系的span引用关系.(例如:多个span被批量处理,span在同一个队列中等等)  

6. Logs
> 每个span可以进行多次Logs操作,每一次Logs操作,都需要一个带时间戳的时间名称,以及可选的任意大小的存储结构.标准中定义了一些日志（logging）操作的一些常见用例和相关的log事件的键值，可参考 [Data Conventions Guidelines 数据约定指南](https://wu-sheng.gitbooks.io/opentracing-io/content/pages/api/data-conventions.html)   

7. Tags
> 每个span可以有多个键值对（key:value）形式的Tags,Tags是没有时间戳的,支持简单的对span进行注解和补充.和使用Logs的场景一样,对于应用程序特定场景已知的键值对Tags,tracer可以对他们特别关注一下.更多信息,可参考 [Data Conventions Guidelines 数据约定指南](https://wu-sheng.gitbooks.io/opentracing-io/content/pages/api/data-conventions.html)   

8. Baggage
> Baggage是存储在SpanContext中的一个键值对(SpanContext)集合.它会在一条追踪链路上的所有span内全局传输,包含这些span对应的SpanContexts.在这种情况下.”Baggage”会随着trace一同传播,他因此得名（Baggage可理解为随着trace运行过程传送的行李）.鉴于全栈OpenTracing集成的需要,Baggage通过透明化的传输任意应用程序的数据,实现强大的功能.例如:可以在最终用户的手机端添加一个Baggage元素,并通过分布式追踪系统传递到存储层,然后再通过反向构建调用栈,定位过程中消耗很大的SQL查询语句.  

9. Baggage vs. Span Tags
* Baggage在全局范围内,（伴随业务系统的调用）跨进程传输数据.Span的tag不会进行传输,因为他们不会被子级的span继承.
* span的tag可以用来记录业务相关的数据,并存储于追踪系统中.实现OpenTracing时,可以选择是否存储Baggage中的非业务数据,OpenTracing标准不强制要求实现此特性.

10. Inject and Extract
> SpanContexts可以通过Injected操作向Carrier增加,或者通过Extracted从Carrier中获取,跨进程通讯数据（例如:HTTP头）.通过这种方式,SpanContexts可以跨越进程边界,并提供足够的信息来建立跨进程的span间关系(因此可以实现跨进程连续追踪).  






