---
title: Call a Closure, RECURSIVEly?
date: 2023-09-04
tags: [dev,rust]
categories: [dev]
---

# Call a Closure, RECURSIVEly?

如果你需要递归调用一个函数，你会怎么做？这还不简单，直接在函数体内调用自己不就行了。

```rust
# rust
fn factorial(n: i32) -> i32 {
    if i <= 1 {
        1
    } else {
        n * factorial(n - 1)
    }
}
```

```racket
# lisp
> (define (fact n)
    (cond ((<= n 1) 1)
          (else (* n (fact (- n 1))))))
> (fact 5)
120
```

但是如果我们不使用 `define` ，只用闭包来定义这个函数，这种做法是否会出现问题呢？

```racket
(lambda (n)
  (cond ((<= n 1) 1)
        (else (* n (??? (- n 1))))))
```

当需要递归调用时，我们惊喜地发现，我们无法找到这个函数的名字，也就无从调用。而 Lambda 演算和图灵机是等价的，实际上是可以找到一种方法递归调用的（当然，实际的逻辑关系正好相反，正是因为 Lambda 演算拥有递归的能力才能够图灵等价）。

## 回调函数？

我们很容易想到一个最简单的方法：既然我们在闭包内无法获得这个闭包的名字，那么我们就给这个闭包添加一个参数，接受一个函数。只要这个参数的实参是闭包本身，就可以用这个参数的名字替代闭包，做到递归调用的效果。我们先从 Lisp 入手。

```racket
> ((lambda (n callback)
     (cond ((<= n 1) 1)
           (else (* n (callback (- n 1) callback)))))
   5
   (lambda (n callback)
     (cond ((<= n 1) 1)
           (else (* n (callback (- n 1) callback))))))
120
```

可以看到，为了递归调用，我们添加了 `callback` 参数，最后将闭包自身作为 `callback` 参数进行调用。调用这个函数需要使用类似这样的形式：`(fact n fact)` 。如果需要不止一个参数（阶乘只需要一个 n ），我们可以将 `callback` 的参数顺序调整到第一个，同时进行柯里化，就能得到这样的形式： `((fn fn) args ...)` 。

```racket
> (((lambda (fact)
     (lambda (n)
       (cond ((<= n 1) 1)
             (else (* n ((fact fact) (- n 1)))))))
   (lambda (fact)
     (lambda (n)
       (cond ((<= n 1) 1)
             (else (* n ((fact fact) (- n 1))))))))
   5)
120
```

对于这种形式 `(fn fn)` ，我们可以使用 ω 组合子来简化一下。

```racket
# 虽然这里用了 define , 但是可以当作是宏的定义
# omega = λf . f f
> (define omega
    (lambda (f)
      (f f)))
> ((omega (lambda (fact) (lambda (n)
                           (cond ((<= n 1) 1)
                                 (else (* n ((omega fact) (- n 1))))))))
   5)
120
```

ω 组合子的作用就是将闭包本身作为闭包的第一个参数。这样我们就能在闭包中调用到闭包自己了。

具体来说，如果我们拥有一个变量 `w: w -> a` ，它接受与自己类型相同的变量作为第一个参数，我们将其传入 ω 组合子，我们就可以通过 `(w w)` 得到一个 `a` ，同时，整个 ω 组合子表达式的结果也是 `a` 。得益于 ω 组合子，我们获得了给闭包起名字的能力（通过第一个参数）。

对于 Rust ，我们似乎也可以用同样的方法来解决这个问题。

```rust
let fact = |fact: ???, n: i32| {
    if n <= 1 {
        1
    } else {
        n * fact(fact, n - 1)
    }
}
```

这里需要解决一个问题，闭包的参数中， `fact` 的实际类型是什么？ `fact` 的实际类型是 `Self` ，其中 `Self: Fn(Self, i32) -> i32` 。我们可以发现，这实际上一个递归类型，这个递归类型让这个闭包可以无限递归，但是 Rust 无法接受这样一个可以无限递归的类型。因此，我们需要把这个闭包用 `Struct` 包裹起来，使用它提供的 `Self` 类型。

