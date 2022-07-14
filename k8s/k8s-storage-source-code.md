---
title: k8s pv pvc 源码
comments: true
---

最近在做集群对接存储的工作，期间经历了很多挫折，遇到了很多问题，从一知半解到了解，从纸上谈兵到实战演练，环环相扣。在开始前我先抛出两个问题，带着问题去思考收获可能会更多。

- 动态配置中，创建 PVC 后，是先有 PV 还是现有 Volume
- PV 与 PVC 怎么样才算绑定到一起

<!--more-->

从 *Dynamic Provisioning Storage* 搞起，基本概念前面文章都有提到过 [k8s-storage](./k8s-storage.md) 所以在次就不再赘述。

负责 `PVC PV` 的控制器是 PersistentVolumeController，不过我们不着急直接看这个控制器，先简单了解下它的“父控制器”，kube controller manager。

```go
/*

https://kubernetes.io/docs/concepts/architecture/controller/

The Kubernetes controller manager is a daemon that embeds the core 
control loops shipped with Kubernetes. In applications of robotics and
automation, a control loop is a non-terminating loop that regulates the
state of the system. In Kubernetes, a controller is a control loop that
watches the shared state of the cluster through the apiserver and makes
changes attempting to move the current state towards the desired state.
Examples of controllers that ship with Kubernetes today are the replication
controller, endpoints controller, namespace controller, and serviceaccounts
controller.

简言之，就是一个“死”循环用来管理集群中需要共享资源的状态
*/
```

除了注释中列出的 controller 还有很多

```go
// ...
controllers["persistentvolume-binder"] = startPersistentVolumeBinderController
controllers["attachdetach"] = startAttachDetachController
controllers["persistentvolume-expander"] = startVolumeExpandController
controllers["pvc-protection"] = startPVCProtectionController
controllers["pv-protection"] = startPVProtectionController
// ...
```

然后在主控制器中会启动一个 goroutine 用来执行 volume manager

```go
volumeController, volumeControllerErr := persistentvolumecontroller.NewController(params)
if volumeControllerErr != nil {
    return nil, true, fmt.Errorf("failed to construct persistentvolume controller: %v", volumeControllerErr)
}
go volumeController.Run(ctx)
```

这个 manager 我认为就做了两件事情，处理 `pv pvc` 如下：

```go
// 处理 pv
go wait.UntilWithContext(ctx, ctrl.volumeWorker, time.Second)
// 处理 pvc
go wait.UntilWithContext(ctx, ctrl.claimWorker, time.Second)
```

还有一点需要注意的是，存储插件在 kube controller manager 启动的时候就初始化好了，当我们使用外部存储的时候，集群是怎么和他进行交互呢？

```sh
# 内容来自 kube controller manager 日志
Loaded volume plugin "kubernetes.io/host-path"
Loaded volume plugin "kubernetes.io/nfs"
Loaded volume plugin "kubernetes.io/glusterfs"
Loaded volume plugin "kubernetes.io/rbd"
Loaded volume plugin "kubernetes.io/quobyte"
Loaded volume plugin "kubernetes.io/aws-ebs"
Loaded volume plugin "kubernetes.io/gce-pd"
Loaded volume plugin "kubernetes.io/cinder"
Loaded volume plugin "kubernetes.io/azure-disk"
Loaded volume plugin "kubernetes.io/azure-file"
Loaded volume plugin "kubernetes.io/vsphere-volume"
Loaded volume plugin "kubernetes.io/flocker"
Loaded volume plugin "kubernetes.io/portworx-volume"
Loaded volume plugin "kubernetes.io/scaleio"
Loaded volume plugin "kubernetes.io/local-volume"
Loaded volume plugin "kubernetes.io/storageos"
Loaded volume plugin "kubernetes.io/csi"
```

可以看到初始化的插件中有一个 kubernetes.io/csi 这个是 k8s 为外部存储提供的统一接口，我认为在这里是可以理解成依赖倒转原则的。本来 k8s 集群要使用存储功能，拥有存储功能的类别可能有很多，为了消除这种集群与特定的存储类别之间的耦合关系，抽象出来一层 CSI。



我一开始使用内置的 `kubernetes.io/rbd` 对接 ceph ，但是会出现 pvc 无限处于 pending 状态的问题，**最难受的是 events 中没有任何信息**，后来运维同事帮忙排查出来是内置在 kube controller manager 容器中的 ceph 客户端版本问题。所以，就采用了实现了 CSI 接口的 rbd 插件 `rbd.csi.ceph.com` 也就对应到 kubernetes.io/csi。



现在知道它是走的 CSI 插件创建卷，仅知道这些还不够，既然已经走到源码部分了，还得再往深处挖。



前面启动了两个 worker 一个用来处理 PV，另一个用来处理 PVC，他俩分别对应一个队列，去消费队列中的内容。那生产者是谁呢，是谁把内容放到这个队列中的呢？

```go
// resync supplements short resync period of shared informers - we don't want
// all consumers of PV/PVC shared informer to have a short resync period,
// therefore we do our own.
func (ctrl *PersistentVolumeController) resync() {
    klog.V(4).Infof("resyncing PV controller")

    pvcs, err := ctrl.claimLister.List(labels.NewSelector())
    if err != nil {
        klog.Warningf("cannot list claims: %s", err)
        return
    }
    for _, pvc := range pvcs {
        ctrl.enqueueWork(ctrl.claimQueue, pvc)
    }

    pvs, err := ctrl.volumeLister.List(labels.NewSelector())
    if err != nil {
        klog.Warningf("cannot list persistent volumes: %s", err)
        return
    }
    for _, pv := range pvs {
        ctrl.enqueueWork(ctrl.volumeQueue, pv)
    }
}
```

看到如上代码就很清晰了，直接调用 pv pvc 的 list 接口，将获取到的内容入队。启动生产者的地方和启动消费者挨着。

```go
go wait.Until(ctrl.resync, ctrl.resyncPeriod, ctx.Done())
go wait.UntilWithContext(ctx, ctrl.volumeWorker, time.Second)
go wait.UntilWithContext(ctx, ctrl.claimWorker, time.Second)
```

剩下的就是研究这个消费者的处理逻辑是什么，以动态配置创建 PVC 为例。

