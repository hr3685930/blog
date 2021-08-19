# Docker RUN、CMD 和 ENTRYPOINT区别

> RUN、CMD 和 ENTRYPOINT 这三个 Dockerfile 指令看上去很类似，很容易混淆。本节将通过实践详细讨论它们的区别。  
简单的说：
1. RUN 执行命令并创建新的镜像层，RUN 经常用于安装软件包。
2. CMD 设置容器启动后默认执行的命令及其参数，但 CMD 能够被 docker run 后面跟的命令行参数替换。
3. ENTRYPOINT 配置容器启动时运行的命令。

下面我们详细分析。

**Shell 和 Exec 格式**
我们可用两种方式指定 RUN、CMD 和 ENTRYPOINT 要运行的命令：Shell 格式和 Exec 格式 二者在使用上有细微的区别。

Shell 格式
```markdown
<instruction> <command>
例如：
RUN apt-get install python3  
CMD echo “Hello world”  
ENTRYPOINT echo “Hello world” 
```


当指令执行时，shell 格式底层会调用 /bin/sh -c <command> 。
例如下面的 Dockerfile 片段：
ENV name Cloud Man  
ENTRYPOINT echo “Hello, $name” 

执行 docker run <image> 将输出：
Hello, Cloud Man
 
注意环境变量 name 已经被值 Cloud Man 替换。

下面来看 Exec 格式。

Exec 格式
```markdown
<instruction> [“executable”, “param1”, “param2”, …]

例如：
RUN [“apt-get”, “install”, “python3”]  
CMD [“/bin/echo”, “Hello world”]  
ENTRYPOINT [“/bin/echo”, “Hello world”]

```
 

当指令执行时，会直接调用 <command>，不会被 shell 解析。
例如下面的 Dockerfile 片段：
```markdown
ENV name Cloud Man  
ENTRYPOINT [“/bin/echo”, “Hello, $name”]
```

 
运行容器将输出：
Hello, $name
 
注意环境变量“name”没有被替换。
如果希望使用环境变量，照如下修改
```markdown
ENV name Cloud Man  
ENTRYPOINT [“/bin/sh”, “-c”, “echo Hello, $name”]
```

 
运行容器将输出：
Hello, Cloud Man
 
**CMD 和 ENTRYPOINT 推荐使用 Exec 格式**，因为指令可读性更强，更容易理解。
RUN 则两种格式都可以。

**RUN**

RUN 指令通常用于安装应用和软件包。
RUN 在当前镜像的顶部执行命令，并通过创建新的镜像层。Dockerfile 中常常包含多个 RUN 指令。
RUN 有两种格式：
1. Shell 格式：RUN

2. Exec 格式：RUN [“executable”, “param1”, “param2”]

下面是使用 RUN 安装多个包的例子：
```markdown
RUN apt-get update && apt-get install -y \  
 bzr \
 cvs \
 git \
 mercurial \
 subversion
 
```

注意：**apt-get update 和 apt-get install 被放在一个 RUN 指令中执行**，这样能够保证每次安装的是最新的包。如果 apt-get install 在单独的 RUN 中执行，则会使用 apt-get update 创建的镜像层，而这一层可能是很久以前缓存的。

**CMD**

CMD 指令允许用户指定容器的默认执行的命令。
此命令会在容器启动且 docker run 没有指定其他命令时运行。
1. 如果 docker run 指定了其他命令，CMD 指定的默认命令将被忽略。

2. 如果 Dockerfile 中有多个 CMD 指令，只有最后一个 CMD 有效。

CMD 有三种格式：
1. Exec 格式：CMD [“executable”,”param1”,”param2”]
这是 CMD 的推荐格式。

2. CMD [“param1”,”param2”] 为 ENTRYPOINT 提供额外的参数，此时 ENTRYPOINT 必须使用 Exec 格式。

3. Shell 格式：CMD command param1 param2

Exec 和 Shell 格式前面已经介绍过了。
第二种格式 CMD [“param1”,”param2”] 要与 Exec 格式 的 ENTRYPOINT 指令配合使用，其用途是为 ENTRYPOINT 设置默认的参数。我们将在后面讨论 ENTRYPOINT 时举例说明。
下面看看 CMD 是如何工作的。Dockerfile 片段如下：
```markdown
CMD echo “Hello world”

```
 
运行容器 docker run -it [image] 将输出：
Hello world
 
但当后面加上一个命令，比如 docker run -it [image] /bin/bash，CMD 会被忽略掉，命令 bash 将被执行：
root@10a32dc7d3d3:/#
 
**ENTRYPOINT**

ENTRYPOINT 指令可让容器以应用程序或者服务的形式运行。
ENTRYPOINT 看上去与 CMD 很像，它们都可以指定要执行的命令及其参数。不同的地方在于 ENTRYPOINT 不会被忽略，一定会被执行，即使运行 docker run 时指定了其他命令。
ENTRYPOINT 有两种格式：
1. Exec 格式：ENTRYPOINT [“executable”, “param1”, “param2”] 这是 ENTRYPOINT 的推荐格式。

2. Shell 格式：ENTRYPOINT command param1 param2

在为 ENTRYPOINT 选择格式时必须小心，因为这两种格式的效果差别很大。
**Exec 格式**
ENTRYPOINT 的 Exec 格式用于设置要执行的命令及其参数，同时可通过 CMD 提供额外的参数。
ENTRYPOINT 中的参数始终会被使用，而 CMD 的额外参数可以在容器启动时动态替换掉。
比如下面的 Dockerfile 片段：
ENTRYPOINT [“/bin/echo”, “Hello”]  
CMD [“world”]
 
当容器通过 docker run -it [image] 启动时，输出为：
Hello world
 
而如果通过 docker run -it [image] CloudMan 启动，则输出为：
Hello CloudMan
 
**Shell 格式**

ENTRYPOINT 的 Shell 格式会忽略任何 CMD 或 docker run 提供的参数。

**最佳实践**

1. 使用 RUN 指令安装应用和软件包，构建镜像。

2. 如果 Docker 镜像的用途是运行应用程序或服务，比如运行一个 MySQL，应该优先使用 Exec 格式的 ENTRYPOINT 指令。CMD 可为 ENTRYPOINT 提供额外的默认参数，同时可利用 docker run 命令行替换默认参数。

3. 如果想为容器设置默认的启动命令，可使用 CMD 指令。用户可在 docker run 命令行中替换此默认命令。

