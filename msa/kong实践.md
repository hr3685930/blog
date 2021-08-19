# kong实践
## Kong介绍
* API网关是所有的客户端和消费端统一的接入点，它对消费者屏蔽了后端的微服务，并处理非业务功能。通常，网关也提供REST/HTTP的访问API，服务端通过API-GW注册和管理服务。
* Kong是一个可扩展的开放源码的API中间件, 它在任何RESTful API的前面运行，通过插件扩展，它提供了超越核心平台的额外功能和服务。Kong是基于OpenResty（不要问我Openresty是不是基于Nginx），在其基础上做了一系列的抽象概念、提供了相应的Lua脚本编程框架和库；同时，它也提供了集群方案，是一个满足生产级的API网关解决方案。
## 部署实践
由于安装Openresty需要各种依赖，在MacBook上安装了很多次最后都放弃了；最后还是采用docker的方式来部署（不得不又膜拜一下docker，技术都是linux的老技术，重点是如何把这些技术组合成完美的产品。）
## docker-network
> docker network create kong-net  
## 安装数据库
这里我们采用postgres数据库方案：
```
docker run -d —name kong-database \
               —network=kong-net \
               -p 5432:5432 \
               -e “POSTGRES_USER=kong” \
               -e “POSTGRES_DB=kong” \
               postgres:9.6
```
然后需要初始化数据库表结构：
```
docker run —rm \
     —network=kong-net \
     -e “KONG_DATABASE=postgres” \
     -e “KONG_PG_HOST=kong-database” \
     -e “KONG_CASSANDRA_CONTACT_POINTS=kong-database” \
     kong:latest kong migrations bootstrap
```
## 安装Kong
运行Kong需要注意，指定proxy的端口号 KONG_PROXY_LISTEN 并暴露容器的端口出去。如果要使用自定义的lua plugin，还需要使用环境变量KONG_CUSTOM_PLUGINS指定插件的名称。
```
docker run -d —name kong \
     —network=kong-net \
     -e “KONG_DATABASE=postgres” \
     -e “KONG_PG_HOST=kong-database” \
     -e “KONG_CASSANDRA_CONTACT_POINTS=kong-database” \
     -e “KONG_PROXY_ACCESS_LOG=/dev/stdout” \
     -e “KONG_ADMIN_ACCESS_LOG=/dev/stdout” \
     -e “KONG_PROXY_ERROR_LOG=/dev/stderr” \
     -e “KONG_ADMIN_ERROR_LOG=/dev/stderr” \
     -e “KONG_ADMIN_LISTEN=0.0.0.0:8001, 0.0.0.0:8444 ssl” \
     -p 8000:8000 \
     -p 8443:8443 \
     -p 8001:8001 \
     -p 8444:8444 \
     kong:latest
```
    # 如果使用自定义plugin-xxx，需要指定
    -e “KONG_CUSTOM_PLUGINS=plugin-xxx” \
    # 如果指定使用DNS服务器，可以用consul做dns服务器
    -e “KONG_DNS_RESOLVER=x.x.x.x:8600” \
