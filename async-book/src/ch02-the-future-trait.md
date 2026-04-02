# 2. The Future Trait 🟡

> **你将学到什么：**
> - `Future` trait：`Output`、`poll()`、`Context`、`Waker`
> - waker 如何告诉执行器"再次轮询我"
> - 契约：从不调用 `wake()` = 你的程序静默挂起
> - 手写实现一个真正的 future（`Delay`）

## Future 的解剖

Rust 中的一切异步最终都实现这个 trait：

```rust
pub trait Future {
    type Output;

    fn poll(self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<Self::Output>;
}

pub enum Poll<T> {
    Ready(T),   // Future 已完成，携带值 T
    Pending,    // Future 尚未就绪 —— 稍后再调用我
}
```

就是这样。`Future` 是任何可以被*轮询*（poll）的东西 —— 被问"你完成了吗？" —— 并回答"是的，这是结果"或"还没，我准备好时会唤醒你"。

### Output、poll()、Context、Waker

```mermaid
sequenceDiagram
    participant E as Executor
    participant F as Future
    participant R as Resource (I/O)

    E->>F: poll(cx)
    F->>R: 检查：数据就绪了吗？
    R-->>F: 还没有
    F->>R: 从 cx 注册 waker
    F-->>E: Poll::Pending

    Note over R: ... 时间流逝，数据到达 ...

    R->>E: waker.wake() —— "我准备好了！"
    E->>F: poll(cx) —— 再次尝试
    F->>R: 检查：数据就绪了吗？
    R-->>F: 是的！这是数据
    F-->>E: Poll::Ready(data)
```

让我们分解每个部分：

```rust
use std::future::Future;
use std::pin::Pin;
use std::task::{Context, Poll};

// 一个立即返回 42 的 future
struct Ready42;

impl Future for Ready42 {
    type Output = i32; // Future 最终产生的值

    fn poll(self: Pin<&mut Self>, _cx: &mut Context<'_>) -> Poll<i32> {
        Poll::Ready(42) // 总是就绪 —— 无需等待
    }
}
```

**组件**：
- **`Output`** —— future 完成时产生的值的类型
- **`poll()`** —— 由执行器调用以检查进度；返回 `Ready(value)` 或 `Pending`
- **`Pin<&mut Self>`** —— 确保 future 不会在内存中移动（我们将在第 4 章介绍为什么）
- **`Context`** —— 携带 `Waker`，以便 future 可以在准备好取得进展时向执行器发出信号

### Waker 契约

`Waker` 是回调机制。当 future 返回 `Pending` 时，它*必须*安排稍后调用 `waker.wake()` —— 否则执行器永远不会再次轮询它，程序就会挂起。

```rust
use std::task::{Context, Poll, Waker};
use std::pin::Pin;
use std::future::Future;
use std::sync::{Arc, Mutex};
use std::thread;
use std::time::Duration;

/// 一个在延迟后完成的 future（玩具实现）
struct Delay {
    completed: Arc<Mutex<bool>>,
    waker_stored: Arc<Mutex<Option<Waker>>>,
    duration: Duration,
    started: bool,
}

impl Delay {
    fn new(duration: Duration) -> Self {
        Delay {
            completed: Arc::new(Mutex::new(false)),
            waker_stored: Arc::new(Mutex::new(None)),
            duration,
            started: false,
        }
    }
}

impl Future for Delay {
    type Output = ();

    fn poll(mut self: Pin<&mut Self>, cx: &mut Context<'_>) -> Poll<()> {
        // 检查是否已完成
        if *self.completed.lock().unwrap() {
            return Poll::Ready(());
        }

        // 存储 waker，以便后台线程可以唤醒我们
        *self.waker_stored.lock().unwrap() = Some(cx.waker().clone());

        // 在第一次轮询时启动后台计时器
        if !self.started {
            self.started = true;
            let completed = Arc::clone(&self.completed);
            let waker = Arc::clone(&self.waker_stored);
            let duration = self.duration;

            thread::spawn(move || {
                thread::sleep(duration);
                *completed.lock().unwrap() = true;

                // 关键：唤醒执行器，以便它再次轮询我们
                if let Some(w) = waker.lock().unwrap().take() {
                    w.wake(); // "嘿，执行器，我准备好了 —— 再次轮询我！"
                }
            });
        }

        Poll::Pending // 还没完成
    }
}
```

> **关键见解**：在 C# 中，TaskScheduler 自动处理唤醒。
> 在 Rust 中，**你**（或你使用的 I/O 库）负责调用 `waker.wake()`。
> 忘记它，你的程序就会静默挂起。

### 练习：实现一个 CountdownFuture

<details>
<summary>🏋️ 练习（点击展开）</summary>

**挑战**：实现一个 `CountdownFuture`，从 N 倒数到 0，每次被轮询时打印当前计数。当达到 0 时，它以 `Ready("Liftoff!")` 完成。

*提示*：future 需要存储当前计数并在每次轮询时递减。记得总是重新注册 waker！

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
            println!("Liftoff!");
            Poll::Ready("Liftoff!")
        } else {
            println!("{}...", self.count);
            self.count -= 1;
            cx.waker().wake_by_ref(); // 立即安排重新轮询
            Poll::Pending
        }
    }
}
```

**关键要点**：这个 future 每次计数被轮询一次。每次返回 `Pending` 时，它立即唤醒自己以再次被轮询。在生产环境中，你会使用计时器而不是繁忙轮询。

</details>
</details>

> **关键要点 —— The Future Trait**
> - `Future::poll()` 返回 `Poll::Ready(value)` 或 `Poll::Pending`
> - future 必须在返回 `Pending` 之前注册 `Waker` —— 执行器使用它知道何时重新轮询
> - `Pin<&mut Self>` 保证 future 不会在内存中移动（自引用状态机所需 —— 见第 4 章）
> - Rust 异步中的一切 —— `async fn`、`.await`、组合子 —— 都建立在这个 trait 之上

> **另见：**[第 3 章 — How Poll Works](ch03-how-poll-works.md) 了解执行器循环，[第 6 章 — Building Futures by Hand](ch06-building-futures-by-hand.md) 了解更复杂的实现

***
