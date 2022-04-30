---
layout: post
title: 让 TiKV 愉快地使用 async/await 异步编程模型
subtitle: 副标题
date: 2020-11-01
tags: [tikv, rust]
---

几个月前，接了一个大坑，因为 `std::future` 趋于稳定，官方的 `async/await` 也进入了 stable 版本，作为坚决拥护 rust 社区的项目，为 TiKV 升级 future 并引入 `async/await` 也被提上了日程。这里主要总结下作为 Rust 菜🐔在异步编程范式下的一些东西。

![image-20220430110019477](/images/2020-11-01-future-tikv.assets/image-20220430110019477.png)

## 升级 Future 的背景

TiKV 里大量使用了异步编程模式，但是在过去这些代码使用的主要由 futures 0.1 这个库，它在官方没有提供 `std::future` 之前几乎是 Rust 里想要实现异步的事实标准库。但是在去年官方正式支持了 async await 关键字，也提供了官方的 `Future` trait 定义，但并没提供太多的周边功能，例如 timer、sink、channel、stream 等功能。因此想顺利地玩转 Rust 异步，依然需要这个半官方库，配合 rust 标准库的升级，futures 也发布到了 0.3，但是因为使用方式发生了巨大的改变，升级到 futures 0.3，尤其在 TiKV 这样的大型项目还是比较困难的。

## Rust Future 现状

- Future 在 Rust 中是惰性的，只有 poll 才会得到进展
- Async 是零开销的
- Rust 不内置 runtime
- 单/多线程的 runtime 都可以，但各有优缺

异步 rust 比同步 rust 更难使用，并且也带来了更多的维护负担，但良好的设计也能获得一流的性能。虽然 rust 现在提供异步支持，但是很多功能由库实现。

使用带来的挑战：

* 编译错误：异步 rust 依赖很多复杂的语言特性，比如 lifetime 和 pin，因此可能会遇到更多的编译错误提示。

* 运行时错误：stack trace 更加复杂。

* 新错误模式：当你在 async context 下调用了阻塞函数或没正确实现 Future trait。

异步具有传染性，这句话我理解为当底层函数提供了异步的接口，这要求整个上层的 runtime 都必须以异步执行，说直白点，你不能直接在创建的普通线程直接 block_on 一个 future，这会让 future 回退到同步模式，你需要使用一个异步的 runtime（可以是由多线程组成 or 仅仅单线程也可）来执行这个 future。一个 Runtime 一般由两部分组成 1.executor 2.Reactor。Rust 的 Future 被设计成通知并运行的形式。因此 reactor 的工作就是负责通知，executor 负责运行，两者的交互通过 `Waker` 类型交集。

```rust
// futures 0.3 or std::future
pub trait Future {
    type Output;
    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

// futures 0.1
pub trait Future {
    type Item;
    type Error;
    fn poll(&mut self) -> Poll<Self::Item, Self::Error>;
}
```

## 大型项目的挑战

