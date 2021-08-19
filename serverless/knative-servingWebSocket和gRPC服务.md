# knative-serving(WebSocket和gRPC服务)
> 虽然说Knative默认就支持WebSocket和gRPC，但在使用中发现有些时候想要把自己的WebSocket或gRPC部署到Knative中还是会有各种不顺利的地方，尽管最后排查发现大多都是自己的程序问题或者是配置错误导致的。为了方便大 家做验证，这里就分别给出一个WebSocket的例子和一个gRPC的例子。当我们需要在生产或者测试环境部署相关服务的时候可以使用本文给出的示例进行Knative服务的测试。   

## WebSocket
如果自己手动的配置IstioGateway支持WebSocket就需要开启websocketUpgrade功能。但使用KnativeServing部署其实就自带了这个能力。本示例的完整代码放在 https://github.com/knative-sample/websocket-chat/tree/b1.0 ， 这是一个基于WebSocket实现的群聊的例子。使用浏览器连接到部署的服务中就可以看到一个接收信息的窗口和发送信息的窗口。当你发出一条信息以后所有连接进来的用户都能收到你的消息。所以你可以使用两个浏览器窗口分别连接到服务中，一个窗口发送消息一个窗口接收消息，以此来验证WebSocket服务是否正常。 
本示例是在gorilla/websocket基础之上进行了一些优化: 
* 代码中添加了vendor依赖，你下载下来就可以直接使用
* 添加了Dockerfile和Makefile可以直接编译二进制和制作镜像
* 添加了Knative Sevice的yaml文件 (service.yaml)，你可以直接提交到Knative集群中使用
* 也可以直接使用编译好的镜像registry.cn-hangzhou.aliyuncs.com/knative-sample/websocket-chat:2019-10-15

![](knative-servingWebSocket%E5%92%8CgRPC%E6%9C%8D%E5%8A%A1/ACB08C5D-A3B1-476D-B35B-3A520B9D694B.png)

![](knative-servingWebSocket%E5%92%8CgRPC%E6%9C%8D%E5%8A%A1/8A6E01B0-D9FF-4963-931F-2AE0C81F8EF5.png)
打开两个窗口，在其中一个窗口发送一条消息，另外一个窗口通过WebSocket也收到了这条消息。 

## gRPC 
gRPC不能通过浏览器直接访问，需要通过client端和server端进行交互。本示 例的完整代码放在https://github.com/knative-sample/grpc-ping-go/tree/b1.0，本示例会给一个可以直接使用的镜像，测试gRPC服务。 


![](knative-servingWebSocket%E5%92%8CgRPC%E6%9C%8D%E5%8A%A1/C162BE28-894D-4508-80B6-F5ACE022EDA1.png)
测试

![](knative-servingWebSocket%E5%92%8CgRPC%E6%9C%8D%E5%8A%A1/09012EC3-EB37-48F3-BAEE-2EDE16C6C6D7.png)

## 总结
WebSocket示例通过一个chat的方式展示发送和接受消息
gRPC通过启动一个client的方式展示gRPC远程调用的过程
