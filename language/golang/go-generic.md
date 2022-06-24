---
title: go generic
comments: true

---

![](https://s2.loli.net/2022/06/06/LNYfsvCZwxeh3l9.png)



go 泛型学习，距离泛型的发布已经有很长一段时间了，之前大概看了下官方文档，说有些东西可能将来都会改变，不保证向前兼容，就没有具体了解，1.18也发了几个版本了，感觉再不努力又要被小伙伴们卷完了。



<!--more-->

## 困扰

没有泛型的时候带给我们的困扰（虽然没有亲身体会过，但是感觉很麻烦）

### Case 1

```go
// Sum returns the sum of the provided arguments.
func Sum(args ...int) int {
    var sum int
    for i := 0; i < len(args); i++ {
        sum += args[i]
    }
    return sum
}
```



要计算 n 数之和，如上述例子，如果这时要计算 int32, float64 类型元素的和呢？



通常情况下，我们需要进行 asserting type，或者为每种类型写不同的函数 `sumInt` `sumInt32` `sumInt64` 等，在源码中就有类似的代码。



### Case 2

```go
func main() {
    // Declare "host" and "port" in order to create pointers to satisfy the
    // fields in the "request" struct.
    host, port := "local", 80

    print(request{
        host: &host,
        port: &port,
    })
}
```

> This leads to cluttered, hard-to-read code where the only purpose variables serve is for deriving pointers.

对每个变量进行取址操作，*我之前就写过这样的代码* ，为了取址不得不声明一个变量。



针对这种情况，可以实现不同的函数进行取址操作。

```go
// PtrInt returns *i.
func PtrInt(i int) *int {
    return &i
}

// PtrStr returns *s.
func PtrStr(s string) *string {
    return &s
}

func main() {
    // Use the two helper functions that return pointers to their provided
    // values. Remember, this pattern must scale with the number of distinct,
    // defined types that need to be passed by pointer instead of value.
    print(request{
        host: PtrStr("local"),
        port: PtrInt(80),
    })
}
```



看起来可能比上边的优雅了一些，但是还不够，我们看下泛型。

```go
// Ptr returns *value.
func Ptr[T any](value T) *T {
    return &value
}

func main() {
    // No local variables and the typed helper functions can be collapsed into
    // a single, generic function for getting a pointer to a value.
    print(request{
        host: Ptr("local"),
        port: Ptr(80),
    })
}
```



一方面是代码量减少了很多，另一方看是看起来更整洁。



## generic 语法

使用 *泛型* 重写这段代码

```go
// Sum returns the sum of the provided arguments.
func Sum(args ...int) int {
    var sum int
    for i := 0; i < len(args); i++ {
        sum += args[i]
    }
    return sum
}
```



泛型版本：

```go
func Sum[T int](args ...T) T {
    var sum T
    for i := 0; i < len(args); i++ {
        sum += args[i]
    }
    return sum
}
```



`[]` 是用来定义泛型的，常用的模式为 `[<ID> <CONSTRAINT>]`

- `<ID>` 是用来表示泛型的符号；
- `<CONSTRAINT>` 约束，表明可使用的具体类型，如上，只能使用 int；



但是这时候我们想支持 int64 类型的 n 数和应该怎么办？



### Constraint

```go
func Sum[T int|int64](args ...T) T {
    var sum T
    for i := 0; i < len(args); i++ {
        sum += args[i]
    }
    return sum
}
```



使用 `|` 运算符，这时，T 就可以满足 int 或者 int64，至于为什么使用 `|` 运算符不做过多考究。



但是这时还有一个小问题，如果说要支持更多的类型应该怎么做？难道要 `int|int32|int64` ? go 也给我们提供了相应的语法。



#### `any` constraint

```go
// sum_any.go
func Sum[T any](args ...T) T {
    var sum T
    for i := 0; i < len(args); i++ {
        sum += args[i]
    }
    return sum
}

func main() {
    fmt.Println(Sum([]int{1, 2, 3}...))
    fmt.Println(Sum([]int8{1, 2, 3}...))
    fmt.Println(Sum([]uint32{1, 2, 3}...))
    fmt.Println(Sum([]float64{1.1, 2.2, 3.3}...))      
    fmt.Println(Sum([]complex128{1.1i, 2.2i, 3.3i}...))
}
```



去执行试下，会意外的发现使用 `any` 其实行不通，原因是啥呢？

> ./sum_any.go:11:3: invalid operation: operator + not defined on sum (variable of type T constrained by any)



 `+` 运算符并不是对所有的类型都生效。



既然这样，我们是不是可以声明一个东西，限定只能输数字类型的参数呢？



#### Composite constraints

复合类型

```go
// Numeric expresses a type constraint satisfied by any numeric type.
type Numeric interface {
    uint | uint8 | uint16 | uint32 | uint64 |
    int | int8 | int16 | int32 | int64 |
    float32 | float64 |
    complex64 | complex128
}

// Sum returns the sum of the provided arguments.
func Sum[T Numeric](args ...T) T {
    var sum T
    for i := 0; i < len(args); i++ {
        sum += args[i]
    }
    return sum
}
```



下面这种情况如何处理？

```go
// id is a new type definition for an int64
type id int64

func main() {
    fmt.Println(Sum([]id{1, 2, 3}...))
}
```



我们给 int64 起了个别名，这时候编译就会报错了，那我们应该怎样去识别别名呢？



#### Tilde `~`

波浪号就是干这个事情的，举个例子，`~int` 表示：

- 内置的 int 类型
- `type Integer int ` 类型，起别名的类型



```go
type Numeric interface {
    uint | uint8 | uint16 | uint32 | uint64 |
    // 调整如下
    int | int8 | int16 | int32 | ~int64 |
    float32 | float64 |
    complex64 | complex128
}
```



### Type inference

- [ ] [Type Parameters Proposal](https://go.googlesource.com/proposal/+/refs/heads/master/design/43651-type-parameters.md) 待阅读



- Type inference is a convenience feature
- The Go compiler tries *really* hard to infer the intended types, but it does not always work when you think it should
- If you are not sure why something written generically is not working, try providing the types explicitly



### Explicit types

显示指定类型

```go
func main() {
	fmt.Println(Sum(1, 2, 3))
	fmt.Println(Sum([]id{1, 2, 3}...))
	
	// Generic types can be specified explicitly by invoking a function
	// with the bracket notation and the list of types to use. Because
	// the Sum function only has a single, generic type -- "T" -- the
	// call "Sum[float64]" means that "T" will be replaced by "float64"
	// when compiling the code. Since the values "1" and "2" can both
	// be treated as "float64," the code is valid.
    
    // go 类型推断为 int 类型，但是后续出现了浮点数，导致失败。
    // 我们可以明确规定这个为 浮点数类型。
	fmt.Println(Sum[float64](1, 2, 3.0))
}
```



### Multiple generic types

到这里为止，接触到的都是接收单一参数的函数，这里学习下怎么接收多个泛型参数。

```go
// PrintIDAndSum prints the provided ID and sum of the given values to stdout.
// 应该还记得 ~ 的用法吧，适配类型别名
func PrintIDAndSum[T ~string, K Numeric](id T, sum SumFn[K], values ...K) {

    // The format string uses "%v" to emit the sum since using "%d" would
    // be invalid if the value type was a float or complex variant.
    fmt.Printf("%s has a sum of %v\n", id, sum(values...))
}
```



> 注意

在调用这个函数时仍需要显示指定 `sum` 的类型，`PrintIDAndSum("xx", Sum[int32], 1, 2, 3)` ，如果没有指定类型，就会报错：`./mulgen.go:12:22: cannot use generic function Sum without instantiation`



### Declaring a new instance of `T` with `var`

这里跟在上述 sum 中声明变量没什么区别。

### Declaring a new instance of `T` with `new`

new 之后的变量会被分配地址，不再是 nil。

### Structs

将泛型和结构体结合使用，不难看出语法上和函数定义大差不差。

```go
type Ledger[T ~string, K Numeric] struct {
    ID 		T
    Amounts []k
    SumFn	SumFn[K]
}
```



为泛型结构体添加方法，正因为结构体中有泛型，所以在声明方法时也要导入**相应的符号，约束不需要**，因为是为了类型推断使用。

```go
func (l Ledger[T, K]) PrintIDAndSum() {
    fmt.Printf("%s has a sum of %v\n", l.ID, l.SumFn(l.Amounts...))
}
```



简单示例：

```go
// generic struct
package main

type Numeric interface {
    int | int32 | int64
}

type Ledger[T ~string, K Numeric] struct {
    ID T
    Amounts []K
}

func (l Ledger[T, K]) PrintA() {
    print(l.ID)
    print(l.Amounts)
}

func main() {
    Ledger[string, int]{
        ID: "test-qq",
        Amounts: []int{1, 2,3},
    }.PrintA()

    Ledger[string, int32]{
        ID: "test-qq",
        Amounts: []int32{1, 2,3},
    }.PrintA()
}
```



#### Structural Constraints

这个特性还存在一些问题，泛型的结构体暂时禁止访问字段，其实感觉挺一般的，既然更版本了为啥不做好、做完善呢？



...



**匿名结构体**

如果说要接收任意包含这三个字段的结构体（即 Ledger 的实例），需要像下面这样实现，**所有的结构体都实现了匿名结构体**。

```go
func SomeFunc[
    T ~string,
    K Numeric,
    L ~struct {
        ID      T
        Amounts []K
        SumFn   SumFn[K]
    },
](l L) {
}

func main() {
    SomeFunc[string, int, Ledger[string, int]](Ledger[string, int]{
        ID:      "acct-1",
        Amounts: []int{1, 2, 3},
        SumFn:   Sum[int],
    })
}
```



> Structural constraints must match the struct *exactly*, and this means even if all of the fields in the constraint are present, the presence of additional fields in the provided value means the type does not satisfy the constraint.



这里可以暂时不考虑，后续待官方完善了再学也不迟，毕竟现在工作中用不到这些东西..



### Interface constraints

接口约束之前已经见过了， `Numeric`  限制了类型只能是 `int int32 int64`

```go
type Numeric interface {
    int | int32 | int64
}
```

 

对于结构体来说，想通过接口调用方法，需要在上述那样的基础上再添加上方法。

```go
// Ledgerish expresses a constraint that may be satisfied by types that have
// ledger-like qualities.
type Ledgerish[T ~string, K Numeric] interface {
    ~struct {
        ID      T
        Amounts []K
        SumFn   SumFn[K]
    }

    PrintIDAndSum()
}
```



### Careful Constructs



### Internals

#### Type erasure

在了解 Go 中泛型是怎么做到运行时安全前，先了解一下类型擦除。

Wiki 上的定义是：*the load-time process by which explicit type annotations are removed from a program before it is executed at run-time*

翻译过来就是，程序在运行时执行前，进程显示的将类型注解从程序中移除掉。（自己翻译的）



伪代码演示如下：

```go
var ints = List<Int32>{1, 2, 3}
var strs = List<String>{"Hello", "world"}
```

**Type erasure** 执行后

```go
var ints = List{1, 2, 3}
var strs = List{"Hello", "world"}
```

这两个的唯一区别就是将类型擦除掉了，这个特性在很多流行的编程语言中都有。



##### JAVA

java 中 type erasure 的体现：[java-type-erasure](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/01-type-erasure/02-java.md)

##### .Net

.Net 中 type erasure 的体现：[.Net-type-erasure](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/01-type-erasure/03-dotnet.md)

##### Golang

Go 中 type erasure 的体现：[Golang-type-erasure](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/01-type-erasure/04-golang.md) ，这里介绍到，Go中没有反省模板，list 中保留的仍是具体类型，不像 JAVA 中翻译成 object。



#### Runtime-safety

关于泛型这里我一直有个疑问，比如我们定义了一个泛型数组，使用 Go 代码声明如下：

```go
type List[T any] []T
```

假设我们在初始化的时候使用 int，但是后续执行 append 操作的时候加一个 string 能不能行得通？**显然在 Go  中并不行**。那么在其他语言中呢？



##### JAVA

[java-runtime-safety](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/02-runtime-type-safety/01-java.md) 从这里不难看出，java 中擦出了初始化泛型的信息，很难保持在 runtime 时安全。



##### .NET

[.NET-runtime-safety](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/02-runtime-type-safety/02-dotnet.md) 这里保持了泛型的相关信息，所以在运行时是安全的。



##### Golang

[Golang-runtime-safety](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/02-runtime-type-safety/03-golang.md) 通过前面的内容我们知道，Go 中没有进行 type erasure，所以也可以做到 runtime 时安全。



#### Runtime-instantiation

> The ability to instantiate new types using generics at runtime，在运行时使用泛型初始化的能力。



我觉得，如果要支持运行时初始化的能力，需要具备动态推断类型的能力，在运行时检测泛型是哪种类型。



##### JAVA

[java-runtime-instantiation](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/03-runtime-instantiation/01-java.md) 不支持，Java中的泛型纯粹是编译时特性。

##### .NET

[.NET-runtime-instantiation](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/03-runtime-instantiation/02-dotnet.md) 支持.. 这么一看.. .NET 还是牛逼..我有个大学同学人家都是 JAVA，Python，这兄弟学了两年 .NET 哈哈哈哈，后来由于比较冷门，也放弃了哈哈。

##### Golang

[golang-runtime-instantiation](https://github.com/akutz/go-generics-the-hard-way/blob/main/05-internals/03-runtime-instantiation/03-golang.md) 不支持，Go 中的泛型同 java 一样都是编译时的特性，在编译时就确定好了类型。



#### 总结

|      |                          |              |                     |                       |
| ---- | :----------------------: | :----------: | :-----------------: | :-------------------: |
|      | Compile-time type safety | Type erasure | Runtime type safety | Runtime instantiation |
| Java |            ✓             |      ✓       |                     |                       |
| .NET |            ✓             |              |          ✓          |           ✓           |
| Go   |            ✓             |              |          ✓          |                       |



至此，泛型的学习就告一段落了，学到了语法，如何使用，已经在某些简单的场景下使用泛型带来的便利性。也或多或少的了解到了其他语言的泛型特性。



参考链接：

1. [go-generics-the-hard-way](https://github.com/akutz/go-generics-the-hard-way/)

