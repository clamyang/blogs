---
title: Rust-Ownership
comment: true
---

Rust 核心概念了解 —— Ownership

<!--more-->

Ownership 就是定义了一些规则，比如在函数中传递参数的时候是怎么处理参数的，官网的解释为**Rust程序如何管理内存的一系列规则**。



**内存管理的三种方式**

- Go 语言这类，自动垃圾回收机制
- C 语言，手动垃圾回收，显示分配、释放
- Rust 独一档，结合**编译器、ownership 定义的规则**，如果说没有按照 Rust Ownership 定义的规则，那么编译就不会通过。



第一次知道，还能有这样的内存管理方式..



# ownership by example

（通过例子学习 ownership）



> The Stack and the Heap

`All data stored on the stack must have a known, fixed size. Data with an unknown size at compile time or a size that might change must be stored on the heap instead.`

`Accessing data in the heap is slower than accessing data on the stack because you have to follow a pointer to get there. `



## ownership Rules

- Rust 中每一个值都有一个被叫做 `owner` 的变量，比如 a = 1, 1 的 owner 就是 a。
- 同一时刻有且仅有一个 owner。
- 当 owner 超出范围时（超出作用域），value 会被丢弃。



## 变量作用域

作用域大家了解的已经很多了，要是直接看 C 代码其实还是有点懵逼的。。不信你试试。。

```c
#include <stdio.h>

int main(void) {
    int x = 1;
    {
        int x = 2;
        printf("%d\n", x);
    }
    printf("%d\n", x);
    return 0;
}
```



> 很显然，答案不是 2 ，2； 正确答案为：2， 1；



Rust 中其实也类似，如下：

```rust
fn main() {
    {
        let s = "hello";	// 这时候变量开始有效
    } 					    // 从这时起，就无效
}
```



两个重要的点：

- 变量 s 在作用域内，有效
- 变量 s 超出作用域，无效



## 内存和分配

Rust 中内存释放的方式，**超出了作用域自动释放**。Rust 也算是帮我们做了内存管理，虽然没有 GC，但是也不用我们手动进行 free，避免出现 double free 问题，如上述分配的 s 变量，超出了作用域就被释放掉。



## ownership Move

Stack 

```rust
fn main() {
    let x = 1;
    let y = x;
}
```

Heap

``` rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1;
}
```



两段代码表达的意思相近，但是，在不同的内存空间上有着很大的差别。

- stack 不赘述

