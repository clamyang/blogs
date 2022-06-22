---
title: kubernetes in action 读书笔记
comments: true
---

关于 kubernetes in action 的读书笔记

<!--more-->

我读的这个是第一版，发现有很多东西都已经过时了，该书的原作者正在编辑第二版，manning 上可以看到第二版的相关内容，正处在 MEAP 阶段。我一开始能够免费阅读已开放的章节，今天到了公司突然不行了.. 需要花钱购买 MEAP 版本..

## 1-Introducing Kubernetes

首先介绍了传统 VM 和 Container 的区别，以及各自的优缺点，还针对 microservice 进行了系统的概述，怎么一步步从 Big  Monolithic 架构演进到 microservice，以及垂直扩展、水平扩展。

如何对 Big Monolithic 架构拆分出来的 service 进行管理？

结尾描述了 k8s 集群的 architectue：

从硬件角度区分，可以分为 master node 和 worker node

- master  node 运行 K8s Control Plane 用来控制和管理 k8s 系统
- worker nodes 是实际运行我们 app 的节点

master node 又是由以下几个组件组成：

- Kubernetes API Server 用来和其他的 K8s Control Plane 组件进行通信
- Scheduler 调度我们的部署的
- Controller Manager 执行集群级别的函数，比如跟踪 work node，replicating components，处理失败的节点等
- etcd 一个可靠的分布式数据存储，用来持久化存储集群配置

worker nodes 有如下的组件：

- Kubernetes Service Proxy 可以在我们的应用组件件提供 LB 功能
- Docker, rkt (rock-it) 用来执行我们的容器化的app

- Kubelet 管理当前 node 上的容器和 API server 通信

![k8s-overview](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/k8s-overview.png)

看到这句话，感觉真的是爆笑... 太逗了哈哈

![image-20211216230209359](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211216230209359.png)

## 2-First steps with Docker and Kubernetes

第二章，主要涉及到一些动手实践的东西，Docker 中启动一个容器，K8s 中运行一个容器，学习使用 kubectl。

首先介绍了 docker 的简单使用，如下这个简版的 docker 运行 container 的过程

<img src="https://gitee.com/yangbaoqiang/images/raw/master/blogpics/docker-overview.png" alt="docker-overview" style="zoom:150%;" />

书中讲了一些关于 docker 的基本使用，Dockerfile 的构建，启动、运行、停止、查看 container，怎么把image推送到 registry。

接下来的内容就是，配置 k8s，让我们的 container 跑在 k8s 中，而不是直接通过 docker 来运行。



需要注意的是，使用 Dockerfile 构建镜像的过程，会把需要的文件上传到 docker daemon，所以**在构建目录下去除没必要的文件**，否则会降低 build 的速度，尤其当 docker daemon 在远程的时候。



### 配置 K8s Cluster

上来就是当头一棒，告诉我们配置一个多个节点的 k8s 集群很难，要对 linux 和 网络非常熟悉，因为需要使多个物理机、虚拟机设置在一个适当的网络环境中，让他们彼此通信。

####  `minikube` 安装 kubernetes

