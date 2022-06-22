---
title: chi 框架路由分析
comment: ture
---

（暂时还没写完，不过为了方便阅读，我也发表了出来。这样回家就不用带电脑了。。）

chi 框架相对容易一些，跟上一篇文章中的 appsrv 框架进行一个简单地对比，看一下实现的差异在哪里。

<!--more-->

废话不多说，直接上源码

------

> 下面这段话是我看过后才写的，这个框架中涉及到的了节点的拆分替换。这样做的好处就是可以节省更多的空间，如果彼此之间没有公共部分的话。

我用这三个路由进行的测试：

```
// 1 "/"
// 2 "/hello/world"
// 3 "/hello"
```

如果是按照上述方式进行插入，执行的 2 的时候，node 的存储方式就是一个 `hello/world` ，但是当 3 执行的时候，匹配到了相同的前缀 `hello` 此时会将 `hello/world` 拆分成两个 node，而 3 中的 hello 这个node的实现，就是 2中的 hello。不知道这样表述是否清晰。。

------



```go
// 路由的插入
func (n *node) InsertRoute(method methodTyp, pattern string, handler http.Handler) *node {
    var parent *node
    // 获取访问路径，形如 /hello/world
    search := pattern

    for {
        // Handle key exhaustion
        if len(search) == 0 {
            // Insert or update the node's leaf handler
            n.setEndpoint(method, handler, pattern)
            return n
        }

        // We're going to be searching for a wild node next,
        // in this case, we need to get the tail
        var label = search[0]
        var segTail byte
        var segEndIdx int
        var segTyp nodeTyp
        var segRexpat string
        if label == '{' || label == '*' {
            segTyp, _, segRexpat, segTail, _, segEndIdx = patNextSegment(search)
        }

        var prefix string
        if segTyp == ntRegexp {
            prefix = segRexpat
        }

        // Look for the edge to attach to
        parent = n
        n = n.getEdge(segTyp, label, segTail, prefix)

        // No edge, create one
        if n == nil {
            child := &node{label: label, tail: segTail, prefix: search}
            hn := parent.addChild(child, search)
            hn.setEndpoint(method, handler, pattern)

            return hn
        }

        // Found an edge to match the pattern

        if n.typ > ntStatic {
            // We found a param node, trim the param from the search path and continue.
            // This param/wild pattern segment would already be on the tree from a previous
            // call to addChild when creating a new node.
            search = search[segEndIdx:]
            continue
        }

        // Static nodes fall below here.
        // Determine longest prefix of the search key on match.
        // 匹配最长前缀，一个时间复杂度为 O(n) 的算法，遇到不同的直接 break
        commonPrefix := longestPrefix(search, n.prefix)
        if commonPrefix == len(n.prefix) {
            // 走到这个 if 里边，说明当前 parent 的前缀，被完全包含在了 child 中
            // the common prefix is as long as the current node's prefix we're attempting to insert.
            // keep the search going.
            search = search[commonPrefix:]
            continue
        }

        // Split the node
        child := &node{
            typ:    ntStatic,
            prefix: search[:commonPrefix],
        }
        // replace 的过程就是将，旧 node 替换成最大相同前缀的样子
        parent.replaceChild(search[0], segTail, child)

        // Restore the existing node
        // 上述已经替换了，但是还剩下 旧 node 除了最大相同前缀余下的部分，
        // 这里的操作就是将这个 node 的后半部分重新插入
        n.label = n.prefix[commonPrefix]
        n.prefix = n.prefix[commonPrefix:]
        child.addChild(n, n.prefix)

        // If the new key is a subset, set the method/handler on this node and finish.
        // 拆分后，如果search就没了，说明 search 是 parent node 的 prefix 的子集
        // 直接更新当前 node 的 handler 即可
        search = search[commonPrefix:]
        if len(search) == 0 {
            child.setEndpoint(method, handler, pattern)
            return child
        }

        // Create a new edge for the node
        // 这里其实也好理解了，如果说拆分后，search 中还有内容，
        // 也不用管了，直接扔到下一个child中，之后再添加的时候，
        // 通过匹配最大相同前缀的方式，继续拆。。
        subchild := &node{
            typ:    ntStatic,
            label:  search[0],
            prefix: search,
        }
        hn := child.addChild(subchild, search)
        hn.setEndpoint(method, handler, pattern)
        return hn
    }
}
```

