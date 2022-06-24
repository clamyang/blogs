---
title: 初识 go ast
comments: true
---

![](https://s2.loli.net/2022/06/09/xjzPQnLJZUb7BIw.png)

从介绍 AST 到动手实践解析一个文件，再到查看开源代码的使用场景，一步步了解 AST。



前一阵子学到了一个 linter（检查err是否被 wrap），实现的方式用的就是 AST ，正好借这个机会了解一下。

<!--more-->

AST (Abstact Syntax Tree) 抽象语法树，第一次听到这个应该还是在读大学时候，没想到终究还是要填坑。

> AST 是源代码的一种抽象表示。



![go-ast-types](https://s2.loli.net/2022/06/09/1SkCYVgRaWsZDwd.jpg)

这张大图把 AST 中的对象进行了归类，包含了语句（If, For, Range）、表达式、声明、注释。



然后我们看一个官方关于 Inspect 的例子：

```go
package main

import (
	"fmt"
	"go/ast"
	"go/parser"
	"go/token"
	"log"
)

func main() {
	fset := token.NewFileSet()
	src := `
package p
const c = 1.0
var X = f(3.14)*2 + c
`
	astFile, err := parser.ParseFile(fset, "src.go", src, 0)
	if err != nil {
		log.Fatalln(err)
	}

	ast.Inspect(astFile, func(n ast.Node) bool {
		var s string
		switch x := n.(type) {
		//case *ast.BasicLit:
		//	s = x.Value
		case *ast.Ident:
			s = x.Name
		}
		if s != "" {
			fmt.Printf("%s:\t%s\n", fset.Position(n.Pos()), s)
		}
		return true
	})
}

/*
	Output:
        src.go:2:9:	p
        src.go:3:7:	c
        src.go:4:5:	X
        src.go:4:9:	f
        src.go:4:21:	c
*/
	
```

把 `BasicLit` 注释掉了，方便一步步观察输出内容，通过输出内容可以得出，`ast.Ident` 的含义就是标识符。



然后像是这种的 `xxxLit`  都是字面量的声明方式如：

```go
// BasicLit
var a = "string"
const name = "bqyang"

// FuncLit
var fn = func() {}
```

其中 `BasicLit` 中包含：token.INT, token.FLOAT, token.IMAG, token.CHAR, or token.STRING 内容。



这么看不太直观，可以使用 [GoAst Viewer](https://yuroyoro.github.io/goast-viewer/index.html) 输出一颗完整的语法树，还是使用上述内容：

```
     0  *ast.File {
     1  .  Doc: nil
     2  .  Package: foo:1:1
     3  .  Name: *ast.Ident {
     4  .  .  NamePos: foo:1:9
     5  .  .  Name: "p"
     6  .  .  Obj: nil
     7  .  }
     8  .  Decls: []ast.Decl (len = 2) {
     9  .  .  0: *ast.GenDecl {
    10  .  .  .  Doc: nil
    11  .  .  .  TokPos: foo:2:1
    12  .  .  .  Tok: const
    13  .  .  .  Lparen: -
    14  .  .  .  Specs: []ast.Spec (len = 1) {
    15  .  .  .  .  0: *ast.ValueSpec {
    16  .  .  .  .  .  Doc: nil
    17  .  .  .  .  .  Names: []*ast.Ident (len = 1) {
    18  .  .  .  .  .  .  0: *ast.Ident {
    19  .  .  .  .  .  .  .  NamePos: foo:2:7
    20  .  .  .  .  .  .  .  Name: "c"
    21  .  .  .  .  .  .  .  Obj: *ast.Object {
    22  .  .  .  .  .  .  .  .  Kind: const
    23  .  .  .  .  .  .  .  .  Name: "c"
    24  .  .  .  .  .  .  .  .  Decl: *(obj @ 15)
    25  .  .  .  .  .  .  .  .  Data: 0
    26  .  .  .  .  .  .  .  .  Type: nil
    27  .  .  .  .  .  .  .  }
    28  .  .  .  .  .  .  }
    29  .  .  .  .  .  }
    30  .  .  .  .  .  Type: nil
    31  .  .  .  .  .  Values: []ast.Expr (len = 1) {
    32  .  .  .  .  .  .  0: *ast.BasicLit {
    33  .  .  .  .  .  .  .  ValuePos: foo:2:11
    34  .  .  .  .  .  .  .  Kind: FLOAT
    35  .  .  .  .  .  .  .  Value: "1.0"
    36  .  .  .  .  .  .  }
    37  .  .  .  .  .  }
    38  .  .  .  .  .  Comment: nil
    39  .  .  .  .  }
    40  .  .  .  }
    41  .  .  .  Rparen: -
    42  .  .  }
    43  .  .  1: *ast.GenDecl {
    44  .  .  .  Doc: nil
    45  .  .  .  TokPos: foo:3:1
    46  .  .  .  Tok: var
    47  .  .  .  Lparen: -
    48  .  .  .  Specs: []ast.Spec (len = 1) {
    49  .  .  .  .  0: *ast.ValueSpec {
    50  .  .  .  .  .  Doc: nil
    51  .  .  .  .  .  Names: []*ast.Ident (len = 1) {
    52  .  .  .  .  .  .  0: *ast.Ident {
    53  .  .  .  .  .  .  .  NamePos: foo:3:5
    54  .  .  .  .  .  .  .  Name: "X"
    55  .  .  .  .  .  .  .  Obj: *ast.Object {
    56  .  .  .  .  .  .  .  .  Kind: var
    57  .  .  .  .  .  .  .  .  Name: "X"
    58  .  .  .  .  .  .  .  .  Decl: *(obj @ 49)
    59  .  .  .  .  .  .  .  .  Data: 0
    60  .  .  .  .  .  .  .  .  Type: nil
    61  .  .  .  .  .  .  .  }
    62  .  .  .  .  .  .  }
    63  .  .  .  .  .  }
    64  .  .  .  .  .  Type: nil
    65  .  .  .  .  .  Values: []ast.Expr (len = 1) {
    66  .  .  .  .  .  .  0: *ast.BinaryExpr {
    67  .  .  .  .  .  .  .  X: *ast.BinaryExpr {
    68  .  .  .  .  .  .  .  .  X: *ast.CallExpr {
    69  .  .  .  .  .  .  .  .  .  Fun: *ast.Ident {
    70  .  .  .  .  .  .  .  .  .  .  NamePos: foo:3:9
    71  .  .  .  .  .  .  .  .  .  .  Name: "f"
    72  .  .  .  .  .  .  .  .  .  .  Obj: nil
    73  .  .  .  .  .  .  .  .  .  }
    74  .  .  .  .  .  .  .  .  .  Lparen: foo:3:10
    75  .  .  .  .  .  .  .  .  .  Args: []ast.Expr (len = 1) {
    76  .  .  .  .  .  .  .  .  .  .  0: *ast.BasicLit {
    77  .  .  .  .  .  .  .  .  .  .  .  ValuePos: foo:3:11
    78  .  .  .  .  .  .  .  .  .  .  .  Kind: FLOAT
    79  .  .  .  .  .  .  .  .  .  .  .  Value: "3.14"
    80  .  .  .  .  .  .  .  .  .  .  }
    81  .  .  .  .  .  .  .  .  .  }
    82  .  .  .  .  .  .  .  .  .  Ellipsis: -
    83  .  .  .  .  .  .  .  .  .  Rparen: foo:3:15
    84  .  .  .  .  .  .  .  .  }
    85  .  .  .  .  .  .  .  .  OpPos: foo:3:16
    86  .  .  .  .  .  .  .  .  Op: *
    87  .  .  .  .  .  .  .  .  Y: *ast.BasicLit {
    88  .  .  .  .  .  .  .  .  .  ValuePos: foo:3:17
    89  .  .  .  .  .  .  .  .  .  Kind: INT
    90  .  .  .  .  .  .  .  .  .  Value: "2"
    91  .  .  .  .  .  .  .  .  }
    92  .  .  .  .  .  .  .  }
    93  .  .  .  .  .  .  .  OpPos: foo:3:19
    94  .  .  .  .  .  .  .  Op: +
    95  .  .  .  .  .  .  .  Y: *ast.Ident {
    96  .  .  .  .  .  .  .  .  NamePos: foo:3:21
    97  .  .  .  .  .  .  .  .  Name: "c"
    98  .  .  .  .  .  .  .  .  Obj: *(obj @ 21)
    99  .  .  .  .  .  .  .  }
   100  .  .  .  .  .  .  }
   101  .  .  .  .  .  }
   102  .  .  .  .  .  Comment: nil
   103  .  .  .  .  }
   104  .  .  .  }
   105  .  .  .  Rparen: -
   106  .  .  }
   107  .  }
   108  .  Scope: *ast.Scope {
   109  .  .  Outer: nil
   110  .  .  Objects: map[string]*ast.Object (len = 2) {
   111  .  .  .  "c": *(obj @ 21)
   112  .  .  .  "X": *(obj @ 55)
   113  .  .  }
   114  .  }
   115  .  Imports: nil
   116  .  Unresolved: []*ast.Ident (len = 1) {
   117  .  .  0: *(obj @ 69)
   118  .  }
   119  .  Comments: nil
   120  }
```

```go
// Inspect traverses an AST in depth-first order: It starts by calling
// f(node); node must not be nil. If f returns true, Inspect invokes f
// recursively for each of the non-nil children of node, followed by a
// call of f(nil).
```

