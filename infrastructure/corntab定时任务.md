# corntab定时任务

## at 单一计划任务
```
##计划任务
at TIME
at> 多行命令
at> <EOT>  #ctrl+d结束任务设置
#返回job NUm

##查看任务
at -c  jobNum

##撤销某个at任务
at -d  jobNum
atrm   jobNum

#罗列当前用户的at任务
at -l 
at q



时间格式：
now|(HH:MM[am|pm]) 【YYYY-MM-DD】|([Month_EN] [Date_Num])|(+ Num [minutes|hours|days|weeks])


每条任务被写入到 /var/spool/at/ 目录下一个新文件

at任务账号约束：
/etc/at.allow  允许at的用户
/etc/at.deny   at.allow不存在，则考虑这边deny的用户
两个文件都不存在， 则仅root允许
```

## crontab定时任务
```
## 定时任务格式，周0和周7都是周日。
 分    时    日    月    周    命令
0~59  0~23  1~31  1~12  0~7

辅助字符：
*    #任意值
,    #罗列几个值
——   #连接两个值范围
/n   #指定时间间隔， 如 */5 ， 0-59/5 等等 为每隔开5个单位

### 设定系统cron任务 ###
vi /etc/crontab

**************************************************************************************

### 设定用户cron任务 ####

##用户 设置 或 删除 某一条定时任务
crontab -e

##罗列当前用户所有定时任务
crontab -l

##删除当前用户所有定时任务
crontab -r
#比用户定时任务多了一列 user-name
#若删除一条请用crontab -e编辑



每条任务被写入到 /var/spool/cron/ 目录下一个当前账号命名的新文件
一个用户的所有定时任务都在同一个账户命名的文件中
任务日志 /var/log/cron文件

cron任务账号约束配置
/etc/cron.allow  允许cron的用户
/etc/cron.deny   cron.allow不存在，则考虑这边deny的用户
```