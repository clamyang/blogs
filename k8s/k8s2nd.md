---
title: k8s 2nd
comment: true
---

各个部分及其字段的含义。

<!--more-->

![image-20220302162130087](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220302162130087.png)

![image-20220302162138717](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220302162138717.png)



更通俗的方式：

`kubectl explain nodes`

可以查看 node 的相关信息。



一个 pod 不应该运行多个容器：

- 比如多个进程都会输出日志到控制台，会出现日志混乱。
- container runtime 指挥在 root 进程崩溃的时候进行重启，并不关心子进程。
- 如果在一个 pod 中运行多个 container ，水平扩展的时候，会根据 pod 进行扩展，而不是根据 pod 中的 container 进行扩展。



什么时候需要一个 pod 运行多个容器？

![image-20220303135731675](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220303135731675.png)



怎么确定是需要 sidecar 还是要 separate 模式？

• Do these containers have to run on the same host? 

• Do I want to manage them as a single unit? 

• Do they form a unified whole instead of being independent components? 

• Do they have to be scaled together? 

• Can a single node meet their combined resource needs? 





只输出 yaml 文件 `kubectl run kubia --image=luksa/kubia:1.0 --dry-run=client -o yaml > mypod.yaml`

--dry-run=client 只输出定义不执行创建。



使用 port-forward 的执行过程。

![image-20220303152703523](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220303152703523.png)

pod 日志的相关操作：

- 查看日志 `kubectl logs kubia`
- 实时查看 `kubectl logs --follow kubia` 简版 `kubectl logs -f kubia`
- 查看最近两分钟的日志 `kubectl logs kubia --since=2m`
- 查看从某个时间点开始的日志 `kubectl logs kubia --since-time=xxxxxx`
- 查看倒数10行的日志 `kubectl logs kubia --tail=10`



日志文件存储在哪里了？

pod 中的container日志，通常会存储在 node 的 `/var/log/containers` 中，每一个容器对应上一个独立的文件。**如果 container 重启了，它的日志会写入到新的文件中**，想要查看以前的日志文件，使用 `--previous(-p)` 命令。

![image-20220303155702882](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220303155702882.png)

对应的日志文件是长这个样子的.. 里边存储的日志也是 json 格式的。

------

作者提到一个pod运行多个容器，现在要往一个运行容器的pod中再添加一个容器？

要怎么才能做到呢？



擦，感觉被骗了，人家后边是新创建一个 pod，新 pod  中包含两个 container。



分别查看pod中多个容器的日志

- `kubectl logs kubia-ssl -c kubia` **指定容器的名称**
- `kubectl logs kubia-ssl --all-containers` 查看所有容器的日志



### Pod 的生命周期

状态变化过程：

![image-20220304100100883](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220304100100883.png)



Pod condition 变化过程：

![image-20220304102134299](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220304102134299.png)



Pod 重启策略

![image-20220304134321707](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220304134321707.png)



Pod 重启中的指数回退算法：

防止 pod 频繁重启，所以增加了每次 pod 的重启间隔。

![image-20220304140257223](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220304140257223.png)

当容器正常运行 10 分钟之后，这个重置时间会被清零。



探活过程，及参数解释：

![image-20220304145015728](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220304145015728.png)



- initialDelaySeconds：延迟探测时间

- periodSeconds ：探测时间间隔

- timeoutSeconds ：响应超时时间

- failureThreshold ：失败次数



Pod 探活指针，如果说要通过 httpGet 进行实现，那么在 handler 中**不要做额外的重试工作**，这种就像 k8s 通过 failureThreshold  参数给你搞了一个 for 循环，for 一次执行一次你的 handler，然后你在 handler 中又添加了 for 循环，其实这样是没有必要的。



比起这样直接添加 failureThreshold  的值即可。



在 Pod **开始后，结束前**执行某些特定操作。

![image-20220308181524107](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220308181524107.png)



post-start AND pre-stop

> post-start  并不是开始后执行，而是容器创建了就开始执行。



pre-stop

当容器在初始化时候被终止，所有类型的探针都不会被执行。



**Pod 的声明周期**

![image-20220322180937014](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220322180937014.png)



imagePullPolicy：（init 阶段执行）

- Not specified 未指定的话，默认就是 always
- Always 每一次启动或重启容器，都拉取镜像。**如果有本地缓存，就不会重新拉取了**
- Never
- IfNotPresent 

![image-20220322181518924](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220322181518924.png)

拉一个 initContainer 镜像，执行一个初始化，然后再拉下一个 init 镜像，直到所有的 initContainer 执行完毕。

![image-20220322181920040](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220322181920040.png)



**Pod 的完整声明周期**

![image-20220322215904044](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220322215904044.png)

![image-20220322215918443](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220322215918443.png)



