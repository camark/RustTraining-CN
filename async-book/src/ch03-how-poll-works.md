# 3. How Poll Works 🟡

> **你将学到什么：**
> - 执行器的轮询循环：poll → pending → wake → poll again
> - 如何从头构建一个最小化执行器
> - 虚假唤醒（spurious wake）规则及其重要性
> - 工具函数：`poll_fn()` 和 `yield_now()`

## 轮询状态机

执行器运行一个循环：轮询一个 future，如果它是 `Pending`，则挂起它直到其 waker 触发，然后再次轮询。这与 OS 线程根本不同，后者由内核处理调度。

```mermaid
stateDiagram-v2
    [*] --> Idle : Future created
    Idle --> Polling : executor calls poll()
    Polling --> Complete : Ready(value)
    Polling --> Waiting : Pending
    Waiting --> Polling : waker.wake() called
    Complete --> [*] : Value returned
```

> **重要：** 在 *Waiting* 状态下，future **必须** 已向 I/O 源注册 waker。没有注册 = 永久挂起。

### 一个最小化执行器

为了揭开执行器的神秘面纱，让我们构建最简单的一个：

```rust
use std::future::Future;
use std::task::{Context, Poll, RawWaker, RawWakerVTable, Waker};
use std::pin::Pin;

/// 最简单的执行器：忙循环轮询直到 Ready
fn block_on<F: Future>(mut future: F) -> F::Output {
    // 将 future pinned 到栈上
    // SAFETY: `future` 在此之后永不会移动 —— 我们只
    // 通过 pinned 引用访问它直到完成。
    let mut future = unsafe { Pin::new_unchecked(&mut future) };

    // 创建一个 no-op waker（只是不断轮询 —— 效率低但简单）
    fn noop_raw_waker() -> RawWaker {
        fn no_op(_: *const ()) {}
        fn clone(_: *const ()) -> RawWaker { noop_raw_waker() }
        let vtable = &RawWakerVTable::new(clone, no_op, no_op, no_op);
        RawWaker::new(std::ptr::null(), vtable)
    }

    // SAFETY: noop_raw_waker() 返回一个有效的 RawWaker，带有正确的 vtable。
    let waker = unsafe { Waker::from_raw(noop_raw_waker()) };
    let mut cx = Context::from_waker(&waker);

    // 忙循环直到 future 完成
    loop {
        match future.as_mut().poll(&mut cx) {
            Poll::Ready(value) => return value,
            Poll::Pending => {
                // 真实的执行器会在这里 park 线程
                // 并等待 waker.wake() —— 我们只是自旋
                std::thread::yield_now();
            }
        }
    }
}

// 用法：
fn main() {
    let result = block_on(async {
        println!("Hello from our mini executor!");
        42
    });
    println!("Got: {result}");
}
```

> **不要在生产环境中使用这个！** 它忙循环，浪费 CPU。真实的执行器
> （tokio、smol）使用 `epoll`/`kqueue`/`io_uring` 睡眠直到 I/O 就绪。
> 但这展示了核心思想：执行器只是一个调用 `poll()` 的循环。

### 唤醒通知

真实的执行器是事件驱动的。当所有 futures 都是 `Pending` 时，执行器睡眠。waker 是一种中断机制：

```rust
// 真实执行器主循环的概念模型：
fn executor_loop(tasks: &mut TaskQueue) {
    loop {
        // 1. 轮询所有已被唤醒的任务
        while let Some(task) = tasks.get_woken_task() {
            match task.poll() {
                Poll::Ready(result) => task.complete(result),
                Poll::Pending => { /* 任务留在队列中，等待唤醒 */ }
            }
        }

        // 2. 睡眠直到有东西唤醒我们（epoll_wait、kevent 等）
        //    这就是 mio/polling 做繁重工作的地方
        tasks.wait_for_events(); // 阻塞直到 I/O 事件或 waker 触发
    }
}
```

### 虚假唤醒（Spurious Wakes）

即使 future 的 I/O 未就绪，也可能被轮询。这称为*虚假唤醒*。Futures 必须正确处理这种情况：

