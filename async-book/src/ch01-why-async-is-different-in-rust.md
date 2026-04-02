# 1. 为什么异步在 Rust 中与众不同 🟢

> **你将学到什么：**
> - 为什么 Rust 没有内置的异步运行时（以及这对意味着什么）
> - 三个关键特性：惰性执行、无运行时、零成本抽象
> - 何时异步是正确的选择（何时更慢）
> - Rust 的模型如何与 C#、Go、Python 和 JavaScript 比较

## 根本区别

大多数带有 `async/await` 的语言隐藏了内部机制。C# 有 CLR 线程池。JavaScript 有事件循环。Go 有 goroutines 和内置到运行时的调度器。Python 有 `asyncio`。

**Rust 什么都没有。**

没有内置运行时，没有线程池，没有事件循环。`async` 关键字是一种零成本的编译策略 —— 它将你的函数转换为一个实现 `Future` trait 的状态机。必须由其他人（一个*执行器*）来驱动这个状态机前进。

### Rust 异步的三个关键特性

```mermaid
graph LR
    subgraph "C# / JS / Go"
        EAGER["惰性执行<br/>任务立即启动"]
        BUILTIN["内置运行时<br/>包含线程池"]
        GC["GC 管理<br/>无生命周期问题"]
    end

    subgraph "Rust（以及 Python*）"
        LAZY["惰性执行<br/>直到被轮询/await 才发生"]
        BYOB["自带运行时<br/>你选择执行器"]
        OWNED["所有权适用<br/>生命周期、Send、Sync 很重要"]
    end

    EAGER -. "相反" .-> LAZY
    BUILTIN -. "相反" .-> BYOB
    GC -. "相反" .-> OWNED

    style LAZY fill:#e8f5e8,color:#000
    style BYOB fill:#e8f5e8,color:#000
    style OWNED fill:#e8f5e8,color:#000
    style EAGER fill:#e3f2fd,color:#000
    style BUILTIN fill:#e3f2fd,color:#000
    style GC fill:#e3f2fd,color:#000
```

> * Python 协程像 Rust future 一样是惰性的 —— 它们在被 awaited 或调度之前不会执行。但 Python 仍然使用 GC，没有所有权/生命周期问题。

### 没有内置运行时

```rust
// 这能编译，但什么都不做：
async fn fetch_data() -> String {
    "hello".to_string()
}

fn main() {
    let future = fetch_data(); // 创建 Future，但不执行它
    // future 只是一个放在栈上的结构体
    // 没有输出，没有副作用，什么都没发生
    drop(future); // 静默丢弃 —— 工作从未开始
}
```

与 C# 对比，`Task` 是急切启动的：
```csharp
// C# —— 这立即开始执行：
async Task<string> FetchData() => "hello";

var task = FetchData(); // 已经在运行！
var result = await task; // 只是等待完成
```

### 惰性 Future vs 急切 Task

这是最重要的思维转变：

| | C# / JavaScript | Python | Go | Rust |
|---|---|---|---|---|
| **创建** | `Task` 立即开始执行 | 协程是**惰性的** —— 返回对象，直到被 awaited 或调度才运行 | Goroutine 立即启动 | `Future` 在被轮询之前什么都不做 |
| **丢弃** | 分离的任务继续运行 | 未 awaited 的协程被垃圾回收（带警告） | Goroutine 运行直到返回 | 丢弃 Future 会取消它 |
| **运行时** | 内置于语言/虚拟机 | `asyncio` 事件循环（必须显式启动） | 内置于二进制文件（M:N 调度器） | 你选择（tokio、smol 等） |
| **调度** | 自动（线程池） | 事件循环 + `await` 或 `create_task()` | 自动（GMP 调度器） | 显式（`spawn`、`block_on`） |
| **取消** | `CancellationToken`（协作式） | `Task.cancel()`（协作式，抛出 `CancelledError`） | `context.Context`（协作式） | 丢弃 future（立即） |

```rust
// 要真正运行一个 future，你需要一个执行器：
#[tokio::main]
async fn main() {
    let result = fetch_data().await; // 现在才执行
    println!("{result}");
}
```

