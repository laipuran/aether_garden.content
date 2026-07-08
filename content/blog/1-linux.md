---
slug: 1-linux
title: 早期 Linux 的 Copy On Write
date: 2026-07-09
excerpt: 为什么 Kernel 中在 main() fork 之后调用的函数都需要 inline
tags:
  - linux
  - kernel
status: published
updatedAt: 2026-07-09
---

### Linus 在 main.c 中写的注释：

```C
/*
* we need this inline - forking from kernel space will result
* in NO COPY ON WRITE (!!!), until an execve is executed. This
* is no problem, but for the stack. This is handled by not letting
* main() use the stack at all after fork(). Thus, no function
* calls - which means inline code for fork too, as otherwise we
* would use the stack upon exit from 'fork()'.
*
* Actually only pause and fork are needed inline, so that there
* won't be any messing with the stack from main(), but we define
* some others too.
*/
```

意思就是，Copy On Write 这个特性是基于 CPU 写保护特性的。在 x86 上，CR0 寄存器的 WP 位控制内核态是否遵守页级写保护：WP=0 时内核写入只读页不会触发 page fault，COW 就不会发生。早期 Linux 确实如此，所以 fork 后内核空间页面没有 COW，父子进程共享同一份物理内存。

但这里的问题**主要不是"写只读页"**。

fork 之后**所有页面都是共享可写的**，包括栈——问题恰恰出在这里。如果 `fork()` 是个普通函数调用，父子进程返回后都会操作同一块栈内存，互相踩踏。所以 Linus 把 `fork()` 及之后的代码全做成内联展开，`main()` 不再依赖栈，从而避免冲突，直到子进程 `execve()` 替换地址空间。