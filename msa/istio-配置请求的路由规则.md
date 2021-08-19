# istio-配置请求的路由规则
> 使用istio我们可以根据权重和HTTP headers来动态配置请求路由。  

## 基于内容的路由
因为BookInfo示例部署了3个版本的评论微服务，我们需要设置一个默认路由。 否则，当你多次访问应用程序时，会注意到有时输出包含星级，有时候又没有。 这是因为没有明确的默认版本集，Istio将以随机方式将请求路由到服务的所有可用版本。
注意：假定您尚未设置任何路由。如果您已经为示例创建了冲突的路由规则，则需要在以下命令中使用replace而不是create。
下面这个例子能够根据网站的不同登陆用户，将流量划分到服务的不同版本和实例。跟kubernetes中的应用一样，所有的路由规则都是通过声明式的yaml配置。关于reviews:v1和reviews:v2的唯一区别是，v1没有调用评分服务，productpage页面上不会显示评分星标。

- 将微服务的默认版本设置成v1。
> istioctl create -f samples/apps/bookinfo/route-rule-all-v1.yaml  

- 使用以下命令查看定义的路由规则。
> istioctl get route-rules   
由于对代理的规则传播是异步的，因此在尝试访问应用程序之前，需要等待几秒钟才能将规则传播到所有pod。

- 在浏览器中打开BookInfo URL（http://ingress.istio.io/productpage ）您应该会看到BookInfo应用程序的产品页面显示。 产品页面上没有评分星，因为reviews:v1不访问评级服务。

- 将特定用户路由到reviews:v2。
> istioctl create -f samples/apps/bookinfo/route-rule-reviews-test-v2.yaml  
为测试用户test启用评分服务，将productpage的流量路由到reviews:v2实例上。

- 使用jason用户登陆productpage页面。
你可以看到每个刷新页面时，页面上都有一个1到5颗星的评级。如果你使用其他用户登陆的话，将因继续使用reviews:v1而看不到星标评分。

## 内部实现
在这个例子中，一开始使用istio将100%的流量发送到BookInfo服务的reviews:v1的实例上。然后又根据请求的header（例如用户的cookie）将流量选择性的发送到reviews:v2实例上。
验证了v2实例的功能后，就可以将全部用户的流量发送到v2实例上，或者逐步的迁移流量，如10%、20%直到100%。

你想将流量迁移到reviews:v1迁移到reviews:v3版本上，只需要运行如下命令：
- 将50%的流量从reviews:v1转移到reviews:v3上。
> istioctl replace -f samples/bookinfo/kube/route-rule-reviews-50-v3.yaml  
注意这次使用的是replace命令，而不是create，因为该rule已经在前面创建过了。

- 登出jason用户，或者删除测试规则，可以看到新的规则已经生效。
删除测试规则。
> istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml  
> istioctl delete -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml  
现在的规则就是刷新productpage页面，50%的概率看到红色星标的评论，50%的概率看不到星标。
注意：因为使用Envoy sidecar的实现，你需要刷新页面很多次才能看到接近规则配置的概率分布，你可以将v3的概率修改为90%，这样刷新页面时，看到红色星标的概率更高。

- 当v3版本的微服务稳定以后，就可以将100%的流量分摊到reviews:v3上了。
> istioctl replace -f samples/bookinfo/kube/route-rule-reviews-v3.yaml  
现在不论你使用什么用户登陆productpage页面，你都可以看到带红色星标评分的评论了。