---
title: k8s 存储内容
comments: true
---

总结 k8s 存储相关知识点，以及在实践中遇到的问题。

<!--more-->

## Volume 概念

物理机上大家共用同一套文件系统，但是在容器内，每个容器的镜像都提供了一套文件系统，它们之间之相互独立的。容器运行时可以对系统中的文件进行修改，但是当他们运行结束后，container 被销毁了（进程没了，进程中的内容也不会存在了），他修改的东西也就都没了



想让数据持久存储的方法就是给 POD 挂载卷，绑定到 POD 内的容器中。所以不要再傻傻的以为卷是挂给 POD 的，严格的说是挂载给运行中的容器。

<img src="https://s2.loli.net/2022/07/04/ybZlBV1AJSiQv7R.png" style="zoom:80%;" />

目前为止我们提到的都是容器重启，现有的方案可以解决容器重启后数据丢失的问题，那么如上图中的结构，Pod 进行重启后，数据还可以进行持久化存储吗？



**不能的** ，在上图中，Volume 和 Pod 是共享生命周期的，我们需要一种能够独立于 Pod  生命周期的存储类别。



## External Storage Volume

外部存储，独立于 Pod 的生命周期。即使 Pod 被调度到了其他的工作节点，依然可以连接到我们的外部存储卷，相比于 hostpath 的好处。

