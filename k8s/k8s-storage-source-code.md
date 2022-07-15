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

还有一点需要注意的是，存储插件在 kube controller manager 启动的时候就初始化好了，当我们使用外部存储的时候，集群怎么找到它？

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



所以我这里的场景应该是 plugin 返回 nil：

```go
func (ctrl *PersistentVolumeController) findProvisionablePlugin(claim *v1.PersistentVolumeClaim) (vol.ProvisionableVolumePlugin, *storage.StorageClass, error) {
    // provisionClaim() which leads here is never called with claimClass=="", we
    // can save some checks.
    claimClass := storagehelpers.GetPersistentVolumeClaimClass(claim)
    class, err := ctrl.classLister.Get(claimClass)
    if err != nil {
        return nil, nil, err
    }

    // Find a plugin for the class
    if ctrl.csiMigratedPluginManager.IsMigrationEnabledForPlugin(class.Provisioner) {
        // CSI migration scenario - do not depend on in-tree plugin
        return nil, class, nil
    }
    plugin, err := ctrl.volumePluginMgr.FindProvisionablePluginByName(class.Provisioner)
    if err != nil {
        // 不是以 kubernetes.io/ 开头的
        if !strings.HasPrefix(class.Provisioner, "kubernetes.io/") {
            // External provisioner is requested, do not report error
            return nil, class, nil
        }
        return nil, class, err
    }
    return plugin, class, nil
}
```



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

> 一开始这块确实没看懂，他们把逻辑写在定时任务中，同一个对象在不同的周期会走到不同的逻辑。

在 PVC 创建阶段我们主要关注这个地方：

```go
// syncClaim is the main controller method to decide what to do with a claim.
// It's invoked by appropriate cache.Controller callbacks when a claim is
// created, updated or periodically synced. We do not differentiate between
// these events.
// For easier readability, it was split into syncUnboundClaim and syncBoundClaim
// methods.
func (ctrl *PersistentVolumeController) syncClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
    klog.V(4).Infof("synchronizing PersistentVolumeClaim[%s]: %s", claimToClaimKey(claim), getClaimStatusForLogging(claim))

    // Set correct "migrated-to" annotations on PVC and update in API server if
    // necessary
    newClaim, err := ctrl.updateClaimMigrationAnnotations(ctx, claim)
    if err != nil {
        // Nothing was saved; we will fall back into the same
        // condition in the next call to this method
        return err
    }
    claim = newClaim
	
    // 这里只区分了两种状态的 pvc 未绑定，已绑定
    // 如果是已绑定的 pvc 会被打上 pv.kubernetes.io/bind-completed 的标签
    if !metav1.HasAnnotation(claim.ObjectMeta, storagehelpers.AnnBindCompleted) {
        return ctrl.syncUnboundClaim(ctx, claim)
    } else {
        return ctrl.syncBoundClaim(claim)
    }
}
```

未绑定的 PVC 自然走的就是 `ctrl.syncUnboundClaim` 处理函数：

