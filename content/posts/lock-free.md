+++
title = "Exploring Lock-freedom"
date = "2024-03-07"
tags = ["dev"]
categories = ["dev"]
+++

# 探索无锁

```rust
struct Stack<T> {
    head: Option<Box<Node<T>>>,
}

struct Node<T> {
    val: T,
    next: Option<Box<Node<T>>>,
}

impl<T> Stack<T> {
    pub fn new() -> Self {
        Self { head: None }
    }

    pub fn pop(&mut self) -> Option<T> {
        if let Some(head) = self.head.take() {
            self.head = head.next;
            Some(head.val)
        } else {
            None
        }
    }

    pub fn push(&mut self, val: T) {
        let mut new_head = Box::new(Node { val, next: None });

        new_head.next = self.head;
        self.head = Some(new_head);
    }
}
```

实现一个只用于单线程的栈是非常容易的。如果想在多线程环境中使用这个栈实现，最简单的方法就是使用一个 `Mutex` ，它可以在多线程环境中提供互斥的可变引用。然而， `Rust` 内部默认使用的实现为 `pthread_mutex` ，性能损失比较大。对于性能敏感的环境，我们更倾向于使用无锁的数据结构。（虽然也存在 [parking_log](https://webkit.org/blog/6161/locking-in-webkit/) 这类在低竞争环境下性能尚可的锁）

## Atomic

在将这个栈实现修改为无锁版本之前，我们先观察一下单线程环境中的实现。为了弹出栈顶的一个元素，我们需要将栈顶赋值为当前栈顶的下一个元素。为了压入一个元素，我们需要构造一个新元素，并将新元素的 `next` 指针指向原栈顶，然后将栈顶赋值为新元素。使用不提供互斥访问的容器在多个线程共享同一个栈，这违背了 `Rust` 的所有权规则，可能会引起 Bug ：当一个线程读取 `next` 后，重新设置 `head` 前，可能已经有另一个线程将 `head` 值进行了修改并释放了内存，重新赋值后造成了 use after free 问题。

造成这种错误有两个原因：

- 线程调度。在更换 `head` 值的过程中切换到了另外一个线程执行。
- 修改扩散。当前线程对值的修改可能还没有扩散到其他核心的缓存中，其他核心可能会读取到以前的值。

最朴素的想法就是在重新赋值前检查以下 `head` 是否是原本的值。

```rust
let head = self.head;
let next = head.next;
// The thread may yield CPU here.
if head == self.head {
    self.head = next;
} else {
    // Pop failed.
}
```

然而简单的并不是好的，这种做法不能解决问题：在当前线程判断 `head` 值之后，仍旧可能发生线程调度；同时对缓存的修改也不一定扩散到了其他核心。所以，要想在不使用锁的情况下解决这个问题，解决方案需要满足一下两个必要条件：

- 原子性。比较和赋值作为同一个原子指令，其间不可调度。
- 内存屏障。修改后其他核心必须知道这个值需要重新加载进入到缓存中。

[`Rust` 的 `Atomic` 可以帮助我们实现这两个条件](https://doc.rust-lang.org/nomicon/atomics.html)。一个 `Atomic*` 提供了 `compare_and_swap` / `compare_exchange` 方法，可以进行原子地比较和替换。同时作为参数的 `Ordering` 提供了与其他对内存操作的事件的 `happen before` 关系进行了抽象（换句话说，提供了类似 `fence` 指令的功能）。因此，我们可以把栈的实现改成如下这种利用了 `Atomic` 的形式：

```rust
use std::ptr::{self, null_mut, NonNull};
use std::sync::atomic::AtomicPtr;
use std::sync::atomic::Ordering::{Acquire, Relaxed, Release};

pub struct Stack<T> {
    head: AtomicPtr<Node<T>>,
}

struct Node<T> {
    val: T,
    next: Option<NonNull<Node<T>>>,
}

impl<T> Stack<T> {
    pub fn new() -> Stack<T> {
        Stack {
            head: AtomicPtr::new(null_mut()),
        }
    }
}

impl<T> Stack<T> {
    pub fn pop(&self) -> Option<T> {
        loop {
            let head = self.head.load(Acquire);

            if head == null_mut() {
                return None;
            } else {
                let next = unsafe { (*head).next.map(|n| n.as_ptr()).unwrap_or(null_mut()) };
                if self.head.compare_and_swap(head, next, Release) == head {
                    return Some(unsafe { ptr::read(&(*head).val) });
                }
            }
        }
    }

    pub fn push(&self, t: T) {
        let n = Box::into_raw(Box::new(Node { val: t, next: None }));
        loop {
            let head = self.head.load(Relaxed);
            unsafe {
                (*n).next = NonNull::new(head);
            }
            if self.head.compare_and_swap(head, n, Release) == head {
                break;
            }
        }
    }
}
```

这个实现提供了内部可变性，所以 `push` 和 `pop` 方法现在只需要接受不可变引用。同时，由于在检查到原本的 `head` 被修改后会先放弃本次修改，随后重新尝试加入或删除元素，所以这个结构是 `Sync` 的（内部可变性是互斥的）。

## 内存泄露

如果程序中使用了大量的 `push` 和 `pop` 操作，以上的实现会发生内存泄露：因为我们根本没有回收内存。注意到在 `pop` 的实现中，之前在 `push` 中利用 `Box` 分配的内存地址作为 `head` ，但是在我们丢弃 `head` 时，并没有将其转换为 `Box` ，这造成了内存泄露。这是为了正确性有意为之。假设我们在 `pop` 实现中加入了释放内存的代码。

```rust
if self.head.compare_and_swap(head, next, Release) == head {
    let val = Some(unsafe { ptr::read(&(*head).val) });
    // Box the value and drop it.
    unsafe { Box::from_raw(head); }
    return val;
}
```

## ABA Problem

现在我们拥有一个栈 `A -> B -> C` ，每一个字母表示一个内存地址，箭头表示该内存地址的 `next` 指向的地址。假设我们现在有两个线程同时分别执行下面的操作：

- Thread 1 ：弹出一个元素。
- Thread 2 : 两次弹出元素，压入一个元素。

此时， 在 Thread 1 读入 `head` 值后，调度程序让 Thread 1 停止，同时 Thread 2 开始执行。按照我们的实现，Thread 2 将会释放 A 和 B 对应的内存。在 `push` 中，Thread 2 申请了一个新内存空间用于安放新的头部节点。此时问题出现：分配器有可能会重用已经被释放的内存。也就是说，分配器分配的新地址可能会是 A 。此时，如果 Thread 1 开始执行：

```rust
let next = unsafe { (*head).next.map(|n| n.as_ptr()).unwrap_or(null_mut()) };
// Thread 1 stopped here before.

// Thread 1 pass the check, because the allocator allocates the same address for the new head.
if self.head.compare_and_swap(head, next, Release) == head {
    return Some(unsafe { ptr::read(&(*head).val) });
    // Because Thread 1 remember the previous next, which points to B, which is dangling now,
    // future accesses will panic due to `use after free`! 🤯
}
```

造成这个问题的原因主要有两个：

- 在删除一个节点时，我们无法保证没有其他线程正在使用这个指针，因为其他线程可能记录了将被删除的指针的值。
- 内存复用。

所以在实现了垃圾回收机制的语言不存在这个问题。对于这种问题的解决方案的关注点都在于：我们何时能够释放一个节点。在 [Crossbeam](https://github.com/crossbeam-rs/crossbeam) 中使用了 EBR 来解决这个问题。

## Epoch-based Reclamation

实现这个机制需要：

- 一个全局 epoch 计数器（epoch 数值合理范围为 0-2 ）。
- 每一个 epoch 的全局垃圾容器。
- 每一个线程的激活标记。
- 每一个线程的 epoch 计数器。

与传统的垃圾回收不同的是，该机制并不需要通过图找到那些实际不再需要的垃圾，而是通过 epoch 来回收：需要清理的内存在当前 epoch 的两代之前。具体的步骤如下：

当一个线程需要对无锁结构进行操作时，它激活自己的标记，然后将自己的计数器更新至与全局相同的值。如果它需要移除一个内存地址，它所做的并不是直接将其释放，而是 **把内存放入到对应 epoch 的垃圾容器内** 。最后将标记改为未激活。

当一个线程需要进行回收时，需要检查所有的激活线程是否都在当前的 epoch 中。如果都在当前 epoch ，那么就将全局计数器加一，如果修改成功，就可以清理两代之前的内存。这样可以保证所有当前正在被引用的内存都在上一代，且两代之前的内存已经被清理。

