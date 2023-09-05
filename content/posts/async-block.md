---
title: Why is this async block `FnOnce`
date: 2023-03-16
tags: [dev,rust,async]
categories: [dev]
---
# Intro

写这篇post的原因是, 在给机器人写 warp server 的代码时,
遇到了下面这个编译错误:

``` rust
    warp::path!("select" / "coordinate")
        .and(warp::post())
        .and(warp::body::content_length_limit(1024))
        .and(warp::body::json())
        .then(|SelectCoordinate{x, y}: SelectCoordinate| async move {
                if let Err(_) = update_sender.send(UpdateMessage::Update(StateUpdate::AddNewRouteByCoordinate(crate::robot::Coordinate(x, y)))).await {
                    return_code(1)
                } else {
                    return_code(0)
                }
        })


error[E0525]: expected a closure that implements the `Fn` trait, but this closure only implements `FnOnce`
  --> robot_admin/src/route.rs:92:15
   |
92 |         .then(|SelectCoordinate{x, y}: SelectCoordinate| async move {
   |          ---- ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^ this closure implements `FnOnce`, not `Fn`
   |          |
   |          the requirement to implement `Fn` derives from here
93 |                 if let Err(_) = update_sender.send(UpdateMessage::Update(StateUpdate::AddNewRouteByCoordinate(crate:...
   |                                 ------------- closure is `FnOnce` because it moves the variable `update_sender` out of 
its environment
   |
   = note: required for `[closure@robot_admin/src/route.rs:92:15: 92:57]` to implement `warp::generic::Func<(SelectCoordinat
e,)>`

For more information about this error, try `rustc --explain E0525`.
```

这段代码生成一个 `POST /select/coordinate` 的路由, 它接收一个json
form并更新机器人的路径信息. 这个更新的工作交由一个 "异步闭包" 来完成.

我并 " 没有将 `update_sender` 消耗或者移动所有权 ", 但是这个闭包却是
`FnOnce` 而不是 `Fn` , 不符合要求, 编译失败.
这种情况可以简化成下面的代码:

``` rust
use std::future::Future;

#[derive(Debug, Clone)]
struct S { pub a: i32 }

fn take_fn<T: Future<Output = ()>>(_f: impl Fn() -> T) -> () {}
fn take_fnonce<T: Future<Output = ()>>(_f: impl FnOnce() -> T) -> () {}

fn main() {
    // TODO Why is `f` a `FnOnce`, which means that `f` takes the ownership of s and consumes it.
    let s = S { a: 1 };
    let f = || async move {
        println!("a is {:?}", s);
    };
    take_fn(f);     // ERROR!
    take_fnonce(f); // OK!

    println!("Hello world!");
}
```

[这个Playground可以直接尝试](https://play.rust-lang.org/?version=stable&mode=debug&edition=2021&gist=3029d58d12f93f8a744481a5b6c1aa9e)

# `f` 是一个异步闭包吗?

`rust` 的异步闭包的语法是 `async [move] |_| { /* block */ }` .
但是注意看 `f` 的声明, `async move` 写在了参数的后面,
说明这不是一个异步闭包, 而只是一个普通的闭包, 它返回一个 `impl Future` .
`take_fn*` 的签名也证明了这一点.

异步闭包[现在是一个不稳定的特性](https://github.com/rust-lang/rust/issues/62290),
一般都通过一个返回 `Future` / 异步 block 的普通闭包实现. `FnOnce`
说明了我们写的这个 闭包确实消耗了所有权, 只能执行一次 (对比 `Fn` ,
它并不消耗所有权, 所以可以执行任意多次).

# 为什么是 `FnOnce` ?

`f` 返回的异步 block 是 move 块, 说明这个块会获得捕获变量的所有权.
包裹这个异步块的闭包捕获了 `s`, 随后又被异步块获取了 所有权.
异步块作为返回值, 将 `s` 的所有权转移到了外部, 这个时候闭包已经不拥有
`s` 的所有权了, 所以为 `FnOnce` .

所有权的转移过程也可以像下面这样展示:

``` rust
let f = || {                      // Closure 从环境中捕获了 s
    async move {                  // Async Block 使用 move 捕获, 直接获取所有权而不是借用
        println!("a is {:?}", s); // Async Block 从 Closure 中捕获了 s
    }                             // Async Block 结束
};                                // s 的所有权和 Async Block 作为返回值一同转移到了外部
```

解决的办法就是在闭包中保留所有权, 比如说直接 `Clone(&self)` 或者使用
`Arc` 来进行管理.

``` rust
let s = S { a: 1 };
let f = move || {
    let s = s.clone();
    async move {
        println!("a is {:?}", s);
    }
};
take_fn(f); // TADAA!
```
