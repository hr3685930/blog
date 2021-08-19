# Cloud Native
### 云原生应用架构应该具备的几个主要特征：
- 符合12因素应用(Twelve-Factor Applications)
- 面向微服务架构(Microservices)
- 自服务敏捷架构(Self-Service Agile Infrastructure)
- 基于API的协作(API-Based Collaboration)
- 抗脆弱性(Antifragility)

> 在2017年10月，在接受InfoQ采访时，则对云原生的定义做了小幅调整，将Cloud Native Architectures定义为具有以下六个特质：  
- 模块化(Modularity):（通过微服务）
- 可观测性(Observability)
- 可部署性(Deployability)
- 可测试性(Testability)
- 可处理性(Disposability)
- 可替换性(Replaceability)
> 而在Pivotal最新的官方网站 https://pivotal.io/cloud-native 上,对cloud native的介绍则是关注如下图所示的四个要点：  
![](Cloud%20Native/pivotal-cloud-native.png)

- DevOps
- Continuous Delivery
- Microservices
- Containers

> CNCF重新定义云原生  
![](Cloud%20Native/Pasted%20Graphic.png)

在2018年，随着社区对云原生理念的广泛认可和云原生生态的不断扩大，还有CNCF项目和会员的大量增加，起初的定义已经不再适用，因此CNCF对云原生进行了重新定位。

2018年6月，CNCF正式对外公布了更新之后的云原生的定义（包含中文版本）v1.0版本：

Cloud native technologies empower organizations to build and run scalable applications in modern, dynamic environments such as public, private, and hybrid clouds. Containers, service meshes, microservices, immutable infrastructure, and declarative APIs exemplify this approach.

云原生技术有利于各组织在公有云、私有云和混合云等新型动态环境中，构建和运行可弹性扩展的应用。云原生的代表技术包括容器、服务网格、微服务、不可变基础设施和声明式API。

These techniques enable loosely coupled systems that are resilient, manageable, and observable. Combined with robust automation, they allow engineers to make high-impact changes frequently and predictably with minimal toil.

这些技术能够构建容错性好、易于管理和便于观察的松耦合系统。结合可靠的自动化手段，云原生技术使工程师能够轻松地对系统作出频繁和可预测的重大变更。

The Cloud Native Computing Foundation seeks to drive adoption of this paradigm by fostering and sustaining an ecosystem of open source, vendor-neutral projects. We democratize state-of-the-art patterns to make these innovations accessible for everyone.

云原生计算基金会（CNCF）致力于培育和维护一个厂商中立的开源生态系统，来推广云原生技术。我们通过将最前沿的模式民主化，让这些创新为大众所用。

新的定义中，继续保持原有的核心内容：容器和微服务，但是非常特别的将服务网格单独列出来，而不是将服务网格作为微服务的一个子项或者实现模式，体现了云原生中服务网格这一个新生技术的重要性。而不可变基础设施和声明式API这两个设计指导理念的加入，则强调了这两个概念对云原生架构的影响和对未来发展的指导作用。
![](Cloud%20Native/cloud-native-cncf-new-definition.png)

 

### 12-Factors

> 12-Factors 经常被翻译为12因素，符合12因素的应用则称为12因素应用.  
https://12factor.net/zh_cn/

1. 基准代码
> 12-Factor应用(译者注：应该是说一个使用本文概念来设计的应用，下同)通常会使用版本控制系统加以管理，如Git, Mercurial, Subversion。一份用来跟踪代码所有修订版本的数据库被称作 代码库（code repository, code repo, repo）。  

2. 依赖
> gomod/composer  
12-Factor规则下的应用程序不会隐式依赖系统级的类库。 它一定通过 依赖清单 ，确切地声明所有依赖项。此外，在运行过程中通过 依赖隔离 工具来确保程序不会调用系统中存在但清单中未声明的依赖项。这一做法会统一应用到生产和开发环境。

3. 配置
> 12-Factor推荐将应用的配置存储于 环境变量 中（ env vars, env ）  

4. 后端服务
> 12-Factor 应用不会区别对待本地或第三方服务。 对应用程序而言，两种都是附加资源，通过一个 url 或是其他存储在 配置 中的服务定位/服务证书来获取数据。12-Factor 应用的任意 部署 ，都应该可以在不进行任何代码改动的情况下，将本地 MySQL 数据库换成第三方服务（例如 Amazon RDS）。类似的，本地 SMTP 服务应该也可以和第三方 SMTP 服务（例如 Postmark ）互换。上述 2 个例子中，仅需修改配置中的资源地址。  

5. 构建，发布，运行
> 12-factor 应用严格区分构建，发布，运行这三个步骤。 举例来说，直接修改处于运行状态的代码是非常不可取的做法，因为这些修改很难再同步回构建步骤。  

6. 进程
> 12-Factor 应用的进程必须无状态且 无共享 。 任何需要持久化的数据都要存储在 后端服务 内，比如数据库。  

7. 端口绑定
> 12-Factor 应用完全自我加载 而不依赖于任何网络服务器就可以创建一个面向网络的服务。互联网应用 通过端口绑定来提供服务 ，并监听发送至该端口的请求。  

8. 并发
> 在 12-factor 应用中，进程是一等公民。12-Factor 应用的进程主要借鉴于 unix 守护进程模型 。开发人员可以运用这个模型去设计应用架构，将不同的工作分配给不同的 进程类型 。例如，HTTP 请求可以交给 web 进程来处理，而常驻的后台工作则交由 worker 进程负责。  

9. 易处理
> 12-Factor 应用的 进程 是 易处理（disposable）的，意思是说它们可以瞬间开启或停止。 这有利于快速、弹性的伸缩应用，迅速部署变化的 代码 或 配置 ，稳健的部署应用。  

10. 开发环境与线上环境等价
> 12-Factor 应用想要做到 持续部署 就必须缩小本地与线上差异。 再回头看上面所描述的三个差异:  
- 缩小时间差异：开发人员可以几小时，甚至几分钟就部署代码。
- 缩小人员差异：开发人员不只要编写代码，更应该密切参与部署过程以及代码在线上的表现。
- 缩小工具差异：尽量保证开发环境以及线上环境的一致性。

11. 日志
> 12-factor应用本身从不考虑存储自己的输出流。 不应该试图去写或者管理日志文件。相反，每一个运行的进程都会直接的标准输出（stdout）事件流。开发环境中，开发人员可以通过这些数据流，实时在终端看到应用的活动。  

12. 管理进程
> 进程构成（process formation）是指用来处理应用的常规业务（比如处理 web 请求）的一组进程。与此不同，开发人员经常希望执行一些管理或维护应用的一次性任务，例如：  
- 运行数据移植（Django 中的 manage.py migrate, Rails 中的 rake db:migrate）。
- 运行一个控制台（也被称为 REPL shell），来执行一些代码或是针对线上数据库做一些检查。大多数语言都通过解释器提供了一个 REPL 工具（python 或 perl） ，或是其他命令（Ruby 使用 irb, Rails 使用 rails console）。
- 运行一些提交到代码仓库的一次性脚本。