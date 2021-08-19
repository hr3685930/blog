# Tekton原理

> Tekton作为google开源出来的一个CD工具，与knative之前的build有很多相似的地方。build的功能相对来将较单一，且社区提出了很多改进的地方都由于其现有的架构设计局限而找不到解决方案。那么tekton基于build，都有哪些提升？  

## Tekton VS. Build
> 相对于build，tekton主要有以下这些改进，可能归纳的不够完整，后续再逐步补充。  
* 提供了更高层抽象
> knative-build只提供了build和build-template这两层，但是如果要做更高级的流水线编排就比较局限。tekton既有现有资源的映射：task->buildTemplate, taskRun->build; 又提供了更高层的抽象，pipeline和pipelineRun。pipeline可以包含多个task，这样就可以支持更加复杂的流水线逻辑。  
* -支持DAG调度
> tekton的pipeline中，各个task之间可以指定其运行的依赖顺序。通过解析这些依赖生成一个DAG图，在调度的时候基于该图的顺序来逐个调度task。  
* Task不再使用initContainer
> knative-build使用initContainer来做流水线的调度，这就导致了所有的step必须是串行的。在tekton中，task不再使用initContainer来实现各个step之间的顺序，而是通过在entrypoint中指定volume依赖来控制同一个pod中不同container的调用顺序。  
>   
## 实现原理
> 首先创建了pipelineRun之后，reconcile流程会分析里面包含的所有task，并生成DAG图。经过一系列对resources的替换处理之后，从pipeline的DAG中调度出待运行的task，并创建taskRun。pipeline reconcile逻辑会及时处理status，更新状态。  
> taskRun的reconcile通过查询该task的steps，也是解析resources并生成并部署pod。它也有对应的timeout逻辑，用于在执行超时的时候更新taskRun的状态。  

![](Tekton%E5%8E%9F%E7%90%86/tekton-code.png)
## Task调度
> 下面通过一个实例，手动创建一个task和taskrun，然后分析其生成的pod的yaml文件来了解其调度逻辑  

- 先创建一个task和taskRun。
task.yaml 和taskrun.yaml
```
task.yaml
apiVersion: tekton.dev/v1alpha1
kind: Task
metadata:
  name: echo-hello-world
spec:
  steps:
    - name: echo-1
      image: ubuntu
      command:
        - echo
      args:
        - “hello world”
    - name: echo-2
      image: ubuntu
      command:
        - echo
      args:
        - “hello world”
 ~/Desktop/tekton  cat taskrun.yaml
apiVersion: tekton.dev/v1alpha1
kind: TaskRun
metadata:
  name: echo-hello-world-task-run
spec:
  taskRef:
    name: echo-hello-world
```
- 查看CRD资源实例
```
kc get task
NAME               AGE
echo-hello-world   34m

kc get taskrun
NAME                        SUCCEEDED   REASON   STARTTIME   COMPLETIONTIME
echo-hello-world-task-run   True                 33m         33m
```

- 调度的pod，显示已经执行完成
```
kc get pod
NAME                                   READY   STATUS      RESTARTS   AGE
echo-hello-world-task-run-pod-01d042   0/3     Completed   0          2m10s
```
执行之后，可以在dashboad看到对应的两个step的状态。
接下来，分析pod的yaml文件。

![](Tekton%E5%8E%9F%E7%90%86/tekton-dashboard.png)
```
kc get pod echo-hello-world-task-run-pod-01d042 -o yaml
apiVersion: v1
kind: Pod
metadata:
  ……
spec:
  containers:
  - args:
    - -wait_file  # 指定其不依赖任何文件，可以在initContainer执行完成后首先执行”
    - “”
    - -post_file
    - /builder/tools/0 # 指定其运行完成后，才生成“/builder/tools/0”
    - -entrypoint   # 在step中指定的，真正用户需要执行的args在最后
    - echo
    - —
    - hello world
    command:
    - /builder/tools/entrypoint
    env:
    - name: HOME
      value: /builder/home
    image: ubuntu
    name: build-step-echo-1
    ……
    volumeMounts:
    - mountPath: /builder/tools
      name: tools
    - mountPath: /workspace
      name: workspace
    - mountPath: /builder/home
      name: home
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-v254r
      readOnly: true
    workingDir: /workspace
  - args:
    - -wait_file  # 指定其依赖于 “/builder/tools/0”
    - /builder/tools/0
    - -post_file
    - /builder/tools/1 # 指定其运行完成后，才生成“/builder/tools/1”
    - -entrypoint  # 在step中指定的，真正用户需要执行的args在最后
    - echo
    - —
    - hello world
    command:
    - /builder/tools/entrypoint
    env:
    - name: HOME
      value: /builder/home
    image: ubuntu
    name: build-step-echo-2
    ……
    volumeMounts:
    - mountPath: /builder/tools
      name: tools
    - mountPath: /workspace
      name: workspace
    - mountPath: /builder/home
      name: home
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-v254r
      readOnly: true
    workingDir: /workspace
  - args:
    - -wait_file # 指定其依赖于 “/builder/tools/1”
    - /builder/tools/1 
    - -post_file
    - /builder/tools/2
    - -entrypoint
    - /ko-app/nop
    - —
    command:
    - /builder/tools/entrypoint
    image: ljchen/tekton_pipeline_cmd_nop:v0.4.0
    name: nop  # 这是所有step执行完成之后才执行的nop container
    ……
    volumeMounts:
    - mountPath: /builder/tools
      name: tools
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-v254r
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  initContainers:
  - command:
    - /ko-app/creds-init #该容器主要用于拉取镜像是秘钥处理
    env:
    - name: HOME
      value: /builder/home
    image: ljchen/tekton_pipeline_cmd_creds-init:v0.4.0
    name: build-step-credential-initializer-rgt72
    volumeMounts:
    - mountPath: /workspace
      name: workspace
    - mountPath: /builder/home
      name: home
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-v254r
      readOnly: true
    workingDir: /workspace
  - args:
    - -c
    - cp /ko-app/entrypoint /builder/tools/entrypoint  #通过挂卷的方式，将entrypoint复制到用户容器目录
    command:
    - /bin/sh
    env:
    - name: HOME
      value: /builder/home
    image: ljchen/tekton_pipeline_cmd_entrypoint:v0.4.0
    name: build-step-place-tools
    volumeMounts:
    - mountPath: /builder/tools
      name: tools
    - mountPath: /workspace
      name: workspace
    - mountPath: /builder/home
      name: home
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-v254r
      readOnly: true
    workingDir: /workspace
  ……
status:
  ……
```
> 上面添加了较多的注释，应该比较好理解。通过编排pod，修改用户指定step的entrypoint，在启动用户commands前指定依赖文件，从而控制同一个pod中多个container的执行顺序。这就是task的调度中不依赖于initContainer来控制多step顺序的方案。  