```go
// syncUnboundClaim is the main controller method to decide what to do with an
// unbound claim.
func (ctrl *PersistentVolumeController) syncUnboundClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
    // pvc is "Pending"
    if claim.Spec.VolumeName == "" {
        // 绑定模式，waitforfirstcustomer or immediate
        delayBinding, err := storagehelpers.IsDelayBindingMode(claim, ctrl.classLister)
        if err != nil {
            return err
        }

        // 查一下是否有存在并且满足 PVC 中条件的 PV
        volume, err := ctrl.volumes.findBestMatchForClaim(claim, delayBinding)
        if err != nil {
            klog.V(2).Infof("synchronizing unbound PersistentVolumeClaim[%s]: Error finding PV for claim: %v", claimToClaimKey(claim), err)
            return fmt.Errorf("error finding PV for claim %q: %w", claimToClaimKey(claim), err)
        }
        if volume == nil {
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: no volume found", claimToClaimKey(claim))
            // No PV could be found
            // OBSERVATION: pvc is "Pending", will retry
            switch {
                // 延迟绑定
                case delayBinding && !storagehelpers.IsDelayBindingProvisioning(claim):
                    if err = ctrl.emitEventForUnboundDelayBindingClaim(claim); err != nil {
                        return err
                    }
                // 获取 pvc 的 storageclass
                case storagehelpers.GetPersistentVolumeClaimClass(claim) != "":
                    if err = ctrl.provisionClaim(ctx, claim); err != nil {
                        return err
                    }
                    return nil
                default:
                	ctrl.eventRecorder.Event(claim, v1.EventTypeNormal, events.FailedBinding, "no persistent volumes available for this claim and no storage class is set")
            }

            // Mark the claim as Pending and try to find a match in the next
            // periodic syncClaim
            if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                return err
            }
            return nil
        } else /* pv != nil */ {
            // Found a PV for this claim
            // OBSERVATION: pvc is "Pending", pv is "Available"
            claimKey := claimToClaimKey(claim)
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q found: %s", claimKey, volume.Name, getVolumeStatusForLogging(volume))
            if err = ctrl.bind(volume, claim); err != nil {
                // On any error saving the volume or the claim, subsequent
                // syncClaim will finish the binding.
                // record count error for provision if exists
                // timestamp entry will remain in cache until a success binding has happened
                metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, err)
                return err
            }
            // OBSERVATION: claim is "Bound", pv is "Bound"
            // if exists a timestamp entry in cache, record end to end provision latency and clean up cache
            // End of the provision + binding operation lifecycle, cache will be cleaned by "RecordMetric"
            // [Unit test 12-1, 12-2, 12-4]
            metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, nil)
            return nil
        }
    } else /* pvc.Spec.VolumeName != nil */ {
        // [Unit test set 2]
        // User asked for a specific PV.
        klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested", claimToClaimKey(claim), claim.Spec.VolumeName)
        obj, found, err := ctrl.volumes.store.GetByKey(claim.Spec.VolumeName)
        if err != nil {
            return err
        }
        if !found {
            // User asked for a PV that does not exist.
            // OBSERVATION: pvc is "Pending"
            // Retry later.
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and not found, will try again next time", claimToClaimKey(claim), claim.Spec.VolumeName)
            if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                return err
            }
            return nil
        } else {
            volume, ok := obj.(*v1.PersistentVolume)
            if !ok {
                return fmt.Errorf("cannot convert object from volume cache to volume %q!?: %+v", claim.Spec.VolumeName, obj)
            }
            klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume %q requested and found: %s", claimToClaimKey(claim), claim.Spec.VolumeName, getVolumeStatusForLogging(volume))
            if volume.Spec.ClaimRef == nil {
                // User asked for a PV that is not claimed
                // OBSERVATION: pvc is "Pending", pv is "Available"
                klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume is unbound, binding", claimToClaimKey(claim))
                if err = checkVolumeSatisfyClaim(volume, claim); err != nil {
                    klog.V(4).Infof("Can't bind the claim to volume %q: %v", volume.Name, err)
                    // send an event
                    msg := fmt.Sprintf("Cannot bind to requested volume %q: %s", volume.Name, err)
                    ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.VolumeMismatch, msg)
                    // volume does not satisfy the requirements of the claim
                    if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                        return err
                    }
                } else if err = ctrl.bind(volume, claim); err != nil {
                    // On any error saving the volume or the claim, subsequent
                    // syncClaim will finish the binding.
                    return err
                }
                // OBSERVATION: pvc is "Bound", pv is "Bound"
                return nil
            } else if storagehelpers.IsVolumeBoundToClaim(volume, claim) {
                // User asked for a PV that is claimed by this PVC
                // OBSERVATION: pvc is "Pending", pv is "Bound"
                klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound, finishing the binding", claimToClaimKey(claim))

                // Finish the volume binding by adding claim UID.
                if err = ctrl.bind(volume, claim); err != nil {
                    return err
                }
                // OBSERVATION: pvc is "Bound", pv is "Bound"
                return nil
            } else {
                // User asked for a PV that is claimed by someone else
                // OBSERVATION: pvc is "Pending", pv is "Bound"
                if !metav1.HasAnnotation(claim.ObjectMeta, storagehelpers.AnnBoundByController) {
                    klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim by user, will retry later", claimToClaimKey(claim))
                    claimMsg := fmt.Sprintf("volume %q already bound to a different claim.", volume.Name)
                    ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.FailedBinding, claimMsg)
                    // User asked for a specific PV, retry later
                    if _, err = ctrl.updateClaimStatus(claim, v1.ClaimPending, nil); err != nil {
                        return err
                    }
                    return nil
                } else {
                    // This should never happen because someone had to remove
                    // AnnBindCompleted annotation on the claim.
                    klog.V(4).Infof("synchronizing unbound PersistentVolumeClaim[%s]: volume already bound to different claim %q by controller, THIS SHOULD NEVER HAPPEN", claimToClaimKey(claim), claimrefToClaimKey(volume.Spec.ClaimRef))
                    claimMsg := fmt.Sprintf("volume %q already bound to a different claim.", volume.Name)
                    ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.FailedBinding, claimMsg)

                    return fmt.Errorf("invalid binding of claim %q to volume %q: volume already claimed by %q", claimToClaimKey(claim), claim.Spec.VolumeName, claimrefToClaimKey(volume.Spec.ClaimRef))
                }
            }
        }
    }
}
```

