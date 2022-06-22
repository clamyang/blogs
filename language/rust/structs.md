---
title: Rust-Structs
comment: true
---

rust 中的结构体和方法（method）。

<!--more-->

# struct

其实各种语言对 Struct 的实例化都差不多，主要就是关注下 Rust 里关于 struct 的语法。

```c
// struct from C
struct foo {
    int age;
    char *name;
};
```



```rust
// struct from Rust
struct bar {
    active: bool,
    username: String,
    email: String,
    sign_in_count: u64,
}
```



```go
// struct from Go
type foobar struct {
    name string
    age int
    email string
}
```



都差不多..



## 打印结构体

没想到 rust 里边打印一个结构体这么麻烦，如果说打印某个字段直接用 `println!("{}", xx.xx);` 就行，但是要是打印整个结构体中的内容，还得给结构体定义前加上 `#[derive(Debug)] `..这样才能打印出调试信息。

```rust
#[derive(Debug)]
struct User {
    name: String,
    age: u32,
}
```



> 输出完整结构体算是调试功能，但是我们需要手动添加相关内容让某个结构体使用这个功能。



添加完以后才能输出完整的结构体：

```rust
// pretty print
println!("{:#?}", xx);
```



## dbg!

可以用来输出调试信息，`dbg!(xx);`





# method

method 的第一个参数永远是 self。	

```rust
struct User {
    name: String,
    age: u32,
}

impl User {
    fn print_name(&self) -> &String {
        &self.name
    }
}

```

