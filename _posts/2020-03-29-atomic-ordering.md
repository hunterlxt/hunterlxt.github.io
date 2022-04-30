---
layout: post
title: 聊一聊 Memory Order
subtitle: 副标题
date: 2020-03-29
tags: [rust]
---

在学习 C++ 的时候接触了 Memory Order 的概念，今天刷知乎时看到有人讨论，写了几个 example 做了验证，发现之前对这个东西的理解有偏差。这次重新学习后做一个整理。

## Atomic

Rust 提供了一种用于并发中的数据类型即 atomic，其用于在线程之间同步信息。对它的操作要输入一个 `Ordering` 表示内存屏障的类型。这一块内容和 C++ 中的相同 [C++20 Draft](http://eel.is/c++draft/atomics.order)。那为什么需要对 atomic 的操作做出这样的屏障处理呢？

## 指令重排

学习体系结构的时候我们一般是在所谓的单核模型下进行的。在具有缓存模型的内存系统中，为了让指令运行的更快，会将内存中的数据放置处理器的缓存中，但是缓存和内存的数据可能会出现不一致的现象，因为其它核也可能会对这个数据进行操作，MESI 协议是约束这种不一致的，以保证不会出现因独立缓存带来的问题。硬件会对某些指令进行重排序以保证数据在不违反依赖性的情况加快运算，但只能保证单线程模型下的依赖性不被违反，对线程不做保证。所以多线程模型下，其它的线程并不知道**此**线程的真实读写顺序，毕竟**此**线程对数据的读写更愿意在自己的缓存中进行，而非所有人可见的全局空间——内存。

当我们需要处理需要同步的数据时，需要一种顺序来约束这种失控的行为。注意：这种所谓的重排在不同体系架构下的机器行为是不同的，因此不用担心在更强约束的机器比如 X86 使用强排序约束会造成性能上的损失，但在类似 ARM 的机器上或许会损失性能。【其实主观上理解这个排序行为可能被特定的体系结构保证了，意思就是它是未定义的行为，最好不要在 X86 上默认它的行为，该使用 Ordering 保证还要使用的】

不管是编译器或者处理器硬件，都会在保证单线程正确的前提下对数据访问进行重排。所以一般我们用信号量、锁等机制做数据同步。

## Rust 中的顺序保证

Rust 对于原子数据的访存提供了 4 种顺序（C++ 提供了 6 种），分别是 Sequentially Consistent、Release、Acquire、Relaxed。

### Sequentially Consistent

这是一种最强的内存栅栏，在这之前和之后的内存指令都不能跨越它。在 Rust 官方的 nomicon 中建议没有特别强烈的需要使用 `SeqCst` 和 `Relaxed` 就够了，运行稍慢总比出错好。

### Acquire-Release

这个理解起来也不难，当 Acquire 模式去**读到**某个原子量的修改值时，用 Release 模式修改相应值的指令之前的所有 store 指令都对 Acquire 之后的所有 load 指令可见。

**注意上一段话**，Acquire 和 Release 并不是锁，有几篇文章将其比喻为 lock 和 unlock，这个比喻极具误导。只有当 Acquire 读取的值是 Release 模式放进去的值才成立对应约束关系，**不是** Acquire 读取阻塞在这里，一直等到所有 Release 之前的操作做完！

那如何将其稍加添加以变成 mutex 模式呢？可以用 `compare_and_swap`，例如：

```rust
while lock.compare_and_swap(false, true, Ordering::Acquire) { }
// so we successfully acquired the lock!
// ...
// ok we're done, release the lock
lock.store(false, Ordering::Release);
```

### Relaxed

这个就是最宽松的语义，仅仅保证所处理的原子变量本身是原子的，没有任何其它保证了。比如你处理一个共享计数器，就可以用它，因为你只是想记录一下总数而已。

## 总结和测试

根据我的经验，到底用什么语义取决于：你希望对所处理的 atomic 的上下文进行什么样的保证。根据上下文互相同步、仅上文和下文同步、上下文无关选择这 3 个 ordering 模型就行了。

那么用 atomic 实现 mutex 和 `sync::Mutex` 谁更快呢？经过一个小实验测试对比了多次数据，平均 atomic 耗时 1.54s，而 Mutex 平均耗时 1.75s。但是 `compare_and_swap` 会耗费一个 CPU 核心做比较的工作，所以单纯作为锁使用，不推荐使用 atomic。



测试代码逻辑如下：

```rust
// 使用 compare_and_swap 创建两个线程
for _ in 0..10000 {
    while a.compare_and_swap(false, true, Ordering::Acquire) {}
    do_something();
    a.store(false, Ordering::Release);
}
// 使用 Mutex 创建两个线程
for _ in 0..10000 {
    let _lk = m.lock().unwrap();
    do_something();
}
// 睡一下做个样子
fn do_something() {
    thread::sleep(Duration::from_micros(10));
}
```