![](https://s2.loli.net/2022/07/04/2a1bxNSVTweWKns.png)

常见的卷类别：

- hostPath 独立于 Pod 生命周期
- emptyDir 与 Pod 生命周期相同
- gcePersistentDisk Google 的持久卷
- cephfs 
- configMap 一般用于挂载文件的，通常是配置文件
- secret 一般用于存储重要信息



## PV

顾名思义，PersistentVolume 对象代表一个用于持久化应用程序数据的存储卷。没有讲 PV 对象前，我们挂载外部存储的的时候，需要指定具体的存储类别，比如使用 Google 的存储卷，当我们需要将 Pod 部署到 AWS 中的集群时，就需要重新修改这个 YAML 文件。

![](https://s2.loli.net/2022/07/04/DhfS5CRILcmW9se.png)



PV 是 K8s 提供给我们的抽象，在 Pod 和底层存储技术之间解耦。

![](https://s2.loli.net/2022/07/04/71qz5ZdxpjWGIeF.png)

### 手动配置的 PV 的生命周期

![](https://s2.loli.net/2022/07/05/I8RzS5uBNakKAs3.png)

1. 现有配置好底层存储
2. 创建 PV 对象，指向底层存储，PV 状态为 Available
3. 创建 PVC 指向刚刚创建的 PV, PVC -- PV 绑定到一起
4. 删除 PVC， PVC -- PV 之间解绑，PV 状态为 Released
5. 重新创建 PV 可使 PV 状态变成 Available



## PVC

Pod 并不会直接引用 PV，而是通过 PVC ，然后 PVC 再对应一个 PV 的方式，正如上图描述的，使用 PV PVC 的最大好处就是将特定于底层存储的细节与 pod 所代表的应用程序分离。



## 使用

![](https://s2.loli.net/2022/07/04/dMFyi8ncsQLl3o6.png)

从这张图可以看出，PV 是给集群管理员使用的，PVC 是给用户使用的。如果安装了动态分配的插件，那我们就可以直接通过创建 PVC 的方式。



卷的访问方式：

| Access Mode      | Abbr.  | Description                                                  |
| ---------------- | ------ | ------------------------------------------------------------ |
| `ReadWriteOnce ` | `RWO ` | The volume can be mounted by a single worker node in read/write mode. While it’s mounted to the node, other nodes can’t mount the volume. |
| `ReadOnlyMany `  | `ROX ` | The volume can be mounted on multiple worker nodes simultaneously in read-only mode. |
| `ReadWriteMany ` | `RWX ` | The volume can be mounted in read/write mode on multiple worker nodes at the same time. |

> 这里指的是 Node



PV 的回收策略：

| Reclaim policy | Description                                                  |
| -------------- | ------------------------------------------------------------ |
| `Retain `      | When the persistent volume is released (this happens when you delete the claim that’s bound to it), Kubernetes *retains* the volume. The cluster administrator must manually reclaim the volume. This is the default policy for manually created persistent volumes. |
| `Delete `      | The PersistentVolume object and the underlying storage are automatically deleted upon release. This is the default policy for dynamically provisioned persistent volumes, which are discussed in the next section. |
| `Recycle `     | This option is deprecated and shouldn’t be used as it may not be supported by the underlying volume plugin. This policy typically causes all files on the volume to be deleted and makes the persistent volume available again without the need to delete and recreate it. |

动态分配默认回收策略是：delete

手动创建 PV 时默认回收策略是：retain

我们在任何时候都可以修改 PV 的回收策略，比如使用动态分配时，默认回收策略是 delete ，但是我们中途不想删除PVC的时候删除这个卷，可以将其修改为 retain。

> 如果是 Released 状态的 PV，我们修改其回收策略为 delete 时，保存退出后，PV 对象和底层的卷都会被删除。

![](https://s2.loli.net/2022/07/05/TPmMsjYJBEdbHqw.png)

上图中我将 PVC 删掉了，现在策略是 Retain 这时候 `kubectl edit ` 一下修改为 Delete。

```shell
persistentVolumeReclaimPolicy: Retain --> persistentVolumeReclaimPolicy: Delete
```



PV 的删除

除了上述情况，假如我们想在 PV 是 Bound 状态下删除，PV 可以被删掉吗？

```shell
[root@k8s]# kubectl delete pv hostvol
persistentvolume "hostvol" deleted
```

会一直卡在这里，Ctrl-C 取消这次操作，再次获取 PV 发现已经是 `Terminating` 状态了。尽管刚刚取消delete命令的执行，但是这个 delete 的操作会被记录在 K8S 中（被释放后仍然会被删除）。



同样的，我们看看删除某个被 POD 引用的 PVC 是什么情况。

```sh
[root@k8s]# kubectl delete pvc hostpvc
persistentvolumeclaim "hostpvc" deleted
```

依然会一直卡在这里，同删除 PV 一样。



## 多个 POD 挂载 ReadWriteOnce 卷

这两天脑子里也一直在想这件事，我们挂载一个可读可写的卷给 POD，如果说这时候需要增加这个 POD  的副本，多个 POD 挂载同一个可读可写的卷，是否可以支持？



**答案是支持的** ，但是有前提，  **扩展出来的POD都必须在同一个 NODE 上** ，Once 指的是挂载到的 Node，只能给 Node 挂载一次，如果扩展的 POD 调度到其他 NODE，POD 是起不来的，会报如下错误：

```shell
$ kubectl describe po data-writer-97t9j
...
  Warning  FailedAttachVolume   ...   attachdetach-controller  AttachVolume.Attach failed 
for volume "other-data" : googleapi: Error 400: RESOURCE_IN_USE_BY_ANOTHER_RESOURCE -
The disk resource 'projects/.../disks/other-data' is already being used by
'projects/.../instances/gkdp-r6j4'
```

由于我这里机器就俩节点，复现不出来，直接把书中内容粘贴出来了。



如果存储的访问模式为 RWX 就说明这个卷可以被多个 Node 绑定，那这时候就对 Pod 的亲和策略没有约束了。



## 动态配置持久卷

先创建 PV ，再创建 PVC 的方式有些繁琐，我这里遇到过一个问题，在对接 ceph 的过程中，我想创建一个 PV 对象指向这个底层的卷，当时需要填一个 `volumeHandle` 对应的值，是 ceph 卷的 ID。对于并不知道怎么操作 ceph 的我来说是比较困难的，需要先创建好底层卷，然后在PV中指定才可以。



但是，通过动态配置的方式就可以解决这个问题，我们不再需要手动准备好存储卷以及PV对象，只需写一个“需求文档”。

### 动态配置流程

![](https://s2.loli.net/2022/07/05/rkYKoNBRMi3nUjV.png)



### StorageClass 与 PVC 的关系

![](https://s2.loli.net/2022/07/11/dQRJFDbnCAuyk5x.png)

通常来讲，不同卷类型对应的不同的底层存储技术，亦或是对应不同的 `provisioner`。



### 卷绑定模式

讲的就是我们创建 PVC 对象后，PV  何时被创建。

| Volume binding mode     | Description                                                  |
| ----------------------- | ------------------------------------------------------------ |
| `Immediate `            | The provision and binding of the persistent volume takes place immediately after the claim is created. Because the consumer of the claim is unknown at this point, this mode is only applicable to volumes that are can be accessed from any cluster node. |
| `WaitForFirstConsumer ` | The volume is provisioned and bound to the claim when the first pod that uses this claim is created. This mode is used for topology-constrained volume types. |

需要 `WaitForFirstConsumer ` 的原因是，使用 Local 这种卷类型的时候，不知道 POD 被调度到哪里，所以需要确定好 POD 位置后再创建 PV。

### 动态绑定的声明周期

![](https://s2.loli.net/2022/07/06/5ABInxbTrwCLl7v.png)



### 卷扩容

需要 POD 配合（重启）来完成，而且还需要注意底层存储是否支持。



### Retain 状态的 PV 怎么恢复为 Available

本来以为 k8s 对接持久化存储已经结束了，突然发现 Rancher 上并不支持，修改 PV 的属性。想要从 Retain 状态修改为 Available 有两种方式：

- 重新创建这个 PV 对象
- edit 这个 PV 对象删除 claimref 部分

![](https://s2.loli.net/2022/07/11/4JN2AMPanYqQ3bV.png)

 将 PVC 对象释放后，获取到的 PV 列表，既然 Rancher 这两种方法都不支持，只能另寻他路了。

> **delete and recreate pv object 会导致底层卷被删除吗？ **
>
> static provisioning:
>
> - pv 只是对底层存储的一个引用，删除 pv 并不会删除底层卷。
>
> dynamic provisioning: 
>
> - delete，pvc 被删除后 pv 和底层存储卷都被删除
> - retain，pvc 被删除后，pv 是 released 状态，删除 pv 不会导致底层卷被删除（已在 ceph 存储上验证）
>
>
> 所以除了在动态配置的回收策略为删除的情况下，删除PV都不会导致底层存储卷会被删除。