至此，在本地访问管理端API:  [http://localhost:8001](http://localhost:8001/)  应该能够返回数据了，说明kong已经开始工作了。
## 安装图形化管理界面
这里使用konga来管理，界面比较美观，只是暂时还没有中文版。
```
docker run -d -p 1337:1337 \
     —network=kong-net \
     —name konga \
     -e “NODE_ENV=development” \
     pantsel/konga
```
安装好后，我们登录上去，默认登录用户名密码为：
Admin login: admin
Password: adminadminadmin
将管理的kong地址设置为:  [http://kong:8001](http://kong:8001/)  即可开启Kong之旅了。
操作顺序依次为：
* upstream
* upstream’s target
* service
* service’s router

## 抽象概念
下面来讲讲kong里面抽象的各种概念，与Nginx中有一些差异。
![](kong%E5%AE%9E%E8%B7%B5/%E6%9C%AA%E7%9F%A5.png)
- upstream: 同nginx的upstream，kong会对upstream下的各个target做健康检查，基于健康检查结果流量负载均衡；因此，其调度的最小单元为upstream。
* target: 就是一个具体的主机或者微服务实例，需要指定Ip和服务Port；
* service：我的理解是这里的service是为了对应于某一个微服务，现在的用法都是一个service对应一个upstream；
* router：类似于kubernetes上ingress的概念，分流的粒度也是可以达到某一个域名下的sub-path上；在konga上，router只能够在service创建之后为该service指定其外部ingress的规则。
## 理想方案
Kong为了适配微服务架构，虽然抽象出了service的概念，但是其upstream的增加还是无法做到自动化。如果能够集成consul之类的服务注册发现工具，将是另一片美好的蓝天。但是… 我们后面会在讲到lua脚本hook其实并没有办法处理这些类似于外部配置的问题。因此如果希望达成这一理想，还需要在外部开发一个程序，该程序实现watch consul上passing的服务，并及时的调用kong admin的api来实现动态更新upstream的target。
## Plugin
Kong的强大就在于它提供了众多的Plugins，进到kong的docker容器，在lua的路径下(/usr/local/share/lua/5.1/kong/plugins/*)可以查看到系统自带的所有lua plugin。
```
/usr/local/share/lua/5.1/kong/plugins # ls
acl                    correlation-id         http-log               loggly                 request-size-limiting  syslog
aws-lambda             cors                   ip-restriction         oauth2                 request-termination    tcp-log
azure-functions        datadog                jwt                    post-function          request-transformer    udp-log
base_plugin.lua        file-log               key-auth               pre-function           response-ratelimiting  zipkin
basic-auth             galileo                ldap-auth              prometheus             response-transformer
bot-detection          hmac-auth              log-serializers        rate-limiting          statsd
```

如何实现一个自己的Plugin：
# 原理
![](kong%E5%AE%9E%E8%B7%B5/%E6%9C%AA%E7%9F%A5%202.png)
上面拍拍贷的一幅图，详细的阐述了Openrestry在nginx转发的过程中设置hook的地方（可以参考kong-nginx.conf配置文件），如果对于该将自己的操作放到哪个函数中实现还不清楚的可以参考下面。
 [kong源码导读](http://techblog.ppdai.com/2018/04/16/20180416/)  <- 这篇文章值得看看
init_by_lua
  发生在master进程启动阶段。这里会对数据访问层进行初始化，加载插件的代码，构造路由规则表。
init_worker_by_lua
  发生在worker进程启动阶段。这里会开启数据同步机制，执行每个插件的init_worker方法。
set_by_lua
  处理请求第一个执行阶段。这里可以做一些流程分支处理判断变量初始化。kong没有使用该阶段。
rewrite_by_lua
  这里可以对请求做一些修改。kong在这里会把处理代理给插件的rewrite方法。
access_by_lua
  kong在这里对请求进行路由匹配，找到后端的upstream服务的节点。
balancer_by_lua
  kong在这里会把上一阶段找到的服务节点设置给nginx的load balancer。如果设置了重试次数，此阶段可能会被执行多次。
header_filter_by_lua
这里可以对响应头做一些处理。kong在这里会把处理代理给插件的header_filter方法。
body_filter_by_lua
  这里可以对响应体做一些处理。kong在这里会把处理代理给插件的body_filter方法。
log_by_lua
  kong在这里会通过插件异步记录日志和一些metrics数据。
# 实现流程
创建Plugin需要定义在目录下创建两个文件：
* schema.lua 指定配置文件
* handler.lua 处理逻辑
schema.lua中需要return出对应的fields（fields里面包含了所有的配置），比如：
```
return {
	no_consumer = true,
	fields = {
		RedisHost = {
			type = “string”,
			required = false,
			default = “127.0.0.1”
		},
	}
}
```
然后在handler.lua中实现各种业务逻辑，这里需要注意的是：所有的plugin都需要集成Kong已经定义好的BasePlugin类，而该类中具体的方法见上文。
```
local BasePlugin = require “kong.plugins.base_plugin”
local access = require “kong.plugins.auth.access”
local rewrite = require “kong.plugins.auth.rewrite”

local AuthHandler = BasePlugin:extend()

AuthHandler.PRIORITY = 1001
AuthHandler.VERSION = “0.1.0”

function AuthHandler:new()
  AuthHandler.super.new(self, “auth”)
end

function AuthHandler:access(conf)
  AuthHandler.super.access(self)
  access.execute(conf)
end
function AuthHandler:rewrite(conf)
  AuthHandler.super.rewrite(self)
  rewrite.execute(conf)
end
return AuthHandler
```
对应access和rewrite的实现就可以写到auth目录下的access.lua 和rewrite.lua中，这里只需要在reqire引入auth目录下的access和rewrite就行了。kong的Plugin是有优先级的，每一个plugin都需要设置一个优先级: AuthHandler.PRIORITY 貌似数值越大越优先。
另外，这里卖个关子。kong里面有众多操作db或者IO的行为，这都应该是一个较耗时的操作，它是如何做到不影响nginx转发性能的呢？有时间大家可以详细研究研究kong自带的lua插件的源码。
## Proxy-Cache
之前大数据的数据服务平台提供的API频繁被外部业务调用，并发量非常高，导致elasticsearch堆内存不足。但是很多数据和之前的调用都是一样的结果，完全没有必要重复到ES中去做查询（因为ES是多个节点，一旦查询请求落到的节点没有缓存之前的结果，就会导致重复查询）。我们希望通过对某些特定的请求做一定时间的缓存，这样可以支持更高的并发，同是也环节了后端ES的压力。
从v1.2.x开始，kong支持proxy-cache插件，该插件就可以满足我们的需求。只是当前官方释放出来的是只基于memory的，而基于外部redis集群的方案并没有开放出来；我们暂且了解一下基于memory的proxy-cache的功能。
首先，proxy-cache支持在几个粒度创建缓存：
	1. consumer
	2. service
	3. route
这三个粒度就不再过多的解释，在实现中，它首先通过使用nginx的 $request_uri 来作为缓存的key。这就除了包含url路径之外，还包括了后面的query参数。同时，用户可以指定需http method和http response，以及http header中的文本格式等。所有这些信息会作为key，当新的请求的这些信息重新hash后的key在缓存中找到对应的value时，直接从cache中返回数据，而不再转发到upstream。当然，用户可以指定缓存的TTL时间。
当使用memory模式的时候，相当于软件模拟了一个内存字典，在字典中缓存了对应的数据。但是由于内存是单机的，没法做多个kong的全局缓存；而使用redis可以做到这一点。但是，对于一般的业务来讲，如果kong的数量不是特别多，只是做了一个主备的话，其实使用memory也问题不大，不外乎后端多承受一部分压力，但是在高并发的场景中，该插件是有实际收益的
