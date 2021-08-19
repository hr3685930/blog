#Supervisor 进程守护工具
> python实现的一款用于 监控&控制 类UNIX系统上进程的C/S系统，能很方便的对子进程进行 监听、启动、停止、重启  

##安装
> sudo yum -y install supervisor  

##配置
* 配置文件路径/etc/supervisor/conf.d/进程名.conf
```
* 配置生效需要重启[program:进程名]
* directory=进程工作目录
* command=进程命令
* autostart=true #服务启动时带起本进程
* autorestart=true  #进程异常自动重启
* user=进程启动用户
* priority=-1  #运行优先级，默认-1
* stopsignal=QUIT  #kill进程的信号，默认是TERM
* redirect_stderr=true  #标准错误重定向到标准输出
* stdout_logfile=/dev/null
* stdout_logfile_maxbytes=0
* stdout_logfile_backups=0
* stderr_logfile=/dev/null
* stderr_logfile_maxbytes=0
* stderr_logfile_backups=0
```

##服务端
```
* 负责启动并管理配置的子进程
* 响应客户端命令# 启动管理服务
* supervisord  [-c /etc/supervisor/supervisord.conf]
```

##客户端
* 交互式Shell模式，supervisorctl命令
* 直接执行命令模式
```  
#服务管理
supervisorctl shutdown  #关闭服务
supervisorctl reload  #重载服务配置
进程管理
supervisorctl status  #进程状态
supervisorctl start 进程名|all
supervisorctl stop 进程名|all
supervisorctl restart 进程名|all
```

