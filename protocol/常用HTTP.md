# 常用HTTP

## 常用HTTP请求头
* Accept 用来标明期待返回的数据格式以及请求API的版本号，例如application/vnd.site.v1+json
* Content-Type 用来标明POST或者PUT时HTTP的BODY内容格式，例如application/json
* Authorization 用来标明请求者的身份认证信息，例如Bearer 25877b47826645fdb5bf50b9be4d02fa

## 常用HTTP返回头
* Link 用来表示资源相关的链接，在POST创建接口表示新创建资源的地址，在列表接口用来表示分页首末页地址
* X-Total-Count 用来表示分页请求时总记录的数量
* X-RateLimit-Limit 频次限制为多少次
* X-RateLimit-Remaining 频次限制还差多少就触发限制
* X-RateLimit-Reset 下一次频次限制重置时间戳
* X-Request-ID 每一次请求唯一的ID
* Access-Control-Allow-Origin CORS允许所有的Ajax跨域请求，例如*
* Access-Control-Expose-Headers CORS允许JS读取的返回头，例如Link,X-Total-Count

## 常用HTTP状态码
* 200 ok 用来表示请求的资源以及对应的操作正确无误的完成
* 201 created 用来表示POST要创建的资源创建成功
* 400 bad_request 用来表示通用的客户端请求错误，没有具体的明确意指，但可以判断是客户端引起的错误
* 401 authentication_failed 用来表示身份认证的错误，即当前接口需要登陆后再请求但未从请求中获取到身份信息
* 403 authorization_failed 用来表示授权类型的错误，即当前登陆用户没有权限执行该API操作
* 404 resource_not_found 用来表示请求的REST资源不存在，查证资源ID后再试
* 405 method_not_allowed 用来表示请求的REST资源不支持该HTTP Method，查证操作方法后再试
* 409 conflict 用来表示服务器存在的资源与要更新的资源存在冲突，无法完成更新，请重新请求最新资源再更新
* 415 unsupported_media_type 用来表示Http Header指定的Accept类型未被支持，需要更换Accept类型
* 422 validation_failed 用来表示提交上来的数据没有通过validation规则，需要求改投递数据
* 429 too_many_requests 用来表示接口请求次数超限，需要暂缓请求，暂缓时间参考Retry-After的Header值
* 500 internal_error 用来表示接口出现了内部错误，很可能是未处理的异常事件，请通知API组进行处理
* 503 service_unavailable 用来表示服务暂时不可用，客户端可稍后重试