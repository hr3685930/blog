# Go Modules依赖管理

> go modules是golang v1.11中引入的新功能，其主要用来解决传统的golang项目必须要放到$GOPATH目录下的问题。通过一套类似于maven的方式来将依赖放到系统指定目录，统一管理。  

## 功能开关
go module支持三种模式：
* on
顾名思义，就是打开go mod
* off
关闭go mod, 使用传统的go vendor模式
* auto
自动模式，当目录在$GOPATH下时，使用 go vendor模式；当在$GOPATH之外时，使用 go mod模式

Go mod开启方式如下:
> export GO111MODULE=on  


## 使用方法
```
go help mod
…

Usage:

	go mod <command> [arguments]

The commands are:

	download    download modules to local cache
	edit        edit go.mod from tools or scripts
	graph       print module requirement graph
	init        initialize new module in current directory
	tidy        add missing and remove unused modules
	vendor      make vendored copy of dependencies
	verify      verify dependencies have expected content
	why         explain why packages or modules are needed

Use "go help mod <command>" for more information about a command.

```


## 生成依赖
### 新项目
对于新项目，直接在项目更目录执行go mod init就可以初始化出go.mod文件。然后，根据需要可以通过命令或者手工的方式往里面添加require依赖。
除了手工添加依赖之外，如果你此时就是在$GOPATH下开发，并且也调试通过了，那么你可以使用go get -m ./… 让它自动查找依赖，并记录在go.mod文件中(你还可以指定 -tags,这样可以把tags的依赖都查找到)。
为了便于复制，我们把命令整理一下
> go mod init  
### 旧项目
对于传统使用go vendor的项目，在根目录下执行go mod init会自动分析vendor中的依赖，并自动写入到go.mod文件的require列表中。



## go.mod文件
Go.mod文件类似于先前govendor的vendor.json, 里面主要包含了各种依赖项目以及对应的版本号和commit id信息。当然，vendor.json还需要指定项目的路径，但是go mod统一将依赖放到$GOPATH/pkg/mod/目录下（其实这类似于maven，节约了磁盘空间）。

```
 ~# cat go.mod
module gitee.com/chenleji/project-1

require (
	github.com/Microsoft/go-winio v0.1.0
	github.com/PuerkitoBio/purell v0.0.0-20170917143911-fd18e053af8a
	github.com/PuerkitoBio/urlesc v0.0.0-20170810143723-de5bf2ad4578
	…
)

```


## 依赖下载
> 在前面已经提到，go mod将依赖项目统一放到$GOPATH/pkg/mod/目录下。对于一些先前没有使用过的依赖项目，目录下显然没有对应的代码，此时我们可以在代码根目录执行以下命令来下载代码依赖：  

> go mod download <path@version>   
> 参数<path@version>是非必写的，path是包的路径，version是包的版本。  


## 依赖更新

如果依赖有变动，go.mod没有及时的更新，可以使用该命令来先做校验：
* 验证依赖项
> go mod verify  

可能有些依赖，我们忘记了在什么时候需要，可以查看为什么需要改以来：
* 解释为什么需要依赖
> go mod why xx  

如果明确依赖需要更新，以下命令可以用来修改依赖：
* 格式化 go.mod 文件
> go mod edit -fmt  

* 添加依赖或修改依赖版本，这里支持模糊匹配版本号
> go mod edit -require=path@version  


* 从 go.mod 删除不需要的依赖、新增需要的依赖，这个操作不会改变依赖版本。
> go mod tidy  


## 从go mod回到go vendor

> 有一些特殊需求，比如传统go vendor模式为docker下编译项目提供了便利性，保障在持续集成的时候，每一次代码构建都从网络上获取依赖。因此，有时候将代码从go mod下转换回go vendor也是有这种需求的。  

> go mod vendor  


##  代码构建
> 在go mod模式下，同样支持使用vendor模式来构建代码：  
* go build -mod=vendor
在开启模块支持的情况下，用这个可以退回到使用 vendor 的时代

* go build -mod=readonly
防止隐式修改 go.mod，如果遇到有隐式修改的情况会报错，可以用来测试 go.mod 中的依赖是否整洁，但如果明确调用了 go mod、go get 命令则依然会导致 go.mod 文件被修改。


## 查看依赖
- 显示依赖关系
> go list -m all  
* 显示详细依赖关系
> go list -m -json all  
* 打印模块依赖图
> go mod graph  


## 配置代理
> export GO111MODULE=on  
> export GOPROXY=https://goproxy.cn  
