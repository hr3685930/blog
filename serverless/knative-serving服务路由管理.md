# knative-serving(服务路由管理)
> Knative默认会为每一个Service生成一个域名，并且IstioGateway要根据域名判断当前的请求应该转发给哪个KnativeService。Knative默认使用的主域名是 example.com，这个域名是不能作为线上服务的。本文我首先介绍一下如何修改默认主域名，然后再深入一层介绍如何添加自定义域名以及如何根据Path关联到不同的KnativeService。   

## 使用自定义主域名 
在阿里云.Knative用户可以通过控制台配置自定义域名，并基于 Path 和 Header 进行路由转发设置。如图所示: 
![](knative-serving%E6%9C%8D%E5%8A%A1%E8%B7%AF%E7%94%B1%E7%AE%A1%E7%90%86/3ED3FFCB-383F-48D7-851A-8C8440FA7504.png)
假设我们自己的域名是:knative.kuberun.com， 现在执行kubectl edit cm config-domain.—namespace.knative-serving. 如图所示， 添 加knative. kuberun.com到ConfigMap中，然后保存退出就完成了自定义主域名的配置。 
![](knative-serving%E6%9C%8D%E5%8A%A1%E8%B7%AF%E7%94%B1%E7%AE%A1%E7%90%86/A297FD3F-20A0-495C-8707-B536CC8BBCEA.png)

### 自定义域名
阿里配置
![](knative-serving%E6%9C%8D%E5%8A%A1%E8%B7%AF%E7%94%B1%E7%AE%A1%E7%90%86/93BD1362-2F10-499C-9191-8A9616B992D9.png)

## 总结
以上主要围绕KnativeService域名配置展开介绍了KnativeServing的路由管理，并且通过阿里云Knative控制台让你更轻松、快捷的实现自定义域名及路由规则，以打造生产可用的服务访问。 

