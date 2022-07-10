---
title: 简易版 pstree
comment: true
---

最近在学习 C 语言的相关知识，想通过 C 语言实现一个简易版的 `pstree` 命令行工具，我认为对 C 新手来说还是有一定难度的，虽然最后只通过 C 实现了一半，但是这个过程是最让我印象深刻的。现代程序员使用着便捷的开发工具，输入几个首字母就能打印出自己想调用的库函数。

<!--more-->

在 C 中，这个还是挺麻烦的，或许跟我使用的环境有关，我在虚机上使用 VIM 写的，没有快捷键，更没有代码自动补全。 想知道一个库函数怎么用、怎么传参都需要通过手册查询，在某些程度上，这个过程也可以强化自己的工具集。



一般，我是通过 `tmux` 开两个 session，一个用来敲代码，另一个查手册。最后卡在 C 上的原因是：

N 叉树实现问题，不知道怎么初始化结构体，又不想把每个节点中的子节点数量写死，所以就通过 Go 实现了后半部分。

```c
//子节点硬编码
struct Node {
    int Id;
    struct Node *Child[10];
}

// 这种在通过 malloc 初始化的时候，存在问题
struct Node {
    int Id;
    struct Node *Child;
}
```



还没有看学到 C struct 中的内容（大学学的早就还给老师了），待我学成归来，一定把这块好好补上。



另一个在 C 中比较难的是，不知道怎么处理字符串，不知道哪些库函数可以做我想做的事。就比如拆分字符串，如果说每次通过 split 拆分一个太复杂了，它有个 strtok 更方便..不过这些问题还好，是自己对语言的不熟悉。



实现 `pstree` 过程中花了较多时间的点：

- 如何找到进程的相关信息，父进程ID，当前进程的名称
- 如何过滤非进程目录
- 如何存储父进程及其子进程
- 如何以缩进的方式输出



```go
package main

import (
    "fmt"
    "sort"
    "log"
    "os"
    "path/filepath"
    "strconv"
    "strings"
)

// init root node
var Root = &Node{Id: 1, Child: make(map[uint]*Node)}


type Node struct {
    Id    uint
    Name  string
    Child map[uint]*Node
}

func (n Node) GetName() string {
    return n.Name
}

func (n *Node) InsertChild(id uint, childName string) {
    if n.Child == nil {
        n.Child = make(map[uint]*Node)
    }

    n.Child[id] = &Node{
        Id:    id,
        Name:  childName,
        Child: nil,
    }
}

// pre-traverse
func TraverseTree(root *Node, indent int) {
    fmt.Printf("%s%d-%s\n", strings.Repeat(" ", indent), root.Id, root.GetName())

    if root.Child == nil {
        return
    }

    for _, child := range root.Child {
        TraverseTree(child, indent+1)
    }
}

func FindParent(parentId uint, root *Node) *Node {
    if parentId == 0 || parentId == 1 {
        return Root
    }

    // the last node
    if root.Child == nil {
        return nil
    }

    if parent, ok := root.Child[parentId]; ok && parent.Id == parentId {
        return parent
    }

    for _, node := range root.Child {
        if parent := FindParent(parentId, node); parent != nil {
            return parent
        }
    }
    return nil
}

func GetProcs() {
    fps, err := filepath.Glob("/proc/[0-9]*")
    if err != nil {
        log.Fatalln(err)
    }
	
    sort.Slice(fps, func(i, j int) bool {
        one := strings.TrimPrefix(fps[i], "/proc/")
        two := strings.TrimPrefix(fps[j], "/proc/")

        num1, _ := strconv.Atoi(one)
        num2, _ := strconv.Atoi(two)

        if num1 < num2 {
            return true
        }
        return false
    })

    for _, procFile := range fps {
        stat, err := os.ReadFile(filepath.Join(procFile, "/stat"))
        if err != nil {
            log.Println(err)
            continue
        }

        pid, parentId, processName := GetInfoFromStat(string(stat))
        if pid == 1 {
            Root.Name = processName
            continue
        }

        // after find parent, insert to proc tree
        // because the proc dir order by pid
        // so parent always insert to tree before child
        parentNode := FindParent(parentId, Root)
        if parentNode == nil {
            log.Fatalln("parent doesn't in tree")
        }
        parentNode.InsertChild(pid, processName)
    }
}

func GetInfoFromStat(stat string) (uint, uint, string) {
    var (
        procName string
        pid      int
        ppid     int
    )

    tokens := strings.FieldsFunc(stat, func(r rune) bool {
        return r == '(' || r == ')'
    })

    if len(tokens) != 3 {
        return 0, 0, ""
    }

    pid, _ = strconv.Atoi(strings.TrimSuffix(tokens[0], " "))
    procName = tokens[1]
    tmp := strings.Split(strings.TrimPrefix(tokens[2], " "), " ")
    ppid, _ = strconv.Atoi(tmp[1])
    return uint(pid), uint(ppid), procName
}

func main() {
 	// populate N Tree
    GetProcs()
    // printf tree
    TraverseTree(Root, 0)
}
```

代码如上，用 Go 实现也遇到了与上述一样的问题，过滤目录，处理字符串。

新学到了 `filepath.Glob()` 还是比较好用的，直接通过正则的方式就将所有进程相关的目录输出了。

以及 `strings.FieldFunc()` 每次可以使用不通过分隔符进行过滤跟 C 中 `strtok` 类似。



最难的就是如何输出这棵树，先把当前实现的输出放出来（是不是挺像那么回事）：

```shell
1-systemd
 19-systemd-journal
 29-systemd-udevd
  9405-systemd-udevd
  9402-systemd-udevd
  9403-systemd-udevd
  9404-systemd-udevd
 76-sshd
  86-sshd
   88-bash
    9406-pstree
 81-systemd-logind
 82-dbus-daemon
 84-agetty
 # pid-process_name
```

通过缩进的方式来显示父子进程之间的关系，通过前序遍历的方式就能输出这个方式但是输出内容都是齐刷刷的，最难的地方在于如何考虑缩进的长度。



最后实在是想不出来了，上 Github 看了一下，我的方向没错，需要在前序遍历过程中加上一个缩进长度，每往下遍历一层缩进长度就加一，最后就可以输出上述的样子。



另外，我们还可以通过命令行的方式，指定一些输出格式，具体可以参考手册中的 `pstree` 就比如：

- 按照特定顺序输出

上述实现方式，每次输出的顺序都是不同的，因为 map 的遍历是无序的。



总的来说，对自己理解进程，任务管理器是有很大帮助的，还是要**敢于动手**。