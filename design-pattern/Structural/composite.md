---
title: 组合模式
date: 2022-07-11
comment: true
---

设计模式的学习逐渐变成了一周一次，一次一周。不求一天之内学多少，重要的是真正了解到模式适用的场景，一味的图快，可能有点狗熊掰棒子那味了。那这周的主要内容是**组合模式**。

<!--more-->

## 组合模式

主要思想：**整体与部分可以被一致对待**

组合模式（Composite），将对象组合成树形结构以表示‘部分-整体’的层次结构。组合模式使得用户对单个对象和组合对象的使用具有一致性。



### 透明方式 与 安全方式

主要区别在于叶子节点是否实现了所有的 Component 接口。



### 何时使用？

需求中是体现**部分与整体**层次的结构时，以及希望用户可以忽略组合对象与单个对象的不同，统一地使用组合结构中的所有对象时，就应该考虑用组合模式。形如总部与分部，文件与文件夹这种形式。



### 代码实现

```go
package composite

import (
    "fmt"
    "strings"
)

type IComponent interface {
    AddDepartment(department IComponent)
    DisplayDepartment(indent int)
}

type HeadQuarters struct {
    Name        string
    Departments []IComponent // 总公司下的分公司
}

func (h *HeadQuarters) AddDepartment(department IComponent) {
    if h.Departments == nil {
        h.Departments = make([]IComponent, 0)
    }
    h.Departments = append(h.Departments, department)
}

func (h HeadQuarters) DisplayDepartment(indent int) {
    fmt.Printf("%s%s\n", strings.Repeat("-", indent), h.Name)

    for _, department := range h.Departments {
        department.DisplayDepartment(indent + 2)
    }
}

type HumanResource struct {
    Name      string
    Resources []IComponent // 不同地区的人力
}

func (h *HumanResource) AddDepartment(department IComponent) {
    if h.Resources == nil {
        h.Resources = make([]IComponent, 0)
    }
    h.Resources = append(h.Resources, department)
}

func (h HumanResource) DisplayDepartment(indent int) {
    fmt.Printf("%s%s\n", strings.Repeat("-", indent), h.Name)
    for _, department := range h.Resources {
        department.DisplayDepartment(indent + 2)
    }
}

type Administrative struct {
    Name      string
    Resources []IComponent // 不同地区的行政
}

func (a *Administrative) AddDepartment(department IComponent) {
    if a.Resources == nil {
        a.Resources = make([]IComponent, 0)
    }
    a.Resources = append(a.Resources, department)
}

func (a Administrative) DisplayDepartment(indent int) {
    fmt.Printf("%s%s\n", strings.Repeat("-", indent), a.Name)
    for _, department := range a.Resources {
        department.DisplayDepartment(indent + 2)
    }
}
```

