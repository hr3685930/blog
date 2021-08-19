# Dingo API
> 提供一系列工具帮助你轻松构建自己的可拓展的API  
> 提供工具集  
* 内容协商
* 多种认证适配
* API版本号
* 访问限制与限流
* 输出数据转换
* 错误和异常处理
* 内部请求
* API文档

## 安装
> composer require dingo/api:1.0.*[@dev](https://my.oschina.net/Thinker277)  

## 服务注册
- Lumen下：在bootstrap/app中注册
> $app->register(Dingo\Api\Provider\LumenServiceProvider::class);  
- Laravel下：在`config/app.php`中注册
> Dingo\Api\Provider\LaravelServiceProvider::class  
- 发布配置
> php artisan vendor:publish --provider="Dingo\Api\Provider\LaravelServiceProvider"  

## Facad注册
- Dingo\Api\Facade\API
- Dingo\Api\Facade\Route
env配置
* API_STANDARDS_TREE 标准树
		* x未注册树 - 用于本地开发（建议值）
		* prs私有树 - 用于非商业交付项目
		* vnd第三方树 - 用于公开交付项目
* API_SUBTYPE 项目简称（小写）
* API_PREFIX 前缀（建议值 api）
* API_DOMAIN 子域名（建议值 api.myapp.com）
* API_VERSION 默认版本号（建议值 v1）
* API_NAME api名称（用于生成文档）
* API_CONDITIONAL_REQUEST 条件请求（用于客户端缓存api，默认true）
* API_STRICT 严格模式（强制要求客户端提供Accept头指定版本号，默认false）
* API_DEFAULT_FORMAT 响应格式（默认json）
* API_DEBUG debug信息追加到异常跟踪栈上（默认false）
## 路由接管
```
$router = app('Dingo\Api\Routing\Router');
$api->version('v1', function ($api) {
    $api->get(路径, 'MyController[@MyAction](https://my.oschina.net/myAction)');
    $api->group(['middleware' => 'foo'], function ($api) {
    });
});
```
## 响应
响应模式
1. 直接返回集合或对象（自动处理为json）
2. 响应构建器

		* 响应输出前morph变形流程 response > transformers > formatters > 输出
		* 基类控制器引入响应构建器
				* Trait：use Dingo\Api\Routing\Helpers;
				* 使用：$this->response或者$this->response()
		* 正常响应
				* return $this->response->array($arrayData)
				* return $this->response->item($obj, $transfomrer)
				* return $this->response->collection($objs, $transfomrer)
				* return $this->response->paginator($paginatedObjs, $transfomrer)
				* return $this->response->noContent()
				* return $this->response->created([$uri])
		* 异常响应
				* return $this->response->error($msg, $statusCode)
				* return $this->response->errorNotFound([$msg])
				* return $this->response->errorBadRequest([$msg])
				* return $this->response->errorForbidden([$msg])
				* return $this->response->errorInternal([$msg])
				* return $this->response->errorUnauthorized([$msg])
		* 增加Http头部 $this->response->withHeader($name, $value)
		* 增加元数据 $this->response->addMeta($name, $value)|setMeta([$name,=>$value])
		* 设置状态码 $this->response->setStatusCode($statuCode)
响应变形
* 监听可选事件 ResponseIsMorphing、ResponseWasMorphed并handle响应

## 错误处理
* 错误时直接抛出相应的HTTP异常，dingo会自动设置json输出及匹配的状态

## Transformer
默认使用Fractal Transfomer
调用模式
* 预先注册好，输出时直接返回对象：app('Dingo\Api\Transformer\Factory')->register('‘MyModel, 'MyModelTransformer')
* 预先未注册，输出对象前手动转换：return $this->response->item($obj, $transfomrer)

## 认证
内置支持的认证适配器
* HTTP Basic (Dingo\Api\Auth\Provider\Basic)
* JSON Web Tokens (Dingo\Api\Auth\Provider\JWT)
* OAuth 2.0 (Dingo\Api\Auth\Provider\OAuth2)

调试请求
* 头部必须带上 Accept: application/vnd.YOUR_SUBTYPE.v1+json