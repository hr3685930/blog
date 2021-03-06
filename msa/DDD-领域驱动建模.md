# DDD-领域驱动建模

> DDD 核心思想是通过领域驱动设计方法定义领域模型，从而确定业务和应用边界，保证业务模型与代码模型的一致性。  

## 领域和子域
领域就是用来确定范围的，范围即边界，这也是DDD 在设计中不断强调边界的原因。

领域可以进一步划分为子领域。
我们把划分出来的多个子领域称为子域，每个子域对应一个更小的问题域或更小的业务范围。

## 核心域、通用域和支撑域
在领域不断划分的过程中，领域会细分为不同的子域，子域可以根据自身重要性和功能属性划分为三类子域，它们分别是：核心域、通用域和支撑域。

通用域是一些业务的通用功能，比如认证，权限等等，这类应用不需要做太多的定制化，没有企业特点。
支撑域具有企业特性，但不具有通用性，比如数据代码类的数据字典等系统。
核心域 : 客户，商品和订单是支撑核心业务

## 限界上下文
理论上限界上下文就是微服务的边界。
我们将限界上下文内的领域模型映射到微服务，就完成了从问题域到软件的解决方案。
