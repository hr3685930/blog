# istio-故障注入
## 使用HTTP延迟注入故障
> 为了测试我们的Bookinfo应用微服务的弹性，我们将在用户“jason”的评论之间*注入7*秒的*延迟*：v2和评分微服务。由于*评论：v2*服务对它的评级服务调用有10秒的超时，我们预计端到端流量将继续保持而不会出现任何错误。  

1. 创建故障注入规则以延迟来自用户“jason”（我们的测试用户）的流量。
> istioctl create -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml  

2. 确认规则已创建：
```
istioctl get routerule ratings-test-delay -o yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: ratings-test-delay
  namespace: default
spec:
  destination:
    name: ratings
  httpFault:
    delay:
      fixedDelay: 7.000s
      percent: 100
  match:
    request:
      headers:
        cookie:
          regex: ^(.*?;)?(user=Jason)(;.*)?$
  precedence: 2
  route:
  - labels:
      version: v1
```


3. 观察应用程序行为
以用户“jason”身份登录。如果应用程序的首页设置为正确处理延迟，我们预计它将在大约7秒内加载。要查看网页响应时间，请打开IE，Chrome或Firefox（通常为组合键*Ctrl + Shift + I*或*Alt + Cmd + I*）的“ 开发工具”菜单，选项卡网络，然后重新加载网页。productpage
您会看到网页在大约6秒钟内加载。评论部分将显示对不起，产品评论目前不可用于此书。

## 内部实现
整个评论服务失败的原因是因为我们的Bookinfo应用程序有一个错误。产品页面和评论服务之间的超时时间少于评论和评分服务（10秒）之间的超时时间（3秒+ 1次重试=总计6秒）。典型的企业应用程序可能出现这些类型的错误，其中不同的团队独立开发不同的微服务。Istio的故障注入规则可以帮助您识别这些异常情况，而不会影响最终用户。
### 请注意，我们只限制对用户“jason”的故障影响。如果您以任何其他用户身份登录，则不会遇到任何延迟。
解决这个问题：在这一点上，我们通常会通过增加产品页面超时或将评论减少到评分服务超时，终止并重新启动固定的微服务来解决问题，然后确认productpage返回的响应没有任何错误。

## 使用HTTP中止注入故障
作为弹性的另一个测试，我们将向用户“jason”的用户评分微服务引入HTTP中止。我们希望页面能够立即加载，与延迟示例不同，并显示“产品评分不可用”消息。
1. 在尝试排除故障规则之前，先删除故障延迟注入规则
> istioctl delete -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml  
2. 创建故障注入规则为用户“Jason”发送HTTP abort
> istioctl create -f samples/bookinfo/kube/route-rule-ratings-test-abort.yaml  
3. 确认规则已创建
```
istioctl get routerules ratings-test-abort -o yaml
apiVersion: config.istio.io/v1alpha2
kind: RouteRule
metadata:
  name: ratings-test-abort
  namespace: default
spec:
  destination:
    name: ratings
  httpFault:
    abort:
      httpStatus: 500
      percent: 100
  match:
    request:
      headers:
        cookie:
          regex: ^(.*?;)?(user=Jason)(;.*)?$
  precedence: 2
  route:
  - labels:
      version: v1
```

4. 观察应用程序行为
以用户“jason”登录。如果规则成功传播到所有窗格，您应该立即看到页面加载了“产品评级不可用”消息。从用户“jason”注销，您应该可以在productpage网页上看到评分v2成功显示。

## 清理
* 删除应用程序路由规则：
> istioctl delete -f samples/bookinfo/kube/route-rule-all-v1.yaml  
> istioctl delete -f samples/bookinfo/kube/route-rule-reviews-test-v2.yaml  
> istioctl delete -f samples/bookinfo/kube/route-rule-ratings-test-delay.yaml  
> istioctl delete -f samples/bookinfo/kube/route-rule-ratings-test-abort.yaml    
