```rust
struct Wrapper<'a, I, O> {
    pub closure: &'a dyn Fn(&Self, I) -> O,
}

impl<'a, I, O> Wrapper<'a, I, O> {
    pub fn new(closure: &'a dyn Fn(&Self, I) -> O) -> Self {
        Self { closure }
    }

    pub fn call(&self, i: I) -> O {
        (self.closure)(self, i)
    }
}

fn main() {
    let fact = Wrapper::new(&|fact: &Wrapper<i32, i32>, n| {
        if n <= 1 {
            1
        } else {
            n * fact.call(n - 1)
        }
    });

    println!("fact(5) = {}", fact.call(5)); // fact(5) = 120
}
```

这里的 `Wrapper::call` 其实就是 ω 组合子，我们就不需要写 `fact.closure(&fact, n)` 了。

## 更简洁的形式


虽然我们已经实现了在闭包中递归调用，不过，每次调用都需要使用 ω 组合子或者把函数名写两遍，显得很累赘，形式不够优美(对于 Rust ，我们可以使用 Fn traits 来简化书写方式，以下不讨论)。为了简洁，我们需要将找到一个新的组合子，满足下面的条件：

```
((new-combinator f') n) = ((omega f) n)
其中
f: (f -> a -> b) -> a -> b
f': f'' -> a -> b
f'': a -> b
```

这个新的组合子与 ω 组合子的不同点在于， `f''` 不再需要一个函数作为参数，我们可以直接将 `a` 作为它的参数来调用，`(f'' a) = b`。

我们先写出这个新组合子的参数部分：

```racket
(lambda (f)
  (wrap f))
```

我们现在还不能知道如何实现 `wrap` 。但可以确定的一点是 `(wrap ...)` 的结果是一个 `a -> b` ，我们可以从 `new-combinator` 的签名得知。既然 `(wrap ...)` 的结果可以直接作为 `f''` ，我们不妨对这一点加以利用，使用 ω 组合子，让这个结果在 `wrap` 内部可用。

```racket
(lambda (f)
  (omega (lambda (g)
    (f (g g)))))
```

基于 ω 组合子的性质，我们可以通过 `(g g)` 获取这个闭包的结果（ `g` 实际上只有一个参数），也就是我们需要的 `f''`。不过，现在的组合子我们不能直接用在部分代码中——对于严格求值的语言，在调用函数前，需要先计算出 `(g g)` 的值，这是一个无限递归的过程。我们需要一个延迟计算的方式。在 `Racket` 中有两种方法：

- 使用 Lazy Lambda ，将参数求值延迟到实际使用参数的时候。对于这种方式，直接上面的定义就行了。
- 在用一个 Lambda 包裹住 `(g g)` 的求值部分。

对于第二种方式，实现如下。

```racket
(define Y
  (λ (f)
    (ω (λ (g)
         (f (λ (x) ((g g) x)))))))
```

这里用一个 `a -> b` 闭包包裹住了 `(g g)` ，这样，只有在传递一个 `a` 时才会触发 `(g g)` 的求值。这就是大名鼎鼎的 Y 组合子，它赋予了 Lambda 演算递归的能力，是一个非常强大的组合子。某些 Lisp 方言利用它实现了函数定义，[比如这门 Heresy 方言](https://github.com/jarcane/heresy/blob/master/lib/theory.rkt)。

我们可以试试 Y 组合子的威力。

```racket
> (define ω
    (λ (f)
      (f f)))
> (define Y
    (λ (target)
      (ω (λ (f)
           (target (λ (x) ((ω f) x)))))))
> (define factorial
    (Y (lambda (factorial)
         (lambda (n)
           (cond ((<= n 1) 1)
                 (else (* n (factorial (- n 1)))))))))
> (factorial 5)
120 # TADAA!
```

Y 组合子的形式定义如下：

```
ω = λf . f f
Y = λ f . ω (λ g . f (ω g))
  = λ f . (λ g . f (g g)) (λ g . f (g g))
```