逻辑上其实一点难度都没，注释写的也很详细（我想加注释的时候发现有点冗余），错误处理方式没有任何简化，官方称为：`space shuttle style`，而且还有一点值得学习的地方：

```go
if volume == nil {
    // logic..
} else /* pv != nil */ { 
    // logic..
}
```

在 else 中填写注释，当我们把 if 代码块折叠起来时，就是这样的：

```go
if volume == nil {} else /* pv != nil */ {}
```

个人觉得这种注释方式真的挺友好，当 if 代码块中逻辑特别长时，可能就忘了其中的条件判断..看到 else 时，直接就一脸懵，这是哪个 if 的 else..



然后我们接着看配置 PVC 的部分：

```go
// provisionClaim starts new asynchronous operation to provision a claim if
// provisioning is enabled.
func (ctrl *PersistentVolumeController) provisionClaim(ctx context.Context, claim *v1.PersistentVolumeClaim) error {
    if !ctrl.enableDynamicProvisioning {
        return nil
    }
    klog.V(4).Infof("provisionClaim[%s]: started", claimToClaimKey(claim))
    opName := fmt.Sprintf("provision-%s[%s]", claimToClaimKey(claim), string(claim.UID))
	// 外部存储情况 plugin == nil
    plugin, storageClass, err := ctrl.findProvisionablePlugin(claim)
    // findProvisionablePlugin does not return err for external provisioners
    if err != nil {
        ctrl.eventRecorder.Event(claim, v1.EventTypeWarning, events.ProvisioningFailed, err.Error())
        klog.Errorf("error finding provisioning plugin for claim %s: %v", claimToClaimKey(claim), err)
        // failed to find the requested provisioning plugin, directly return err for now.
        // controller will retry the provisioning in every syncUnboundClaim() call
        // retain the original behavior of returning nil from provisionClaim call
        return nil
    }
    ctrl.scheduleOperation(opName, func() error {
        // create a start timestamp entry in cache for provision operation if no one exists with
        // key = claimKey, pluginName = provisionerName, operation = "provision"
        claimKey := claimToClaimKey(claim)
        ctrl.operationTimestamps.AddIfNotExist(claimKey, ctrl.getProvisionerName(plugin, storageClass), "provision")
        var err error
        // 第一次处理PVC走这儿，主要为了把 provisioner 填到 PVC 中。
        if plugin == nil {
            _, err = ctrl.provisionClaimOperationExternal(ctx, claim, storageClass)
        } else {
            _, err = ctrl.provisionClaimOperation(ctx, claim, plugin, storageClass)
        }
        // if error happened, record an error count metric
        // timestamp entry will remain in cache until a success binding has happened
        if err != nil {
            metrics.RecordMetric(claimKey, &ctrl.operationTimestamps, err)
        }
        return err
    })
    return nil
}
```
