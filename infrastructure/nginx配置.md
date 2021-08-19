# nginx配置

## nginx.conf
```
user nobody;  # 指定服务运行用户
worker_processes  4;  # 工作进程数设定（一般和cpu数一致）

# 全局错误日志
#error_log  logs/error.log;
#error_log  logs/error.log  notice;
#error_log  logs/error.log  info;

#pid logs/nginx.pid;  # 进程文件指定
keepalive_timeout 60;

#工作模式设定
events {
    use epoll;  # linux2.6+内核可设定epoll（多路复用io）模式提高性能
    worker_connections 1024;  # 每个工作进程的并发连接数
}


http {
    # 日志格式设定
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"'; 

    access_log  logs/access.log  main;  # 全局日志设定
    gzip  on;  # 开启gzip压缩

    # 虚拟主机配置
    server {
        listen    80;
        server_name  www.site.com;
        root /service/www;
        access_log  logs/nginx.access.log  main;  # 当前虚拟主机访问日志

        # php脚本处理
        location ~ .php$ {
            fastcgi_pass 127.0.0.1:9000;
            fastcgi_index index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include fastcgi_params;
        } 
    }
}
```
nginx并发总数 = worker_processes * worker_connections （反向代理一般会使并发性能降低4倍）