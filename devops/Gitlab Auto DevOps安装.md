# Gitlab Auto DevOps(安装)
> 前面已经介绍了Gitlab k8s Runner 及 Gitlab CI 的流程 今天来介绍一下GItlab的新特性Auto Devops  

> 开启Auto Devops之前打开Gitlab的container registry和添加k8s cluster

1. 打开Gitlab Container Registry

```
version: ‘2’
services:
  gitlab:
    container_name: gitlab
    image: gitlab/gitlab-ce:latest
    restart: always
    ports:
      - ‘1022:22’
      - ’80:80’
      - ‘443:443’
      - '4567:4567'
    environment:
      TZ: 'Asia/Shanghai'
      GITLAB_OMNIBUS_CONFIG: |
        # Add any other gitlab.rb configuration here, each on its own line
        external_url 'http://10.10.160.39'
        gitlab_rails['gitlab_shell_ssh_port'] = 1022
        nginx['redirect_http_to_https'] = true
        nginx['ssl_dhparam'] = "/etc/gitlab/ssl/dhparam.pem"
        nginx['ssl_certificate'] = "/etc/gitlab/ssl/domain.crt"
        nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/domain.key"
        high_availability['mountpoint'] = ["/etc/gitlab", "/var/log/gitlab", "/var/opt/gitlab"]

        unicorn['worker_processes'] = 1
        unicorn['worker_memory_limit_min'] = "300 * 1 << 20"
        unicorn['worker_memory_limit_max'] = "400 * 1 << 20"
        unicorn[‘worker_timeout’] = 15
        sidekiq[‘concurrency’] = 10
        sidekiq_cluster[‘enable’] = false
        sidekiq_cluster[‘ha’] = false
        redis[‘maxclients’] = “100”
        nginx['worker_processes'] = 2
        nginx['worker_connections'] = 512
        nginx['keepalive_timeout'] = 300
        nginx['cache_max_size'] = '200m'
        mattermost['enable'] = false
        mattermost_nginx['enable'] = false
        gitlab_pages['enable'] = false
        pages_nginx['enable'] = false
        postgresql['shared_buffers'] = "256MB"
        postgresql['max_connections'] = 30
        postgresql['work_mem'] = "8MB"
        postgresql['maintenance_work_mem'] = "16MB"
        postgresql['effective_cache_size'] = "1MB"
        postgresql['checkpoint_timeout'] = "5min"
        postgresql['checkpoint_warning'] = "30s"

        #Registry配置
        registry_external_url "https://10.10.160.39:4567"  # ContainerRegistry的外部访问地址
        registry_nginx['ssl_certificate'] = "/etc/gitlab/ssl/domain.crt"
        registry_nginx['ssl_certificate_key'] = "/etc/gitlab/ssl/domain.key"
        gitlab_rails['registry_host'] = "10.10.160.39"
        gitlab_rails['registry_port'] = "4567"
        gitlab_rails['registry_api_url'] = "http://localhost:5000" #docker官方5000
        gitlab_rails['gitlab_default_projects_features_builds'] = false #默认关闭CI
        gitlab_rails[‘gitlab_default_projects_features_container_registry’] = false #关闭Registry
    volumes:
      - ./gitlab/config:/etc/gitlab
      - ./gitlab/logs:/var/log/gitlab
      - ./gitlab/data:/var/opt/gitlab
      - ./gitlab/certs:/etc/gitlab/ssl
```
> 在项目下的设置>General project打开  

2. 添加k8s Cluster
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/B0DC21F7-A6DC-4410-89C2-DAFDB031E92D.png)


## 安装组件
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/ED61CAD8-114D-4425-8120-D5F0B9ADC3F3.png)
> 这里因为墙的关系可能会安装不了,我们可以更改配置文件或者先拉取到节点来解倔, 这里我采用修改gitlab配置文件来解决  
1. 安装helm
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/12ED75F5-A8E0-4F71-BF5D-BF07B96FFC32.png)
报错直接修改镜像

2. 安装ingress
> 首先更改helm镜像 vim /opt/gitlab/embedded/service/gitlab-rails/lib/gitlab/kubernetes/helm/client_command.rb  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/DD1CABCA-8835-4AC8-B1BA-2DCB4B9E2716.png)
> 修改完后重新加载 gitlab-ctl reconfigure && gitlab-ctl restart  
> 同helm也会报错直接替换镜像googlecontainer/defaultbackend-amd64:1.5  

![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/2CB36057-410A-46CC-9B67-CF279957005E.png)
> 此时ingress会调用阿里的slb生成VIP  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/64343365-A03B-4531-BA32-A7FB3E728039.png)

3. 安装Prometheus
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/4EE105C5-884A-466D-B826-F1A73B2CC1B0.png)
> 包如下错误是因为没有pv存储,我们这里用来测试暂时修改为emptyDir  

![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/1351973A-C95D-4A02-ABC4-2E54C75DD2B9.png)

4. 安装Runner
很顺利~~~
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/EB3AF46D-DA08-425D-984F-C4AE5F78CBCB.png)


## 测试Auto devops
我们启用auto devops
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/34BB12D4-14D3-4BC4-A83D-2FA37CF8C1C5.png)
> 这里导入一个nodeJS模板  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/AB8ED207-C461-43F4-91D2-9615EC9FCAB8.png)
> 可以看到流水线已经触发  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/FEF82D7D-997C-4567-8281-BC5EE634C65E.png)
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/9E8207DD-8E80-45B3-88F3-25CE5879AE49.png)
registry.gitlab.com的镜像可能会拉取失败,重试就好了

> 我们先禁用掉自动测试功能  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/68AF2A92-249E-447F-A837-330AE947DBC5.png)
> 流水线成功  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/808E7312-129E-40BA-93A3-03DB7A6765C5.png)
> 查看域名并访问  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/E287C399-D857-4D09-8E5A-77128651EBEB.png)
> 查看监控  
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/2F5A5753-D6F9-4B9A-BAC5-A666ECDCADD9.png)
![](Gitlab%20Auto%20DevOps%E5%AE%89%E8%A3%85/35F7E9BC-EF64-4913-867C-F3C1F8F3D07C.png)