**为什么要给 Pod 挂卷**

为的就是重启的时候持久化数据。

> 卷的生命周期和pod的生命周期是保持一致的，卷的生命周期独立于容器（即容器重启了，并不影响卷。如果pod被删除了，卷也会被删除）。



卷的生命周期独立于 Pod 的情况，也就是挂载的卷不在 pod 内。

![image-20220324100102363](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220324100102363.png)



**emptyDir 卷类型**

从下图中可以看出，该类型的卷还是存储在了 Pod 所在的 Node 上

![image-20220324143444116](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220324143444116.png)

`/var/lib/kubelet/pods/<pod_UID>/volumes/kubernetes.io~empty-dir/<volume_name>`



hostPath 的用途？

通常来讲，Pod 被调度到哪个 Node 上是未知的，所以 hostPath 适用场景比较少。



当采用外部存储的时候，如果直接指定了存储卷的IP地址，那么在将 Pod 移动到另一个集群上时，比如从google移动到 Amazon，在 Google 的存储卷，在 Amazon 上不支持。



由此，k8s 引用了 PV(persistent volume)，抽象出来一个持久卷，从 yaml 直接指定存储卷，到间接引用 pv 对象，进行了引用间的解耦。

> Pod 并不会直接引用 PV，而是通过 PVC 。



PV 的三种访问模式

| 访问模式      | 缩写 | 描述                                                         |
| ------------- | ---- | ------------------------------------------------------------ |
| ReadWriteOnce | RWO  | 卷可以被一个 Worker Node 以读、写模式绑定。当他绑定到某个 node 时，不能被其他node 绑定。 |
| ReadOnlyMany  | ROX  | 卷可以被多个 node 以 读 的方式进行绑定。                     |
| ReadWriteMany | RWX  | 卷可以被多个 node 以 读/写的方式进行绑定。                   |



Pod 挂载到了 PVC，在删除的时候需要注意：

- 删除了 pod pvc 显示的仍然是已绑定状态。
- 删除 pvc 后，绑定的 pv 是 released 状态，claim 列仍然显示已经被删除的 pvc。

这是因为，pv 中可能保存了一些 application 存储在其中的数据，如果想要重复使用，需要将其中的数据清空。



想要 pv 可以重复使用，只有删除掉，然后重新创建。

> 这个不会影响 volume 中存储的数据

另一种方式是 edit pv 中的 claimRef，把这个 ref 删除掉， pv 的状态也会恢复为 available。



pv 的回收策略:

- retain
- delete
- recycle





使用 configMap 的三种方式

单条 key - value 的方式进行注入

将整个 configMap 进行注入

将 configMap 以 volume 的方式进行注入

- 通过 volume 的方式也可以指定特定的 key value，不需要卷下的所有内容。



secret

worker node 上，secret 中的内容是存储在内存中的，并不会存储在磁盘上，进一步增加安全性。



downwardAPI

主要是用来暴露 pod 的一些信息给 pod 中的 application



还提供了一种聚合了 configMap、secret、downwardAPI 的 volume 类型，projected



### service

如果两个 pod 需要进行通信，想要知道另一个 pod 的 ip 地址是很困难的，原因如下：

- pod 只有在调度到指定 node 上才会有 IP 地址
- pod 可能随时被替换，或者调度到其他的 node 上
- 水平扩展时，会出现很多相同服务的 pod



 更新 service selector

`kubectl set selector service service_name label_key=label_value`



external service LB

配置分发模式，externalTrafficPolicy:

- Local 只转发给 service 选择的 node，不会通过这个node传递给另一个node

- Cluster 如果说 service 选择的 node 没有我们想要的 pod 服务，那么会转发给下一个 node，需要额外的开销，下一跳，source ip 也会改成 转发 packet 的 node ip。



### Ingress

![12image002](https://s2.loli.net/2022/04/27/wJybIj2eWUGXvx3.png)

![12image003](https://s2.loli.net/2022/04/27/UOyvTbXYrnofKu8.png)

有了 service 还要 ingress 干啥？

如果说通过 service 对外提供服务，数量少还好，如果服务数量多，会需要占用很多外部IP，导致没必要的资源浪费。



可以通过 ingress 对 service 进行一层封装，这就好像我们不是直接访问 pod 一样，而是通过 service 访问一样。统一通过 ingress 访问我们集群体提供的服务，而且 ingress 是 7 层的，可以提供更多功能，比如 cookie TLS 等，相比之下 service 是 4 层的负载均衡。



所谓的 7，4 层负载均衡，都是在请求到达真正服务器之前做的一些处理，差别简单的说就是，在做负载均衡的时候，使用的信息不一样，在 7 层可以用 http 相关的东西，4 层可以用 TCP/UDP 相关的东西。