升级 future 我这里选择了反金字塔的过程，这样可以将 PR 独立提交，对每个 crate 分别进行升级，grpc-rs 依然使用 0.1 的版本，当其他的升级都完成了，再升级 grpc-rs。futures 提供了 [compat](https://docs.rs/futures/latest/futures/compat/index.html) 以让你可以将 Future 在 0.1 和 0.3 之间转换，这个功能是分离升级的关键。而且各个 crate 通常会使用一些对方的接口（不能循环），因此想象这是一个树状依赖结构关系，你最好从叶子节点开始动手。

总结即为，找到被依赖最少的节点，对你升级的模块进行隔离，对与其交互的部分使用 compat，以此类推知道升级完所有的老旧代码。

## futures 0.3 的变化

首先是最基本的 Future trait 的定义发生了变化，过去的 Error 如今需要你自己为 Output 实现一个 Result 的结构体，好消息是大量本就不需要 Error 的地方可以省略掉了。

### Pin

除了基本的 trait，下一个就要深刻理解 pin。因为只有被 pin 住的对象才能调用 poll or poll_next, balabala.... 但这是个什么呢？

future 是一个可以被 move 的对象，但是考虑如下代码：

```rust
async {
    let mut x = [0; 128];
    let read_into_buf_fut = read_into_buf(&mut x);
    read_into_buf_fut.await;
    println!("{:?}", x);
}
```

当它被移动后实际的内存地址可能发生变化，下次继续执行用的就是错误的指针地址，所以我们需要 pin，它可以把 future 固定在内存中固定的位置，这样在 async 内部就可以安全的创建引用了。Pin 类型其实包裹了一个指针类型，保证了该指针所指向的值不会移动。`Pin<&mut T>`, `Pin<&T>`, `Pin<Box<T>>` 都表示了 T 不会被移动。但还有一些类型移动的时候是保证没问题的，比如 u8，就实现了 Unpin。我们可以方便地使用类似 `Box::pin()` 来创建出实现了 Unpin 的类型。

```rust
// A function which takes a `Future` that implements `Unpin`.
fn execute_unpin_future(x: impl Future<Output = ()> + Unpin) { /* ... */ }
let fut = async { /* ... */ };
execute_unpin_future(fut); // Error: `fut` does not implement `Unpin` trait
// Pinning with `Box`:
let fut = async { /* ... */ };
let fut = Box::pin(fut);
execute_unpin_future(fut); // OK
```

### async/await

有了这两个关键词，我们可以在升级后省很多代码量。比如在 `components/pd_client/src/client.rs` 这个例子：

```rust
// 0.1
Box::new(handler.map_err(Error::Grpc).and_then(move |mut resp| { 
// 0.3
Box::pin(async move {
	let mut resp = handler.await?;
```

过去的 0.1 会倾向使用大量的链式调用 `a_future.map_err().and_then().map_err().and_then()`，这样写人都要傻了，如今你完全可以像写同步代码那样写异步的编程，只要他们在 async block 包裹就可以，其中 await 代表这部分必须等到这个 future 被返回了结果才能继续执行后面的代码。



**后面复习一下这两个关键字：**

async 有两种写法，一种是 `async fn`，一种是 `async` 代码块。不管怎么样，里面的代码都是惰性的，必须用 executor 来驱动它们运行。内部可以有 `.await` 意思就是非得 block 在这里直到推进下去。

**async Lifetimes**

和传统得函数不太相同，`async fn` 接收的引用或者其它非静态生命周期的参数，返回的 future 也被生命周期所限定。

```rust
async fn foo(x: &u8) -> u8 { *x }
// 上下两个等价
fn foo_expanded<'a>(x: &'a u8) -> impl Future<Output = u8> + 'a {
    async move { *x }
}
```

这说明 async 建立的 future 是可能携带有生命周期限制的。意思就是运行 await 的时候都要确认其生命周期还是有效的。

```rust
fn bad() -> impl Future<Output = u8> {
    let x = 5;
    borrow_x(&x) // ERROR: `x` does not live long enough
}

fn good() -> impl Future<Output = u8> {
    async {
        let x = 5;
        borrow_x(&x).await
    }
}
```

**async move**

不过也可以学习闭包，直接接管所使用变量的生命周期。

```rust
/// `async` block:
///
/// Multiple different `async` blocks can access the same local variable
/// so long as they're executed within the variable's scope
async fn blocks() {
    let my_string = "foo".to_string();

    let future_one = async {
        // ...
        println!("{}", my_string);
    };

    let future_two = async {
        // ...
        println!("{}", my_string);
    };

    // Run both futures to completion, printing "foo" twice:
    let ((), ()) = futures::join!(future_one, future_two);
}
/// `async move` block:
///
/// Only one `async move` block can access the same captured variable, since
/// captures are moved into the `Future` generated by the `async move` block.
/// However, this allows the `Future` to outlive the original scope of the
/// variable:
fn move_block() -> impl Future<Output = ()> {
    let my_string = "foo".to_string();
    async move {
        // ...
        println!("{}", my_string);
    }
}
```

**多线程中的 .await**

使用多线程 Future executor 时，Future 可能会在线程之间移动，因此异步主体中使用的任何变量都必须能够在线程之间传播，因为任何 .await 都可能导致切换到新线程。

这就说明使用到外部的变量必须实现 Send trait。而且一定要注意，在 `.await` 之间只用非 future 感知的锁很危险，可能会造成 executor 线程池锁住：考虑一个单线程的 executor，一个任务可能拿到了一个锁，然后通过 .await 归还给了调度器（.await 很容易切换任务），另一个任务尝试拿到锁就 block 了当前线程，而当前线程被 block 整个程序僵在了这里，就会死锁。为了避免这个使用 `futures::lock` 。

## Waker

Task 表示被提交给 executor 的 Future。waker 提供了一个 `wake` 方法以告诉 executor 关联的任务应该被唤醒，当 wake 被调用后，executor 就知道和这个 waker 相关联的任务可以被推进了。Waker 被实现了 clone 所以它可以被复制和存储。

这个[例子](https://rust-lang.github.io/async-book/02_execution/03_wakeups.html)实现用 waker 实现一个简单计时器。在检测到任务没完成就把 cx.waker().clone() 给这个 future 里的共享字段存起来，每次都这样做一遍是因为一个 future 可能在 executor 中在不同的 task 中移动，这样及时地更新 waker 以确保唤醒成功。完成这个实验后 future 还是不能用，因为我们还没有 executor。

## 最后

当然不是所有的代码都能升级，比如有个测试用到了 `spawn.poll_future_notify`，这个在 0.3 没有对应的方法，所以只能把整个测试重写一遍实现类似的功能：

```rust
    #[test]
    fn test_switch_between_sender_and_receiver() {
        let (tx, mut rx) = unbounded::<i32>(4);
        let future = async move { rx.next().await };
        let task = Task {
            future: Arc::new(Mutex::new(Some(future.boxed()))),
        };
        // Receiver has not received any messages, so the future is not be finished
        // in this tick.
        task.tick();
        assert!(task.future.lock().unwrap().is_some());
        // After sender is dropped, the task will be waked and then it tick self
        // again to advance the progress.
        drop(tx);
        assert!(task.future.lock().unwrap().is_none());
    }
    #[derive(Clone)]
    struct Task {
        future: Arc<Mutex<Option<BoxFuture<'static, Option<i32>>>>>,
    }
    impl Task {
        fn tick(&self) {
            let task = Arc::new(self.clone());
            let mut future_slot = self.future.lock().unwrap();
            if let Some(mut future) = future_slot.take() {
                let waker = task::waker_ref(&task);
                let cx = &mut Context::from_waker(&*waker);
                match future.as_mut().poll(cx) {
                    Poll::Pending => {
                        *future_slot = Some(future);
                    }
                    Poll::Ready(None) => {}
                    _ => unimplemented!(),
                }
            }
        }
    }
    impl ArcWake for Task {
        fn wake_by_ref(arc_self: &Arc<Self>) {
            arc_self.tick();
        }
    }
```

