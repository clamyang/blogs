---
title: git rebase
comments: true
---

以前合分支，用的都是 git merge 但是发现用 merge 合并总是会出现一条自动添加的 commit 这样导致历史提交记录显得非常乱。

<!--more-->

就比如下面这个样子

<img src="https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20211209110709842.png" alt="image-20211209110709842" style="zoom:67%; margin: 0 auto;"/>





没几个 commit 全都是 Merge branch xxxx into xxxx。有一次我在提交 pr 的时候，当时已经提交完了，但是显示我的分支是落后的，上边有个 merge 按钮，我点了以后就出现了这种“不干净的” commit。



在了解之后，通过 rebase 命令可以避免这种情况，所以简单看下 git rebase 是怎么玩的。



rebase  和 merge 都是用来整合一个分支到另一个分支上，但是主要的区别为：rebase 后产生的是一个线性的历史分支。可以结合实际使用场景举个例子：

假设，当前有一个 main 分支，我要开发一个查询功能，所以基于 main 新建了一个 feature 分支。当你在  feature 分支上开发完成后，与此同时，main 分支已经被更新了，你想在  feature 分支上获取 main 刚刚提交的 commit，这时候有两种方式

- merge
- rebase

想要以一种最“干净”的方式来合并，就好像你这个 branch 刚刚从 main 分支上新建出来的一样，这时候就要用到 git base。

（所谓的干净是什么意思呢，我理解的是，它看起来是干净的，并且在需要回滚的时候也是非常清除的。）

<img src="https://gitee.com/yangbaoqiang/images/raw/master/blogpics/v2-73db63a5abb3cac70f913155a854cf29_1440w.jpg" alt="v2-73db63a5abb3cac70f913155a854cf29_1440w" style="zoom:50%; margin: 0 auto;" />

如上图所示，主要的区别就是怎么对待以前的 commit 信息。



参考：

[知乎-丁哥开讲](https://zhuanlan.zhihu.com/p/75499871)

[bitbucket](https://www.atlassian.com/git/tutorials/rewriting-history/git-rebase)