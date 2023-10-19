---
title: rCore CH2 Note
date: 2023-10-19
tags: [dev,rust]
categories: [dev]
---

# rCore Ch2 笔记

最近开始刷 [rCore 第三版教程](https://rcore-os.cn/rCore-Tutorial-Book-v3/index.html) ，尝试用 Rust 在 RISC-V 模拟器上实现一个操作系统内核。在这里留下一些实现过程中踩的坑以及相关的笔记。笔记从教程的第二章开始编号。之所以不从第一章开始，是因为第一章的内容不是很多，所以就和第二章一起写了，如果之前有写过操作系统相关或者 bare metal 的代码，很快就能上手。

对应的代码在[这里](https://github.com/NaviHX/rcore/tree/ch2)

## 启动流程

RISCV-V 芯片启动，进行一些简单的初始化后，就将运行放置到内存地址 0x1000 处的 SBI 代码（在我们平常使用的电脑上，负责同样功能的程序被叫做 BIOS ）对芯片进行初始化操作。在使用 qemu 运行的 rCore 内核实验中，我们使用了 [rustsib-qemu](https://github.com/rustsbi/rustsbi-qemu) 作为内核的引导程序。在完成初始化工作后， CPU 将会转到 Supervisor 特权级，并且跳转到 0x80200000 继续执行。关于特权级的描述稍后进行。

为了让我们的内核可以在 SBI 完成工作后可以运行，我们需要使用链接器脚本将内核的入口地址放到地址 0x80200000 处。

1. 将入口函数的段标记为 .text.entry 。
2. 在 linker.ld 中将 .text.entry 放到 0x80200000

```
SECTIONS {
    ...
    . = 0x80200000
    .text : {
        *(.text.entry)
        *(.text .text.*)
    }
    ...
}
```

### SBI 提供的服务

就像操作系统可以给应用程序提供系统调用的接口一样，SBI 也为 Supervisor 特权级运行的内核代码提供了接口使用硬件的服务。按照 SBI 的标准，我们可以向下面这样调用 SBI 提供的接口。

```rust
pub fn ecall(extension: usize, fid: usize, args: [3; usize]) -> usize {
    let mut ret;
    unsafe {
        asm!(
            "ecall",
            inlateout("x10") arg[0] => ret,
            in("x11") arg[1],
            in("x12") arg[2],
            in("x16") fid,
            in("x17") extension,
        );
    }
    ret
}
```

上面的例子可以调用一个最多三个参数的 SBI 接口。 ecall 指令可以让我们将 CPU 的特权级提升（或者不变），我们实现的操作系统内核也可以提供类似的 ABI 接口供应用程序调用，也就是系统调用。与 ecall 指令相对的，还有一个 eret 指令，可以将 CPU 退回到之前的特权级，关于中断与异常的部分会在「处理中断与异常」的部分作更多的介绍。

## RISCV-V Call Convention

当我们编写内核代码时，难免会进行函数调用。函数调用的过程中，可能会使用一些寄存器，其中的数值可能会被覆盖，为了让调用函数的过程可以成功继续执行，我们需要恢复调用者的寄存器状态。哪些寄存器需要保存，哪些不需要，每个寄存器什么用处，这就是 Call Conventions 「调用约定」（如果你曾经写过汇编，或者做过 FFI 相关的项目，你应该知道我在说什么）。

下面是 RISC-V 的调用约定。

| Name  | Register Number | Usage                              | Saver  |
|-------|-----------------|------------------------------------|--------|
| zero  | 0               | Hard-wired zero                    | N/A    |
| ra    | 1               | Return address                     | Caller |
| sp    | 2               | Stack pointer                      | Callee |
| gp    | 3               | Global pointer                     | N/A    |
| tp    | 4               | Thread pointer                     | N/A    |
| t0-2  | 5-7             | Temporaries                        | Caller |
| s0/fp | 8               | Frame pointer                      | Callee |
| s1    | 9               | Saved register                     | Callee |
| a0-1  | 10-11           | Function arguments / Return values | Caller |
| a2-7  | 12-17           | Function arguments                 | Caller |
| s2-11 | 18-27           | Saved registers                    | Callee |
| t3-6  | 28-31           | Temporaries                        | Caller |

在我们进行函数调用的地方，编译器都会为我们生成类似的代码：

```asm
__function:
    addi sp, sp, -64 # 分配栈
    sd ra, 56(sp) # 保存返回地址
    sd s0, 48(sp) # 保存 frame pointer
    addi s0, sp, 64 # 新的 frame pointer

    # Some code

    ld ra, 56(sp) # 恢复返回地址
    ld s0, 48(sp) # 恢复 frame pointer
    addi sp, sp, 64 # 恢复栈
    ret
```

为了让我们的内核支持函数调用，我们需要做到以下几点。

1. 为函数调用提供一个栈，将 sp 指向栈的顶部（因为栈是向下增长的）。
2. 为每一次函数调用的周围做好执行环境的保存与恢复工作。这一项工作通常有编译器帮助我们执行。

## 特权级

为了让应用程序出现故障时，不会让整个计算机停止运行，同时为了限制应用程序进行某些需要特权的操作， RISC-V 提供了多个特权级： Machine 、Hypervisor 、Supervisor 、User 。当应用程序发生错误时，会陷入到内核中进行处理，终止应用程序的执行，恢复整个系统的运行。也可以使用 ecall 调用系统提供的服务。

- ecall : 可以使得 CPU 提升到不低于当前特权级的状态。
- eret : 可以使得 CPU 恢复到不高于当前特权级的状态。

这两条命令进行组合，就可以实现系统调用的功能，向应用程序提供服务。

## 处理中断和异常

在我们实现的内核中，我们要做的事情非常简单：当应用程序 trap 到内核中后，调用相应的代码处理异常 / 中断，最后返回用户态执行。在处理中断的过程中，需要注意这些寄存器：

| CSR     | Description                          |
|---------|--------------------------------------|
| sstatus | 发生 trap 前 CPU 处于的特权级        |
| sepc    | 发生 trap 前执行的最后一条指令的地址 |
| scause  | 描述 trap 的原因                     |
| stval   | trap 的附加信息                      |
| stvec   | 处理 trap 的代码地址                 |

为了能够处理中断，我们需要完成下面的工作：

1. 编写处理 trap 的代码，把地址写入 stvec 寄存器中。这里 stvec 被设置为了 Direct 模式，直接写入地址就行了。
2. 编写恢复用户态的代码。
3. 在进入 trap 后，需要将 sp 设置到内核栈。返回时重新指向用户栈。（用 sscratch 保存）

为了正确恢复用户态的执行，我们需要保存所有通用寄存器的数值到内核栈中——这里不存在由调用者保存与由被调用者保存，所有的寄存器在内核处理其间都有可能被使用。同时需要保存 sepc 的寄存器的值，这里考虑到了嵌套 trap 与切换任务的可能（在 Ch3 中有涉及）。

在 rCore 的代码中我们可以发现这样一个结构。

```asm
__alltraps:
    # 保存状态到栈上
    # 调用处理 trap 各种情况的函数

__restore:
    # 恢复状态
    # 返回用户态
    sret
```

这里看似有两个函数 __alltraps 和 __restore ，但在调用 alltraps 的过程中只有一个函数，因为在 restore 前并没有 ret 。restore 可以用于运行第一个任务：构造一个保存用户态状态的地址，将 sp 指向这个地址，调用 restore 就可以运行构造的程序。

## 运行应用程序

通过构造「用户态寄存器状态」调用 restore 即可执行。
