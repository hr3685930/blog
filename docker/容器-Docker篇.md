# 容器-Docker篇
> 容器的本质是一种特殊的进程  

## namespace和cgroup 
namespace是隔离容器进程
cgroup是限制容器CPU、内存、磁盘、网络带宽等
Linux Cgroups 就是 Linux 内核中用来为进程设置资源限制的一个重要功能。
Linux Cgroups 的全称是 Linux Control Group。它最主要的作用，就是限制一个进程组能够使用的资源上限，包括 CPU、内存、磁盘、网络带宽等等

## Mount namespace与rootfs
> 同一台机器上的所有容器，都共享宿主机操作系统的内核  
Mount namespace 用来隔离文件系统的挂载点，这样进程就只能看到自己的 mount namespace 中的文件系统挂载点。

Rootfs 是一个操作系统所包含的文件、配置和目录，并不包括操作系统内核。

在 Linux 操作系统中，这两部分是分开存放的，操作系统只有在开机启动时才会加载指定版本的内核镜像。

## 镜像
存储各种系统的文件(可分层),容器启动时copy一份到容器里
![](%E5%AE%B9%E5%99%A8-Docker%E7%AF%87/0B281B05-6232-4E39-B192-358E6805E5B9.png)
这个容器进程“python app.py”，运行在由 Linux Namespace 和 Cgroups 构成的隔离环境里；而它运行所需要的各种文件，比如 python，app.py，以及整个操作系统文件，则由多个联合挂载在一起的 rootfs 层提供。这些 rootfs 层的最下层，是来自 Docker 镜像的只读层。在只读层之上，是 Docker 自己添加的 Init 层，用来存放被临时修改过的 /etc/hosts 等文件。而 rootfs 的最上层是一个可读写层，它以 Copy-on-Write 的方式存放任何对只读层的修改，容器声明的 Volume 的挂载点，也出现在这一层。
