---
title: Deployment 更新策略
comment: true
---

将笔记进行拆分，都写在一个里边很拥挤，不方便翻阅查看。

<!--more-->

# Deployment

## 更新策略

| 更新策略      | 描述                                                         |
| ------------- | ------------------------------------------------------------ |
| Recreate      | 见名知意，更新的时候就是把已存在的 Pod 删除，创建一个新的    |
| RollingUpdate | 滚动更新，先准备新的 Pod，就绪后再删除老的 Pod，在更新期间也可以正常提供服务。**默认配置**。 |

![img](https://s2.loli.net/2022/05/09/uwBeEpl6Gq41WkC.png)

## Recreate 如何影响应用

在重启期间，无法正常访问服务，正如上图中会有一段空档期。



## Deployment 和 Replicas 的关系

![img](https://s2.loli.net/2022/05/09/cmYjDQrSAguyaN3.png)

正如我们看到的，`Deployment` 和 `Replica` 并不是总是一对一的关系，在更新操作执行后，会出现上图中的情况。



## RollingUpdate

![img](https://s2.loli.net/2022/05/09/GzleTsRbmFaNLwI.png)

### Deployment 配置

```yaml
kind: Deployment
metadata:
  name: kiada
spec:
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 0
      maxUnavailable: 1
  minReadySeconds: 10
  replicas: 3
  selector:
    ...
```

### 配置一次替换多少个 Pod

通过 `maxSurge` 和 `maxUnavailable` 配置

| 属性           | 描述                                                         |
| -------------- | ------------------------------------------------------------ |
| maxSurge       | The maximum number of Pods above the desired number of replicas that the Deployment can have during the rolling update. The value can be an absolute number or a percentage of the desired number of replicas.<br /><br />在滚动更新期间 Deployment 可以超过 replica 中最大的 Pod 数量。 |
| maxUnavaliable | The maximum number of Pods relative to the desired replica count that can be unavailable during the rolling update. The value can be an absolute number or a percentage of the desired number of replicas.<br /><br />最少可用的 Pod 数量为 total - maxUnavailable，最多不可用的 Pod 数量为 maxUnavailable |

 `maxSurge` 和 `maxUnavailable` 会影响 Replica 中期望的 Pod 数量。



**maxSurge = 0, maxUnavailable = 1**

![img](https://s2.loli.net/2022/05/09/toFLwvXl5HuqObE.png)

**maxSurge = 1, maxUnavailable = 0**

![img](https://s2.loli.net/2022/05/09/cE5UJVdDAa8mkrg.png)

**maxSurge = 1, maxUnavailable = 1**

![img](https://s2.loli.net/2022/05/09/mnLPUaQyldEHxj3.png)

### 暂停、恢复 RollingUpdate

`kubectl rollout pause deployment kiada` 和 `kubectl rollout resume deployment kiada`

这个骚操作还是第一次听说..



### minReadySeconds

在 RollingUpdate 过程中，新创建的 Pod Ready 后，**还不算结束**，要等待 `minReadySeconds` 这么久后， Pod 才会变为 available 状态。



默认情况下是 0，Pod Ready 了就算 available.


### 检查 rollout 是否在执行

`kubectl describe deployment xxx`



### 回滚 Rolling Back

- 回滚到上一版本

  `kubectl rollout undo deployment kiada`

> undo 命令也可以在 RollingUpdate 过程中使用，用来取消这个升级



- 回滚到指定版本

  `kubectl rollout undo deployment kiada --to-revision=1`



### 回滚和使用 yaml 文件恢复的区别

- 回滚使用 `kubectl rollout undo` 命令，只是恢复 pod-templete 内容
- 使用 apply -f 的方式会将所有的内容进行恢复

比如我们在 1.2 版本中修改了升级方式为 RollingUpdate，1.1 仍然为 Recreate，这时候如果我们通过，apply 的方式，会将我们改的内容进行覆盖。



一般情况下，都是用 `kubectl rollout undo` 命令，之前没学到这个命令，在操作更新版本的时候，都是 edit deployment 然后修改一下 镜像的版本，出了问题也是简单粗暴的处理，直接将版本恢复回去。



下次上线升级的时候，可以考虑使用这种方式，两个字，优雅。



### 显示 rollout 历史

`kubectl rollout history deploy kiada`

- 查看某次升级的具体信息

  `kubectl rollout history deployment kiada --revision 2`

  查看 kiada 这个 Deployment 在 revision 为 2 的版本做了哪些调整。

![img](https://s2.loli.net/2022/05/09/XWgT2rFj7blSqU9.png)

- `revisionHistoryLimit` 用来配置保存多少个历史记录，默认是 10。



## Traffic shadowing

（这个应该翻译成啥呢？）

![img](https://s2.loli.net/2022/05/09/Pu9zvCGAU4e1dZy.png)

利用 Ingress 进行流量复制，将请求转发一份到新的版本中，然后忽略掉响应。



## 完