先到 [minikube](https://minikube.sigs.k8s.io/docs/start/) 官网下载一个安装包，然后直接执行 `minikube start` 命令，会拉取一些镜像。在拉取镜像的过程安装 [kubectl](https://kubernetes.io/docs/tasks/tools/) 用来和  K8s 进行交互，直接下载对应的可执行文件即可，然后添加到我们的环境变量中。

（貌似在执行完 minikube start 之后就已经自动安装了 kubectl，删除垃圾容器的时候还误删了自己的一个的 practise container .. 幸亏以前打了镜像，能恢复过来不然又要从头开始..）



 以上步骤都执行完成后，使用 `kubectl cluster-info` 查看一下 cluster 的信息。

![image-20211217140936336](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211217140936336.png)



`minikube ssh` 登录到刚刚部署的 master node

![image-20211217141152654](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211217141152654.png)



可以看到，在这个 master node 上正在运行的线程，有第一章提到的，etcd/ apiserver/ schedule 等。

`kubectl get nodes` 可以查看节点信息，比如有多少个 control plane(master)、多少个 worker。

`kubectl describe node node_name` 可用来查看一个 node 详细信息，带 name 显示特定的 node 信息，不带 name 显示所有。

下面作业又教我们设置快捷键。。确实是，每次想看看看服务的日志信息，都需要输入完整的命令实在是太痛苦了，这部分章节再次就不再赘述。

### 部署 Node.js app

因为还没学习 yaml 文件的方式进行部署，所以只使用 run 命令进行部署 `kubectl run --image=yangbaoqiang2019/kubia --port=8080 ` 

这里跟书中内容有点出入，可能是因为书中内容的版本比较老，通过 `kubectl version` 查看一下当前 k8s 的版本信息，如果大于 1.18 版本在部署的时候就不需要添加 `--generator=run/v1` 命令

#### Pods

K8s 并不直接对单独的 container 进行处理，取而代之的是，Pod。Pod 是一个或多个紧密关联的容器组合，这些容器会在同一个 worker node 或 Linux namespace 运行。

**Pod-node-container 之间的关系**

![image-20211217151611381](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211217151611381.png)

`kubectl get pods` 查看 pods，k8s 中没办法直接查看容器，因为容器在k8s中并不是一个独立的对象。

`kubectl describe pod pod_name` 用来查看pod的详细信息。

看一下执行 kubectl run 命令后发生了什么

![1639725901(1)](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/1639725901(1).jpg)

感觉大体上跟 docker 的执行过程相似。



**如何访问我们部署的pod**

如上边的图 2.5 每一个pod都有自己的 IP 地址，但是这个IP是集群内部的IP, 并不能从外部进行访问

要是想把内部的pod暴露出去，需要通过 service 来实现，创建一个 LB 的 service，创建一个常规的 service 还不行，也只能从内部进行访问。

- 创建 service object
  - `kubectl expose rc kubia --type=LoadBalancer --name kubia-http`

​			  这里又搞了点小插曲.. 1.21 版本的 k8s 不支持 rc 这个对象了.. 通过上述命令创建不出来 LB，并且 minikube 是不支持 LB 的，externalID 一直都是空。。



**为什么我们要搞一个 service**

无论你后边的 pod 怎么变我暴露给你的外部externaIP 都是不变的。



**查看我们app跑在哪个node上**

在 k8s 中，app 被调度到哪个 node 上其实不重要，只要能给  app 提供运行所需要的环境就行。

`kubectl get pods -o wide`



第二章末尾还讲了一个 k8s dashborad 主要是一些可视化操作，不过要想逼格高，我觉得还是命令更能装逼哈哈



## 3-Pods： running containers in Kubernetes

第三章就开始深入的对 k8s 的各个组件进行讲解，首先从 pods 开始入手，因为 pods 是 k8s 中最重要，最核心的概念。



通常，一个 pod 只包含一个 container

问题：多个进程跑在一个容器中的问题。

日志管理、崩溃重启等。



Pod 是一个逻辑上的主机，他的表现像是一个 VM 或者物理机。我理解为，就像 Container 运行在 Host 上一样，现在是 container 运行在 pod 上。



**通过 pod 管理容器**

这里，书中举例，两个不相干的服务放在同一个 pod 中是否正确？

答案是，不正确的！作者从资源利用上进行分析，然后又从水平扩展进行了分析，因为在 k8s 中的水平扩展是以 pod 为单位的，每一次水平扩展都会将整个pod复制一份。



既然总说最好要在一个pod中运行一个container，那么什么时候应该在 pod 中运行多个 container 呢？

当 main container 不能独自完成某项任务，需要额外的 container 来辅助执行时。这种情况下就需要将多个 container 运行在同一个 pod 中。比如，在这个 pod 中，有一个container是用来提供目录下的文件服务，另一个 container 往目录中的文件写东西。



作者还给我们提供了几个标准用来判断是否要在同一个pod中运行多个 container

- 他们是否需要在一起运行或他们可以在不同的 host 上运行？
- 他们是否是一个独立服务或一个独立的组件？
- 他们一定要一起扩展或单独的扩展？

![image-20211220152602878](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211220152602878.png)



**通过 YAML 或 JSON 创建 pods**

这个地方和 Dockerfile 上传到 docker daemon 类似。



**学习下 YAML 各个字段的含义**

~~~yaml
# kubernetes API 版本
apiVersion: v1
# k8s obj/res 类型
kind: Pod
# pod 信息
metadata:
  creationTimestamp: "2021-12-17T06:42:21Z"
  labels:
    run: kubia
  name: kubia
  namespace: default
  resourceVersion: "17898"
  uid: c048e5b1-5aa6-4ea7-887a-4601c2f3feff
# pod 详细信息
spec:
  containers:
  - image: yangbaoqiang2019/kubia
    imagePullPolicy: Always
    name: kubia
    ports:
    - containerPort: 8080
      protocol: TCP
    resources: {}
    terminationMessagePath: /dev/termination-log
    terminationMessagePolicy: File
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: kube-api-access-9hqxs
      readOnly: true
  dnsPolicy: ClusterFirst
  enableServiceLinks: true
  nodeName: minikube
  preemptionPolicy: PreemptLowerPriority
  priority: 0
  restartPolicy: Always
  schedulerName: default-scheduler
  securityContext: {}
  serviceAccount: default
  serviceAccountName: default
  terminationGracePeriodSeconds: 30
  tolerations:
  - effect: NoExecute
    key: node.kubernetes.io/not-ready
    operator: Exists
    tolerationSeconds: 300
  - effect: NoExecute
    key: node.kubernetes.io/unreachable
    operator: Exists
    tolerationSeconds: 300
  volumes:
  - name: kube-api-access-9hqxs
    projected:
      defaultMode: 420
      sources:
      - serviceAccountToken:
          expirationSeconds: 3607
          path: token
      - configMap:
          items:
          - key: ca.crt
            path: ca.crt
          name: kube-root-ca.crt
      - downwardAPI:
          items:
          - fieldRef:
              apiVersion: v1
              fieldPath: metadata.namespace
            path: namespace
# pod 和 container 的详细状态            
status:
  conditions:
  - lastProbeTime: null
    lastTransitionTime: "2021-12-17T06:42:21Z"
    status: "True"
    type: Initialized
  - lastProbeTime: null
    lastTransitionTime: "2021-12-20T08:03:03Z"
    status: "True"
    type: Ready
  - lastProbeTime: null
    lastTransitionTime: "2021-12-20T08:03:03Z"
    status: "True"
    type: ContainersReady
  - lastProbeTime: null
    lastTransitionTime: "2021-12-17T06:42:21Z"
    status: "True"
    type: PodScheduled
  containerStatuses:
  - containerID: docker://e010602aa40ef907b61891d7ea5ae2adc5c5837eb1be09b59fed499a1a49cf2d
    image: yangbaoqiang2019/kubia:latest
    imageID: docker-pullable://yangbaoqiang2019/kubia@sha256:90a1f114d736dabfed10348ae0c67206e68f1c736f2b3382c91adc473f4dad73
    lastState:
      terminated:
        containerID: docker://59168997abb548e83da52623973820e56e43e7bbaee4490f1065b7eb5d0d0a83
        exitCode: 255
        finishedAt: "2021-12-20T05:43:35Z"
        reason: Error
        startedAt: "2021-12-20T02:43:44Z"
    name: kubia
    ready: true
    restartCount: 3
    started: true
    state:
      running:
        startedAt: "2021-12-20T08:03:03Z"
  hostIP: 192.168.49.2
  phase: Running
  podIP: 172.17.0.2
  podIPs:
  - ip: 172.17.0.2
  qosClass: BestEffort
  startTime: "2021-12-17T06:42:21Z"
~~~

别看这里列出来的比较多，但是实际上重要的部分并不多。其中 `Metadata` `Spec` `Status` 是一些关键的部分。

- Metadata includes the name, namespace, labels, and other information about the pod.
- Spec  contains the actual description of the pod’s contents, such as the pod’s containers, volumes, and other data.
- *Status* contains the current information about the running pod, such as what condition the pod is in, the description and status of each container, and the pod’s internal IP and other basic info.

status 展示的是一个运行中的 pod 信息，在我们创建 pod 的时候并不需要关心 status 的内容。

注：可以使用 `kubectl explain pod` 来查看关于 pod 的一些字段



**kubectl create** 创建 pod

通过 kubectl create -f filename 命令，创建一个 pod，create 能够创建的不止 pod 

- -f --> filename



**查看 app log**

注：作者提到容器化的应用，通常将日志和错误以流式的方式输出到控制台，而不是将他们写入到文件中。主要是给用户提供了一个标准、简单的方式。

关于日志大小，容器日志采用的是循环写入的方式。



**port-forward**

通过这种方式，方便测试单个 pod，并不需要创建一个 service，指定一个本地端口和 pod 进行关联即可。

`kubectl port-forward kubia-manual 8888:8080`



**使用 labels 组织 pod**

方便 pod 的管理.. 可以在 YAML 中添加 labels 来指定该 pod 的 label，只要不重复可以随便添加。

- 列出所有的 label
  - `kubectl get pods --show-labels`
- 只查看特定的 label
  - `kubectl get pods -L env`
- 修改已存在的 label
  - `kubectl label pod pod_name labelKey=labelValue`
- label selector 支持多条件查询
  - `kubectl get pods -l app=as,rel=stable`



**使用 label 控制调度**

为什么需要将pod调度到特定的 host 呢？本来 k8s 已经给我们把底层抽象出来了，无差别统一，给我们封装成了一个对象。

这个调度呢，还是给我们保留了一定的余地，比如我们有个 GPU 类型的主机，那这时候就需要将 Pod 调度到 GPU 类型的 Host 上。



正是使用了 label 来实现上述的调度。既然想将 pod 根据 label 调度到特定的 worker node， 那么 worker node 也是需要打上 label 的，不然怎么进行一一对应。

![image-20211220210440669](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211220210440669.png)

**namespace 对资源分组**

了解为什么需要 namespace ？因为 namespace 可以将庞大的组件划分为更小且更容易区分的组。也可以用来根据不同的租户分离资源，也可以将资源划分为生产、开发、QA..



注：namespace 还可以提供权限控制甚至限制计算资源（配额）；有些 k8s 中的资源是没有 namespace 的，比如 node。



**创建 namespace**

- 可以通过 YAML 创建
- 可以通过 command line 创建 `kubectl create ns ns_name`



第三章结尾又讲了一些删除 pod 的相关操作.. 



## 4-Replication and other controllers: deploying managed pods

Pod 虽然是最基本的调度单位，但实际上我们并不直接对 Pod 进行操作，而是对 Pod 又做了一层封装，ReplicationContronller 或 Deployment。

这里有个问题就是，我们之前学习到的是直接对 Pod 进行操作，尽管 Pod 故障了，仍然可以在当前 node 上进行重启，但是 node 要是故障了，就会导致当前这个 pod 丢失，无法被恢复。



container 中进程崩溃了 k8s 可以帮助我们进行恢复，如果进程没崩溃只是一种假死状态（执行死循环）应该怎么办？这时候容器中的服务不能正常提供服务了。



**liveness probes**

k8s 有三种机制检测容器是否存活。

- HTTP GET，检测HTTP返回的状态码
- TCP Socket，尝试和 container 建立链接（三次握手过程）
- exec 命令，检查退出码

**基于 HTTP**

![image-20211221143951994](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211221143951994.png)



获取崩溃容器的日志 `kubectl logs mypod --previous` ，通过这种方式可以上去看看日志为什么容器崩溃了。



![image-20211221151336748](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211221151336748.png)

可以看到，图片中显示我们的 container 已经重启了 9 次。

delay 表明这个探活的请求是从容器启动后多久开始发起的，0s 显示，容器一运行这个请求发出去了。delay 的时间是可以调整的 `initialDelaySeconds` **一定要设置该参数，在 delay = 0s 的情况下，容器有可能未准备好接收请求，还在初始化的过程中，这样就会导致 probe 失败重启该容器，陷入死循环**



period 表明发送请求的周期。

success 表明成功的次数。

failure 表明失败的次数。

timeout 表明请求的超时时间，如果超过 1s 就表明探活失败，要重启 container 了。

Exit Code： 退出的状态码，137 表明 exit code is 128 + 9 (SIGKILL); 143 表明 128 +

15 (SIGTERM)

使用这两种信号，是因为一个为了优雅的关停，如果说超过某个阈值container仍然没有停下来，就发送 kill 信号，强制关闭。

注意：这个过程是不需要 master 来参与的，只需要 pod 所在的 node 的 kubelet 即可完成对 pod 的恢复。**但是，如果 node 崩溃了 master control plane 就需要参与进来了，这时候就需要下面章节的知识点 **



**ReplicationControllers**

RC 主要是做什么的？RC 的职责就是确保确切数量的 Pod 总是和 label 要求的个数进行匹配的。举个例子，如果 GPU 类型的 label 要求 5 个，那么 RC 会根据这个 label 来调整。如果说少了，就新增，多了就删除。

![image-20211222112949650](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211222112949650.png)



RC 就是一种 k8s 资源用来确保 pod 总是运行的，如下图所示，RC 将崩溃的Pod又拉了起来，即使Pod所在的Node也崩溃了。

![image-20211222111717870](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211222111717870.png)



但是，Pod A 由于没有被 RC 所管理，Node 崩溃后 Pod A 就直接丢失了。



什么情况下 Pod 副本数量超过了我们的预期？

- 手动创建相同类型 Pod
- 修改了现有的 Pod 类型
- 删除了现有的 Pod 数量



**组成 ReplicationController 的三个重要部分**

- label selector 确定 Pod 在 RC 中的范围
- replica count 指定了希望运行的 pod 数量
- pod template 用来创建新 Pod 的模板

![image-20211222113445164](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211222113445164.png)

**使用 RC 的好处**

- RC 总可以保证 Pod 实在运行的
- 当 node 崩溃的时候也可以将 node 上的 pod 进行恢复
- 支持水平扩展 Pod



(**在新版本的 K8s 中，已经不推荐使用 ReplicationController 了，这里做一个简单的了解即可**)



**ReplicaSets** 是 RC 的一个优化版本，但是实际上并不直接对 ReplicaSet 操作，而是通过 Deployment



**ReplicaSets Vs ReplicationControllers**

实际上他俩没什么区别，但是 RS 有更丰富的 selector，主要就是这两种 Kind 所支持的label 查询方式不同。

还有一种 kind 是，DaemonSet 这种类型，只运行一个 pod 在一个 node 上，以上两种都是讲 Pod 随机的散落在集群的各个位置



**Job**

执行完某项任务就退出，（战略性越过）



## 5-Service：enabling clients to discover  and talk to pods

![image-20211223142954084](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211223142954084.png)

Service 起到的作用像是一个反向代理，提供了负载均衡的功能，接下来就是 Service 的创建，查看。



**Service 创建**

如何确定哪些 pod 属于某个service，哪些 pod 不属于 service？通过 label 的方式来标识，如下图：

![image-20211223143403796](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211223143403796.png)

使用 exec 命令从内部发起请求 `kubectl exec pod_name -- curl -s http://xxx` `--` 双杠代表着 kubectl 命令的结束，如下如展示了这条命令的执行过程：

![image-20211223164517595](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211223164517595.png)



执行上述命令多次后会发现，每次命中的 pod 都是随机的，这里如果想要 client 每次命中相同的 pod （server）需要用到 sessionAffinity。就像 session 一样，service proxy 每次都会将 client 的请求重定向到同一个 pod 上。	



**创建一个支持多个端口的 Service**

![image-20211224161040566](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211224161040566.png)

注：不同的端口要有不同的名字，且这个两个端口都属于同一个 Service 中（label相同）。



**使用命名端口的好处**

比如我们 service 中 80 映射到 8080 端口，使用命名端口带来的好处就是，如果我们的 Pod 的端口修改了，并不影响 service，即 service 不需要修改。



**发现 service **

问题：客户端的是怎么知道 service 的 Ip 和 port 的？难道需要先创建 service 然后再创建 pod 嘛？

- 一种是通过环境变量，service的信息存储在 pod 中
- 一种是通过 DNS 查询 service Ip

kube-system 命名空间下的 coredns pod 为我们提供服务，该 pod 有所有的 service 信息。

关于 resolv.conf 的解释：https://man7.org/linux/man-pages/man5/resolv.conf.5.html



问题：为什么不能直接访问 service 服务？（为什么不能 ping service？）

因为 service IP 只是一个虚拟 IP，只有在结合 port 进行使用的时候 才有意义。



**连接集群之外的服务（别的应用）**

**service endpoint**

endpoint 是存在于 service 与 pod 之间的一个资源。如下所示，Endpoints 是一个描述了 Pod 的 IP 和 Port 的列表。

![image-20211228162028128](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211228162028128.png)



**手动配置 service endpoint**

当要手动配置 endpoint 的时候，创建 service 时就不需要指定 selector 了，所以也不会有任何的 Pod 的加入到当前的 service 中。还需要我们手动创建一个 endpoint。

![image-20211228193514466](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211228193514466.png)

![image-20211228193525422](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211228193525422.png)



**将service暴露给用户**

![image-20211228193804524](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211228193804524.png)

- service type 设置为一下几种类型时可以将 service 设置成从外部访问

  - NodePort

    ![image-20211228221529786](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211228221529786.png)

  - LoadBalance

    注意：使用 curl 命令和通过浏览器访问 lb ，会有一个奇怪的现象。按道理来说 LB 会将客户端的请求转发到某个特定的 pod，但是通过 curl 请求 LB 时，会出现命中不同 Pod 的情况。
  
    解释：通过浏览器访问的时候，使用了 keep-alive 机制，所以每次请求发送走的都是同一个链接；curl 每次都是新建一个链接，
  
    ![image-20211229103947424](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211229103947424.png)
  
    为了减少“跳”的次数，可以设置，externalTrafficPolicy: Local，这样每次外部客户端链接进来的时候，都不会跨越 Node。比如这次命中的是 Node A，那只能在 Node A 中选择一个 Pod 给客户端提供服务。但是缺点就如下图所示，会造成各个 Pod 负载不均衡。
  
    ![image-20211229111114391](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211229111114391.png)
  
  - Ingress resource

​			  ![image-20211229134758873](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211229134758873.png)

Ingress 处在网络拓扑结构中的应用层，可以提供基于会话的功能，cookie-session.



**readiness probes**

用来通知 pod 是否可以正常提供服务了，在某些情况下，pod 可能还没准备好，service 就把请求转发了过来，可能会导致请求有延迟，也可能会造成请求失败。

下图的情况就是 readiness probe 失败的情况：

![image-20211229162534978](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211229162534978.png)



**定位问题**

当我们通过 service 连接不上 Pod 的时候，可以通过以下几个步骤进行排查：（暂时省略）



## 6-Volumes: attaching disk storage to containers