### 何时使用异步（何时不使用）

```mermaid
graph TD
    START["什么类型的工作？"]

    IO["I/O 密集型？<br/>（网络、文件、数据库）"]
    CPU["CPU 密集型？<br/>（计算、解析）"]
    MANY["大量并发连接？<br/>（100+）"]
    FEW["少量并发任务？<br/>（<10）"]

    USE_ASYNC["✅ 使用 async/await"]
    USE_THREADS["✅ 使用 std::thread 或 rayon"]
    USE_SPAWN_BLOCKING["✅ 使用 spawn_blocking()"]
    MAYBE_SYNC["考虑同步代码<br/>（更简单，开销更小）"]

    START -->|网络、文件、数据库 | IO
    START -->|计算 | CPU
    IO -->|是，大量 | MANY
    IO -->|仅少量 | FEW
    MANY --> USE_ASYNC
    FEW --> MAYBE_SYNC
    CPU -->|并行化 | USE_THREADS
    CPU -->|在异步上下文中 | USE_SPAWN_BLOCKING

    style USE_ASYNC fill:#c8e6c9,color:#000
    style USE_THREADS fill:#c8e6c9,color:#000
    style USE_SPAWN_BLOCKING fill:#c8e6c9,color:#000
    style MAYBE_SYNC fill:#fff3e0,color:#000
```

**经验法则**：异步用于 I/O 并发（在等待时做多件事），而非 CPU 并行（让一件事更快）。如果你有 10,000 个网络连接，异步表现出色。如果你在 crunch 数字，使用 `rayon` 或 OS 线程。

### 何时异步可能*更慢*

异步不是免费的。对于低并发工作负载，同步代码可能比异步表现更好：

| 成本 | 原因 |
|------|-----|
| **状态机开销** | 每个 `.await` 添加一个 enum 变体；深度嵌套的 futures 产生大型、复杂的状态机 |
| **动态分发** | `Box<dyn Future>` 添加间接性并破坏内联 |
| **上下文切换** | 协作式调度仍有成本 —— 执行器必须管理任务队列、waker 和 I/O 注册 |
| **编译时间** | 异步代码生成更复杂的类型，减慢编译 |
| **可调试性** | 穿过状态机的堆栈跟踪更难阅读（见第 12 章） |

**基准测试指导**：如果少于约 10 个并发 I/O 操作，在提交使用异步之前进行性能分析。在现代 Linux 上，每个连接一个简单的 `std::thread::spawn` 可以很好地扩展到数百个线程。

### 练习：何时使用异步？

<details>
<summary>🏋️ 练习（点击展开）</summary>

对于每个场景，决定是否适合使用异步并解释原因：

1. 一个处理 10,000 个并发 WebSocket 连接的 Web 服务器
2. 一个压缩单个大文件的 CLI 工具
3. 一个查询 5 个不同数据库并合并结果的服务
4. 一个以 60 FPS 运行物理模拟的游戏引擎

<details>
<summary>🔑 答案</summary>

1. **异步** —— I/O 密集型且大量并发。每个连接大部分时间在等待数据。线程需要 10K 栈。
2. **同步/线程** —— CPU 密集型，单一任务。异步增加开销但没有好处。使用 `rayon` 进行并行压缩。
3. **异步** —— 五个并发 I/O 等待。`tokio::join!` 同时运行所有五个查询。
4. **同步/线程** —— CPU 密集型，对延迟敏感。异步的协作式调度可能引入帧抖动。

</details>
</details>

> **关键要点 —— 为什么异步不同**
> - Rust future 是**惰性的** —— 它们在执行器轮询之前什么都不做
> - **没有内置运行时** —— 你选择（或构建）自己的
> - 异步是**零成本编译策略**，生成状态机
> - 异步在**I/O 绑定并发**中表现出色；对于 CPU 绑定工作，使用线程或 rayon

> **另见：**[第 2 章 — The Future Trait](ch02-the-future-trait.md) 了解使这一切工作的 trait，[第 7 章 — Executors and Runtimes](ch07-executors-and-runtimes.md) 了解如何选择运行时

***