个人感觉这个插入的过程还是挺绕的。。需要结合例子动手看一下，光在脑子里想，抱歉！我智商不够..



chi 框架中，将 node 分成了几个类型，如下：

```go
type nodeTyp uint8

const (
	ntStatic   nodeTyp = iota // /home
	ntRegexp                  // /{id:[0-9]+}
	ntParam                   // /{user}
	ntCatchAll                // /api/v1/*
)
```



看看插入过程涉及到的一些杂七杂八的函数：

```go
// patNextSegment returns the next segment details from a pattern:
// node type, param key, regexp string, param tail byte, param starting index, param ending index
func patNextSegment(pattern string) (nodeTyp, string, string, byte, int, int) {
    ps := strings.Index(pattern, "{")
    ws := strings.Index(pattern, "*")
	
    // 静态node，既不是参数，也不是正则表达式
    if ps < 0 && ws < 0 {
        return ntStatic, "", "", 0, 0, len(pattern) // we return the entire thing
    }

    // Sanity check 完整性检查，也体现了路由的规则，通配符只能出现在 {} 之后
    if ps >= 0 && ws >= 0 && ws < ps {
        panic("chi: wildcard '*' must be the last pattern in a route, otherwise use a '{param}'")
    }

    var tail byte = '/' // Default endpoint tail to / byte

    if ps >= 0 {
        // Param/Regexp pattern is next
        nt := ntParam

        // Read to closing } taking into account opens and closes in curl count (cc)
        // 这里就是找成对的 {} ，虽然没在 leetcode 上做这个题，但是也听过。。
        // 如果 cc > 0，说明花括号多了
        // 如果 cc < 0，说明花括号少了
        // 如果 cc == 0，说明正好匹配上
        cc := 0
        pe := ps
        for i, c := range pattern[ps:] {
            if c == '{' {
                cc++
            } else if c == '}' {
                cc--
                if cc == 0 {
                    pe = ps + i
                    break
                }
            }
        }
        if pe == ps {
            panic("chi: route param closing delimiter '}' is missing")
        }

        // 拿到了括号中的参数名称
        key := pattern[ps+1 : pe]
        pe++ // set end to next position，花括号后边的那个值
		
        // tail 指向的就是花括号后边的第一个字节，比如 /{name}/world
        if pe < len(pattern) {
            tail = pattern[pe]
        }
		
        /*
        	从这里往下开始就是有关正则的内容
        */
        var rexpat string
        if idx := strings.Index(key, ":"); idx >= 0 {
            nt = ntRegexp
            rexpat = key[idx+1:]
            key = key[:idx]
        }

		// 如果是正则匹配，人家还给校验了开始和结束。。
        if len(rexpat) > 0 {
            if rexpat[0] != '^' {
                rexpat = "^" + rexpat
            }
            if rexpat[len(rexpat)-1] != '$' {
                rexpat += "$"
            }
        }
        
        /*
        	注释里边提到了这几个参数是啥意思，不过还是看一眼
        	nt	节点类型
        	key	参数名称
        	rexpat 正则表达式字符串
        	tail	key 参数后边的一个字节
        	ps	参数开始的索引位置
        	pe	参数结束的索引位置
        */
		
        return nt, key, rexpat, tail, ps, pe
    }

    // Wildcard pattern as finale
    if ws < len(pattern)-1 {
        panic("chi: wildcard '*' must be the last value in a route. trim trailing text or use a '{param}' instead")
    }
    return ntCatchAll, "*", "", 0, ws, len(pattern)
}
```



```go
// ntyp		节点类型
// label	label 表示的就是 path 路径上的第一个字节
// tail		表示与 label 相反，表示最后一个字节
// prefix	当前 node 中存储的 path
func (n *node) getEdge(ntyp nodeTyp, label, tail byte, prefix string) *node {
 	// 首先是通过 节点类型找到对应的 节点集合，剩下的就比较简单了..
    nds := n.children[ntyp]
    for i := 0; i < len(nds); i++ {
        if nds[i].label == label && nds[i].tail == tail {
            if ntyp == ntRegexp && nds[i].prefix != prefix {
                continue
            }
            return nds[i]
        }
    }
    return nil
}
```



