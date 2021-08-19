# PHP常见运行模式
PHP常见的运行模式有5种：
1. CGI模式       
该模式下php独立于web server执行，互不影响，安全性和稳定性较高
但每次请求web server都需要fork出一个单独的cgi解释进程
cgi解释进程的反复加载会导致占用资源较多、性能低下，现在基本不用了
2. mod-php模式
作为ApacheModule注入到apache server中
web server会预生成多个解释进程做准备，运行效率较高
3. FastCGI模式
基于cgi架构拓展，其在web server与cgi解释器之间构建中间层作为进程管理器
其管理若干可复用的cgi子进程，cgi解释进程将长驻内存并根据请求来调度
这是最推荐的一种运行模式， 高效稳定、伸缩性、Fail-Over故障切换、平滑重载PHP配置、支持数据库长连接

该模式具体的实现包括:
* php-fpm(PHP最好的FastCGI管理器)
* php-cgi(PHP自带的FastCGI管理器)
* Spawn-FCGI(通用的FastCGI管理服务器)

4. CLI模式
命令行模式
5. ISAPI模式（PHP 5.3-）
运行于IIS上的模块注入模式，微软平台下的东西， 有排他性， 基本很少用
