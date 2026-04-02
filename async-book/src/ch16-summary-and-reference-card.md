# Summary and Reference Card

## 快速参考卡片

### 异步思维模型

```text
┌─────────────────────────────────────────────────────┐
│  async fn → 状态机 (enum) → impl Future             │
│  .await   → 轮询内部 future                          │
│  executor → loop { poll(); sleep_until_woken(); }   │
│  waker    → "嘿，执行器，再次轮询我"                  │
│  Pin      → "我保证不在内存中移动"                   │
└─────────────────────────────────────────────────────┘
```

### 常见模式速查表

| 目标 | 使用 |
|------|------|
| 并发运行两个 futures | `tokio::join!(a, b)` |
| 竞争两个 futures | `tokio::select! { ... }` |
| 生成后台任务 | `tokio::spawn(async { ... })` |
| 在异步中运行阻塞代码 | `tokio::task::spawn_blocking(\|\| { ... })` |
| 限制并发 | `Semaphore::new(N)` |
| 收集许多任务结果 | `JoinSet` |
| 跨任务共享状态 | `Arc<Mutex<T>>` 或 channels |
| 优雅关闭 | `watch::channel` + `select!` |
| 一次处理 N 个元素的流 | `.buffer_unordered(N)` |
| 超时 future | `tokio::time::timeout(dur, fut)` |
| 带退避的重试 | 自定义组合子（见第 13 章） |

### Pinning 快速参考

| 情况 | 使用 |
|------|------|
| 在堆上 pin 一个 future | `Box::pin(fut)` |
| 在栈上 pin 一个 future | `tokio::pin!(fut)` |
| Pin 一个 `Unpin` 类型 | `Pin::new(&mut val)` —— 安全，免费 |
| 返回一个 pinned trait 对象 | `-> Pin<Box<dyn Future<Output = T> + Send>>` |

### Channel 选择指南

| Channel | 生产者 | 消费者 | 值 | 何时使用 |
|---------|--------|--------|------|---------|
| `mpsc` | N | 1 | 流 | 工作队列，事件总线 |
| `oneshot` | 1 | 1 | 单个 | 请求/响应，完成通知 |
| `broadcast` | N | N | 所有接收者获得所有 | 扇出通知，关闭信号 |
| `watch` | 1 | N | 仅最新 | 配置更新，健康状态 |

### Mutex 选择指南

| Mutex | 何时使用 |
|-------|---------|
| `std::sync::Mutex` | 锁持有时间短，从不在 `.await` 期间持有 |
| `tokio::sync::Mutex` | 锁必须在 `.await` 期间持有 |
| `parking_lot::Mutex` | 高竞争，没有 `.await`，需要性能 |
| `tokio::sync::RwLock` | 多读少写，锁跨越 `.await` |

### 决策快速参考

```text
需要并发？
├── I/O 绑定 → async/await
├── CPU 绑定 → rayon / std::thread
└── 混合 → 对 CPU 部分使用 spawn_blocking

选择运行时？
├── 服务器应用 → tokio
├── 库 → 运行时无关（futures crate）
├── 嵌入式 → embassy
└── 最小化 → smol

需要并发 futures？
├── 可以是 'static + Send → tokio::spawn
├── 可以是 'static + !Send → LocalSet
├── 不能是 'static → FuturesUnordered
└── 需要跟踪/取消 → JoinSet
```

### 常见错误信息和修复

| 错误 | 原因 | 修复 |
|------|------|------|
| `future is not Send` | 在 `.await` 期间持有 `!Send` 类型 | 限制值的作用域，使其在 `.await` 之前 drop，或使用 `current_thread` 运行时 |
| `borrowed value does not live long enough` 在 spawn 中 | `tokio::spawn` 需要 `'static` | 使用 `Arc`、`clone()` 或 `FuturesUnordered` |
| `the trait Future is not implemented for ()` | 缺少 `.await` | 为异步调用添加 `.await` |
| `cannot borrow as mutable` 在 poll 中 | 自引用借用 | 正确使用 `Pin<&mut Self>`（见第 4 章） |
| 程序静默挂起 | 忘记调用 `waker.wake()` | 确保每个 `Pending` 路径注册并触发 waker |

### 进一步阅读

| 资源 | 为什么 |
|------|--------|
| [Tokio Tutorial](https://tokio.rs/tokio/tutorial) | 官方实践指南 —— 非常适合第一个项目 |
| [Async Book (official)](https://rust-lang.github.io/async-book/) | 在语言层面涵盖 `Future`、`Pin`、`Stream` |
| [Jon Gjengset — Crust of Rust: async/await](https://www.youtube.com/watch?v=ThjvMReOXYM) | 2 小时内部深度探讨，带现场编码 |
| [Alice Ryhl — Actors with Tokio](https://ryhl.io/blog/actors-with-tokio/) | 状态化服务的生产架构模式 |
| [Without Boats — Pin, Unpin, and why Rust needs them](https://without.boats/blog/pin/) | 来自语言设计者的原始动机 |
| [Tokio mini-redis](https://github.com/tokio-rs/mini-redis) | 完整的异步 Rust 项目 —— 学习质量生产代码 |
| [Tower documentation](https://docs.rs/tower) | axum、tonic、hyper 使用的中间件/service 架构 |

***

*异步 Rust 培训指南结束*

***