```go
// 写的 endpoint，其实就是给 node 设置上 handler，正如人家在下面注释所描述的
func (n *node) setEndpoint(method methodTyp, handler http.Handler, pattern string) {
    // Set the handler for the method type on the node
    if n.endpoints == nil {
        n.endpoints = make(endpoints)
    }

    // {name} {job} paramKeys 中存储的就是这些名字
    paramKeys := patParamKeys(pattern)

    // 往下就没啥好说了，通过位运算找到对应的分支，然后直接赋值就行了。。
    if method&mSTUB == mSTUB {
        n.endpoints.Value(mSTUB).handler = handler
    }
    if method&mALL == mALL {
        h := n.endpoints.Value(mALL)
        h.handler = handler
        h.pattern = pattern
        h.paramKeys = paramKeys
        for _, m := range methodMap {
            h := n.endpoints.Value(m)
            h.handler = handler
            h.pattern = pattern
            h.paramKeys = paramKeys
        }
    } else {
        h := n.endpoints.Value(method)
        h.handler = handler
        h.pattern = pattern
        h.paramKeys = paramKeys
    }
}
```



最后这个就是最核心最复杂的了吧...，不过我们为了方便理解，可以暂时先忽略掉**正则匹配**，**带参数**的部分代码。。只关注 nodetyp 是 static 的即可。

```go
// addChild appends the new `child` node to the tree using the `pattern` as the trie key.
// For a URL router like chi's, we split the static, param, regexp and wildcard segments
// into different nodes. In addition, addChild will recursively call itself until every
// pattern segment is added to the url pattern tree as individual nodes, depending on type.
// 人家上边注释说的非常清晰了..
func (n *node) addChild(child *node, prefix string) *node {
    search := prefix

    // handler leaf node added to the tree is the child.
    // this may be overridden later down the flow
    hn := child

    // Parse next segment
    segTyp, _, segRexpat, segTail, segStartIdx, segEndIdx := patNextSegment(search)

    // Add child depending on next up segment
    switch segTyp {
        // 这里直接判断类型呗，如果是static就直接加入到parent的children节点中
        case ntStatic:
        // Search prefix is all static (that is, has no params in path)
        // noop

        default:
        // Search prefix contains a param, regexp or wildcard

        if segTyp == ntRegexp {
            rex, err := regexp.Compile(segRexpat)
            if err != nil {
                panic(fmt.Sprintf("chi: invalid regexp pattern '%s' in route param", segRexpat))
            }
            child.prefix = segRexpat
            child.rex = rex
        }

        if segStartIdx == 0 {
            // Route starts with a param
            child.typ = segTyp

            if segTyp == ntCatchAll {
                segStartIdx = -1
            } else {
                segStartIdx = segEndIdx
            }
            if segStartIdx < 0 {
                segStartIdx = len(search)
            }
            child.tail = segTail // for params, we set the tail

            if segStartIdx != len(search) {
                // add static edge for the remaining part, split the end.
                // its not possible to have adjacent param nodes, so its certainly
                // going to be a static node next.

                search = search[segStartIdx:] // advance search position

                nn := &node{
                    typ:    ntStatic,
                    label:  search[0],
                    prefix: search,
                }
                hn = child.addChild(nn, search)
            }

        } else if segStartIdx > 0 {
            // Route has some param

            // starts with a static segment
            child.typ = ntStatic
            child.prefix = search[:segStartIdx]
            child.rex = nil

            // add the param edge node
            search = search[segStartIdx:]

            nn := &node{
                typ:   segTyp,
                label: search[0],
                tail:  segTail,
            }
            hn = child.addChild(nn, search)

        }
    }

    n.children[child.typ] = append(n.children[child.typ], child)
    // 排序操作，暂时不知道人家这个用途。。
    n.children[child.typ].Sort()
    return hn
}
```

