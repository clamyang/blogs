---
title: Rust-Enums
comment: true
---

Rust 枚举

<!--more-->

**定义、声明**

```rust
enmu IpAddrKind {
    V4,
    V6,
}
```



定义了一个枚举数据类型，IpAddrKind，枚举中有两个变量，V4，V6。但是这样存在一个问题，实际的 IP 地址应该存储在哪里，这只是定义了两种 Ip 类型。



如何使用 enum 存储值？

方法一，使用 struct：

```rust
struct IpAddr {
    kind: IpAddrKind,
    address: String,
}

let home = IpAddr {
    kind: IpAddrKind::V4,
    address: String::from("127.0.0.1"),
};

let loopback = IpAddr {
    kind: IpAddrKind::V6,
    address: String::from("::1"),
};
```



方法二，使用 enum 存储：

直接使用 enum 存储值，这个有点意思..

```rust
enum IpAddr {
    V4(String),
    V6(String),
}

let home = IpAddr::V4(String::from("127.0.0.1"));

let loopback = IpAddr::V6(String::from("::1"));

```



标准库中如何定义 IpAddr 的：

```rust

struct Ipv4Addr {
    // --snip--
}

struct Ipv6Addr {
    // --snip--
}

enum IpAddr {
    V4(Ipv4Addr),
    V6(Ipv6Addr),
}

```



**可以给 enum 添加 method..**这里其实 Go 也可以实现：

```go
// 类似这样的代码
type Strings []int

func (s Strings) Less() bool {return true}
```



**The `Option` Enum**

