---
title: 一个 web 框架的那些事
comment: true
---

项目上用了一套开源的云管理代码，他们框架中的路由的相关功能都是自研的，学习学习人家是怎么实现的。

<!--more-->

大致分成以下几个部分：

- 使用什么数据结构存储
- 路由的查找过程
- 关于正则的路由是怎么实现的



**实现路由的数据结构**

这里巧妙的运用了 Go 原生 map 的特性，结合 map 高效的查询，构建了一颗**基数树**。

```go
type RadixNode struct {
	data        interface{}
	stringNodes map[string]*RadixNode
	regexpNodes []*RegexpNode
	segNames    map[int]string
}

type RegexpNode struct {
	node   *RadixNode
	regStr string
}
```

光看声明方式或许有些抽象，把图画出来就好多了。（回头补上）



**路由注册过程**

核心的代码就这么多，对于我这种菜鸟来说，阅读起来还行，不是那么费劲，但是要想写出来这种代码，得有几年功底哈哈。

```go
// segments 就是将 path 通过 "/" 进行分割，得到的结果
func (r *RadixNode) add(segments []string, data interface{}, depth int, segNames map[int]string) error {
    // 递归的终止条件
    if len(segments) == 0 {
        // 这里的 data 就是 handler 的信息，最后要通过 path 最终找到handler
        if r.data != nil {
            // 添加了重复路由，会报这个错误
            return fmt.Errorf("Duplicate data for node")
        } else {
            r.data = data
            r.segNames = segNames
            return nil
        }
    } else {
        var nextNode *RadixNode
        // 判断是否为正则匹配的 path （为了方便理解可以先看下边 else 的部分）
        if isRegexSegment(segments[0]) {
            var (
                // 正则字符串
                regStr     string
                segName    string
                // 把 "< >" 去掉
                segStr     = segments[0][1 : len(segments[0])-1]
                // 后续判断是否为 paramName:reg 的这种形式
                splitIndex = strings.IndexByte(segStr, ':')
            )

            if splitIndex < 0 {
                regStr = ".*" // match anything
                segName = "<" + segStr + ">"
            } else {
                // <phone_number:^1[0-9-]{10}$>
                // 结合上边这个表达式，一下就清晰明了了
                regStr = segStr[splitIndex+1:]
                segName = "<" + segStr[0:splitIndex] + ">"
            }
			
            // TODO 暂时没看出来这个 segNames 和 depth 的关系
            if segNames == nil {
                segNames = make(map[int]string, 0)
            }
            segNames[depth-1] = segName
			// 这里的逻辑就和下边的一样了，不再赘述
            if node, ok := isRegstrInRegexpNodes(r.regexpNodes, regStr); ok {
                nextNode = node
            } else {
                nextNode = NewRadix()
                regNode := NewRegexpNode(nextNode, regStr)
                // 这里采用的是 array 存储，所以在查找的时候有一种可能
                // 就是一个node下挂着多个正则匹配，并且秉着先进来的优先级高的方式
                r.regexpNodes = append(r.regexpNodes, regNode)
            }
        } else {
            // 这里的 stringNodes 存储结构见下图，其实很简单，一个 node 代表的就是path中的一部分
            // 如果说两个 path 有重合的部分，那么他们之间就可以共用这些 node，这也是基数树的一个特征
            // ** 正如这里的判断，其实就是验证是否可以有公共的部分 **
            if node, ok := r.stringNodes[segments[0]]; ok {
                nextNode = node
            } else {
                // 走到这里说明，这个 seg 不存在，也就是没有公共部分，需要新建
                nextNode = NewRadix()
                r.stringNodes[segments[0]] = nextNode
            }
        }
        // 递归调用
        return nextNode.add(segments[1:], data, depth+1, segNames)
    }
}

```

![image-20220218174102417](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220218174102417.png)



这里我想了下，不用递归行不行，答案肯定是可以的，所有的递归都是可以通过 for 循环来完成的。但是细想了下，毕竟只有在项目启动中执行一次，也没必要，这样看起来更清晰，代码更简洁。



**路由查找过程**

先简单看看正则匹配的过程。比如我们定义了一个路由 `/user/<^1[0-9-]{10}$>` (按照人家框架的约束条件声明)

那么只要是匹配到该规则的路径，都是相同的 handler 进行处理。



如果上述路由注册的过程看明白了的话，我想你心里已经可以构建出一颗基数树，没有也没关系，我给你画出来，请见下图。

![image-20220220154418325](https://gitee.com/yangbaoqiang/images/raw/master/blogpics/image-20220220154418325.png)



画的比较简陋，不过大概意思就是这样。



**代码实现**

我们可以尝试使用下面的函数，加上上面的图作为辅助，查找一下 `/student/Peter`

```go
// 可以先看下返回结果
// bool 表示是否找到
// RadixNode 则表示具体的 node，也可以理解为最终的 handler
func (r *RadixNode) match(segments []string) (*RadixNode, bool) {
    if len(segments) == 0 {
        return r, true
    } else if len(r.stringNodes) == 0 && len(r.regexpNodes) == 0 {
        return r, false
    }
	
    // 不难看出，这里仍然采用的是递归调用的方式
    if node, ok := r.stringNodes[segments[0]]; ok {
        if rnode, _ := node.match(segments[1:]); rnode != nil && rnode.data != nil {
            return rnode, true
        }
    }

    var nodeTmp *RadixNode
    for _, regNode := range r.regexpNodes {
        if regexp.MustCompile(regNode.regStr).MatchString(segments[0]) {
            if rnode, fullMatch := regNode.node.match(segments[1:]); rnode != nil && rnode.data != nil {
                if fullMatch {
                    return rnode, fullMatch
                } else {
                    if nodeTmp != nil {
                        log.Errorf("segments %v match mutil node", segments)
                        continue
                    }
                    nodeTmp = rnode
                }
            }
        }
    }

    if nodeTmp != nil {
        return nodeTmp, false
    } else {
        return r, false
    }
}
```



> 从上述代码可以看出，路由匹配过程中，优先级最高的是名称相同，其次才是正则匹配。