```rust
impl Future for MyFuture {
    type Output = Data;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Data> {
        // ✅ 正确：总是重新检查实际条件
        if let Some(data) = self.try_read_data() {
            Poll::Ready(data)
        } else {
            // 重新注册 waker（它可能已经改变了！）
            self.register_waker(cx.waker());
            Poll::Pending
        }

        // ❌ 错误：假设 poll 意味着数据就绪
        // let data = self.read_data(); // 可能阻塞或 panic
        // Poll::Ready(data)
    }
}
```

**实现 `poll()` 的规则**：
1. **永远不要阻塞** —— 如果未就绪，立即返回 `Pending`
2. **总是重新注册 waker** —— 它在两次轮询之间可能已经改变
3. **处理虚假唤醒** —— 检查实际条件，不要假定要就绪
4. **不要在 `Ready` 之后轮询** —— 行为**未指定**（可能 panic、返回 `Pending` 或重复 `Ready`）。只有 `FusedFuture` 保证完成后的轮询是安全的

<details>
<summary><strong>🏋️ 练习：实现一个 CountdownFuture</strong>（点击展开）</summary>

**挑战**：实现一个 `CountdownFuture`，从 N 倒数到 0，每次被轮询时*打印*当前计数作为副作用。当达到 0 时，它以 `Ready("Liftoff!")` 完成。（注意：一个 `Future` 只产生**一个**最终值 —— 打印是副作用，不是产生的值。对于多个异步值，见第 11 章的 `Stream`。）

*提示*：这不需要真正的 I/O 源 —— 它可以在每次递减后用 `cx.waker().wake_by_ref()` 立即唤醒自己。

<details>
<summary>🔑 答案</summary>

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

struct CountdownFuture {
    count: u32,
}

impl CountdownFuture {
    fn new(start: u32) -> Self {
        CountdownFuture { count: start }
    }
}

impl Future for CountdownFuture {
    type Output = &'static str;

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output> {
        if self.count == 0 {
            Poll::Ready("Liftoff!")
        } else {
            println!("{}...", self.count);
            self.count -= 1;
            // 立即唤醒 —— 我们总是准备好取得进展
            cx.waker().wake_by_ref();
            Poll::Pending
        }
    }
}

// 与我们的小型执行器或 tokio 一起使用：
// let msg = block_on(CountdownFuture::new(5));
// 打印：5... 4... 3... 2... 1...
// msg == "Liftoff!"
```

**关键要点**：即使这个 future 总是准备好进展，它返回 `Pending` 以在步骤之间让出控制权。它立即调用 `wake_by_ref()` 以便执行器立即重新轮询它。这是协作式多任务处理的基础 —— 每个 future 自愿让出。

</details>
</details>

### 实用工具：`poll_fn` 和 `yield_now`

来自标准库和 tokio 的两个工具函数，避免编写完整的 `Future` 实现：

```rust
use std::future::poll_fn;
use std::task::Poll;

// poll_fn：从闭包创建一个一次性 future
let value = poll_fn(|cx| {
    // 用 cx.waker() 做一些事情，返回 Ready 或 Pending
    Poll::Ready(42)
}).await;

// 实际应用：将基于回调的 API 桥接到异步
async fn read_when_ready(source: &MySource) -> Data {
    poll_fn(|cx| source.poll_read(cx)).await
}
```

```rust
// yield_now：自愿向执行器让出控制权
// 在 CPU 重型异步循环中很有用，避免饿死其他任务
async fn cpu_heavy_work(items: &[Item]) {
    for (i, item) in items.iter().enumerate() {
        process(item); // CPU 工作

        // 每 100 个元素，让出以让其他任务运行
        if i % 100 == 0 {
            tokio::task::yield_now().await;
        }
    }
}
```

> **何时使用 `yield_now()`**：如果你的异步函数在循环中做 CPU 工作
> 而没有任何 `.await` 点，它会独占执行器线程。定期插入
> `yield_now().await` 以实现协作式多任务处理。

> **关键要点 —— How Poll Works**
> - 执行器重复调用已被唤醒的 futures 的 `poll()`
> - Futures 必须处理**虚假唤醒** —— 总是重新检查实际条件
> - `poll_fn()` 让你可以从闭包创建临时 futures
> - `yield_now()` 是 CPU 重型异步代码的协作式调度出口

> **另见：**[第 2 章 — The Future Trait](ch02-the-future-trait.md) 了解 trait 定义，[第 5 章 — The State Machine Reveal](ch05-the-state-machine-reveal.md) 了解编译器生成的代码

***