- heap

  ![s1 and s2 pointing to the same value](https://doc.rust-lang.org/book/img/trpl04-02.svg)



执行完赋值操作后，有两个指向同一块内存的指针，但是这也会存在一种情况，当 s1 和 s2 都用不到时候，会进行 free，就会出现 double free 的情况。



所以，为了保证内存安全，在 `let s2 = s1;` 执行完成后，Rust **认为 s1 不再有效**。即，将 s1 ownership move to s2.



## ownership Clone

(deeply copy)

```rust
fn main() {
    let s1 = String::from("hello");
    let s2 = s1.Copy();
    
    println!("s1 is {}, s2 is {}", s1, s2);
}
```



![s1 and s2 to two places](https://doc.rust-lang.org/book/img/trpl04-03.svg)



## Stack-Only Data: Copy

`The reason is that types such as integers that have a known size at compile time are stored entirely on the stack, so copies of the actual values are quick to make. That means there’s no reason we would want to prevent `x` from being valid after we create the variable `y`. `

总而言之，栈上的数据会进行 Copy，堆上的数据会进行 move。



## 函数之间 ownership 的改变

将变量传递给函数的时候也可能会发生 move 或 copy，和分配差不多。

```rust
fn main() {
    let s = String::from("hello");  // s comes into scope

    takes_ownership(s);             // s's value moves into the function...
    // ... and so is no longer valid here

    let x = 5;                      // x comes into scope

    makes_copy(x);                  // x would move into the function,
    // but i32 is Copy, so it's okay to still
    // use x afterward

} // Here, x goes out of scope, then s. But because s's value was moved, nothing
// special happens.

fn takes_ownership(some_string: String) { // some_string comes into scope
    println!("{}", some_string);
} // Here, some_string goes out of scope and `drop` is called. The backing
// memory is freed.

fn makes_copy(some_integer: i32) { // some_integer comes into scope
    println!("{}", some_integer);
} // Here, some_integer goes out of scope. Nothing special happens.

```



## 函数返回时与 ownership

返回值也会转移 ownership。

```rust
fn main() {
    let s1 = gives_ownership();         // gives_ownership moves its return
    // value into s1

    let s2 = String::from("hello");     // s2 comes into scope

    let s3 = takes_and_gives_back(s2);  // s2 is moved into
    // takes_and_gives_back, which also
    // moves its return value into s3
} // Here, s3 goes out of scope and is dropped. s2 was moved, so nothing
// happens. s1 goes out of scope and is dropped.

fn gives_ownership() -> String {             // gives_ownership will move its
    // return value into the function
    // that calls it

    let some_string = String::from("yours"); // some_string comes into scope

    some_string                              // some_string is returned and
    // moves out to the calling
    // function
}

// This function takes a String and returns one
fn takes_and_gives_back(a_string: String) -> String { // a_string comes into
    // scope

    a_string  // a_string is returned and moves out to the calling function
}

```



综上所述，每次进行函数传参的时候都会发生 ownership 的转移，如果说我们给 funcA 传一个 A 参数后，仍然要使用 A 参数，应该怎么办呢？

-  可以把这个参数从 funcA 中返回

```rust
fn main() {
    let s1 = String::from("hello");
    let (s2, len) = calculate_length(s1);
    println!("The length of string {} is {}", s2, len);
}

fn calculate_length(s: string) -> (String, usize) {
    let length = s.len();
    (s, length)
}
```

- 牛x的 Rust 当然还提供了另一种方式，**reference**。



## Reference and Borrowing

### Reference

本以为这个 reference 和 pointer 是一个东西，但是 rust 中貌似并不是这么定义的，且看且分析。

` A reference is like a pointer in that it’s an address we can follow to access data stored at that address that is owned by some other variable.`



reference 和 pointer 相同的点，存储的都是地址，可以通过这个地址访问存储在这个地址上的数据，这个数据可能是属于别的变量的。 **不同的点**，reference 指向的永远是**有效**的地址。

![&String s pointing at String s1](https://doc.rust-lang.org/book/img/trpl04-05.svg)

因此，上述代码就可以就改为，

```rust
fn main() {
    let s1 = String::from("hello");

    let len = calculate_length(&s1);

    println!("The length of '{}' is {}.", s1, len);
}

fn calculate_length(s: &String) -> usize { // s is a reference to a String
    s.len()
}
```



下面这段引用，再一次解释了，传递引用给函数时发生了什么，不再赘述。

> `When functions have references as parameters instead of the actual values, we won’t need to return the values in order to give back ownership, because we never had ownership.`



### Borrowing

（把创建 reference 的行为定义成 borrowing）。在实际生活中，就跟借东西是一个意思，假设一个人拥有一辆保时捷，你借过来开两天，然后还回去，我们从未拥有过保时捷。



然后问题就来了，比如我们接过来保时捷开两天，发现他的颜色看着不顺眼，你想给他改装，这时候怎么办？如下。

```rust
fn main() {
    let s1 = String::from("hello");

    change(&s1);
}

fn change(some_str: &String) {     
    some_str.push_str("RTFM");     
}
```

```
error[E0596]: cannot borrow `*some_str` as mutable, as it is behind a `&` reference
 --> src/main.rs:8:5
  |
7 | fn change(some_str: &String) {
  |                     ------- help: consider changing this to be a mutable reference: `&mut String`
8 |     some_str.push_str("RTFM");
  |     ^^^^^^^^^^^^^^^^^^^^^^^^^ `some_str` is a `&` reference, so the data it refers to cannot be borrowed as mutable

For more information about this error, try `rustc --explain E0596`.
error: could not compile `borrowing` due to previous error
```



验证了一个结论，默认情况加，**borrow**过来的东西是不可修改的，除非加上**mut**(mutable)，如下：

```
fn main() {
    let mut s1 = String::from("hello");

    change(&mut s1);
}

fn change(some_str: &mut String) {     
    some_str.push_str("RTFM");     
}
```

这样就可肆无忌惮的修改 reference 指向的内容了，形如这样的被称为 `mutable reference`。



Mutable referenct 一个最大限制：**在同一时刻只能拥有某个变量的一个 mut reference**。可以试试下面这段代码：

```rust
fn main() {
    let mut s1 = String::from("RTFM");

    let r1 = &mut s1;
    let r2 = &mut s2;

    println!("{} {}", r1, r2);    
}
```

```
error[E0499]: cannot borrow `s1` as mutable more than once at a time
 --> src/main.rs:5:14
  |
4 |     let s2 = &mut s1;
  |              ------- first mutable borrow occurs here
5 |     let s3 = &mut s1;
  |              ^^^^^^^ second mutable borrow occurs here
6 | 
7 |     println!("{} {}", s2, s3);
  |                       -- first borrow later used here

For more information about this error, try `rustc --explain E0499`.
error: could not compile `borrowing` due to previous error
```



**mutable reference and immutable reference**

**不能同时拥有可变的和不可变的 reference**，意味着，要么只有一个 mutable，要么有多个 immutable，不能有一个 mutable 和多个 immutable 的情况。



## Reference scope

reference 的生效范围，**Note that a reference’s scope starts from where it is introduced and continues through the last time that reference is used. **

```rust
fn main() {
    let mut s = String::from("hello");

    let r1 = &s; // no problem
    let r2 = &s; // no problem
    println!("{} and {}", r1, r2);
    // variables r1 and r2 will not be used after this point

    let r3 = &mut s; // no problem
    println!("{}", r3);
}
```



## Dangling References

悬垂引用，有点类似 dangling pointer， rust 中，编译器会检查 reference 指向的内容是否有效，不会发生这种情况。。

```rust
fn main() {
    let reference_to_nothing = dangle();
}

fn dangle() -> &String { // dangle returns a reference to a String

    let s = String::from("hello"); // s is a new String

    &s // we return a reference to the String, s
} // Here, s goes out of scope, and is dropped. Its memory goes away.
// Danger!
```

```
$ cargo run
   Compiling ownership v0.1.0 (file:///projects/ownership)
error[E0106]: missing lifetime specifier
 --> src/main.rs:5:16
  |
5 | fn dangle() -> &String {
  |                ^ expected named lifetime parameter
  |
  = help: this function's return type contains a borrowed value, but there is no value for it to be borrowed from
help: consider using the `'static` lifetime
  |
5 | fn dangle() -> &'static String {
  |                ~~~~~~~~

For more information about this error, try `rustc --explain E0106`.
error: could not compile `ownership` due to previous error
```



## Rules of Reference

- 任何时刻，要么只能有一个 mutable reference，要么有多个 immutable reference。
- Reference 指向的内容一定是有效的。



## Slices

这个跟 Go 的切片引用类似..，直接贴张图，不再赘述。

![world containing a pointer to the byte at index 6 of String s and a length 5](https://doc.rust-lang.org/book/img/trpl04-06.svg)

## 完结撒花

