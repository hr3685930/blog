# shell査漏

## shell 通用
```
run-parts dir  #执行指定目录下的所有sh脚本
ctrl+z  #进程扔到后台运行
nohup command > /dev/null 2>&1 &  #后台执行命令，忽略所有挂断信号，标准和错误输出都丢弃到/dev/null
ps -aux  #进程信息
ps -aux --sort=-%mem | less  # 内存倒序查看进程信息
netstat -anpt  #tcp端口监听情况
alias ll='ls -l --color=tty'
tar -zxvf xxx.tar.gz -C newpath  # 指定解压输出路径
echo -e "$var" # 保留变量中的换行符输出（-e参数和双引号重要）
lsb_release -a # 查看系统发行信息

fdisk -l  #设备挂载情况
df -hl  #查看磁盘配额
du -sh *   #查看目录列表占用容量

zip -j to.zip from1 from2  #压缩指定文件，不包含目录结构
modprobe use-storage  #挂载u盘，sdb1
printf '\x45\n'  #打印字符
```

## SSH
```
# 登录
ssh -p port user@host_ip

# 文件传输
scp -P port file usename@ip:/dir #本地文件上传远程服务器，可对换参数逆向操作
scp -P port -r dir usename@ip:/dir #本地目录上传远程服务器，可对换参数逆向操作
```

## top命令

```
空格 #立即刷新
shift+c  #进程显示完整命令
shift+p  #按%CPU排行
shift+m  #按%MEM排行
```

## shell快捷键

```
ctrl+k  #删除光标至行尾的命令
ctrl+u  #删除光标至行首的命令
```

## Centos
```
yum repolist  #打印源列表
yum list installed  #罗列已安装yum包
yum list xxx  #列出xxx的所有版本
yum install XXX --disablerepo=* --enablerepo=YYY  #指定yum源安装某个包
chkconfig --list  #罗列所有注册的服务
```