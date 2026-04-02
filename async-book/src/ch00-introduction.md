# 异步 Rust：从 Future 到生产级应用

## 讲师介绍

- 微软 SCHIE（硅与云硬件基础设施工程）团队首席固件架构师
- 业界资深专家，专长于安全、系统编程（固件、操作系统、虚拟化）、CPU 与平台架构，以及 C++ 系统
- 2017 年开始使用 Rust 编程（@AWS EC2），从此深深爱上这门语言

---

这是一本深入讲解 Rust 异步编程的指南。与大多数从 `tokio::main` 开始、对内部原理轻描淡写的异步教程不同，本指南从第一性原理构建理解 —— `Future` trait、轮询（polling）、状态机 —— 然后逐步深入到实际模式、运行时选择和生产环境中的陷阱。

## 目标读者

- 能够编写同步 Rust 但觉得异步很令人困惑的 Rust 开发者
- 了解 `async/await` 但不熟悉 Rust 模型的 C#、Go、Python 或 JavaScript 开发者
- 曾被 `Future is not Send`、`Pin<Box<dyn Future>>` 或"为什么我的程序卡住了？"等问题困扰过的任何人

## 前置要求

你应该熟悉以下内容：

- 所有权（ownership）、借用（borrowing）和生命周期（lifetimes）
- Trait 和泛型（包括 `impl Trait`）
- 使用 `Result<T, E>` 和 `?` 运算符
- 基础多线程（`std::thread::spawn`、`Arc`、`Mutex`）

无需任何异步 Rust 经验。

## 如何使用本书

**首次阅读时请按顺序阅读。** 第一部分到第三部分层层递进。每章包含：

| 符号 | 含义 |
|--------|---------|
| 🟢 | 入门 — 基础概念 |
| 🟡 | 中级 — 需要前置章节知识 |
| 🔴 | 高级 — 深度内部原理或生产模式 |

每章还包括：

- 顶部的 **"你将学到什么"** 模块
- **Mermaid 图表**，适合视觉学习者
- **内嵌练习**，附带隐藏答案
- **关键要点**，总结核心思想
- **交叉引用**，链接到相关章节

## 进度指南

| 章节 | 主题 | 建议时间 | 检查点 |
|----------|-------|----------------|------------|
| 1–5 | 异步如何工作 | 6–8 小时 | 你能解释 `Future`、`Poll`、`Pin`，以及为什么 Rust 没有内置运行时 |
| 6–10 | 生态系统 | 6–8 小时 | 你能手写 future、选择运行时，并使用 tokio 的 API |
| 11–13 | 生产级异步 | 6–8 小时 | 你能编写生产级异步代码，包含 streams、正确的错误处理和优雅关闭 |
| 综合项目 | 聊天服务器 | 4–6 小时 | 你已构建了一个整合所有概念的异步应用 |

**总估计时间：22–30 小时**

## 完成练习

每章内容都包含一个内嵌练习。第 16 章的综合项目将所有内容整合为一个完整的项目。为了获得最佳学习效果：

1. **在查看答案之前先尝试练习** — 挣扎是学习发生的地方
2. **亲手输入代码，不要复制粘贴** — 肌肉记忆对 Rust 语法很重要
3. **运行每个示例** — `cargo new async-exercises` 并边学边测试

## 目录

### 第一部分：异步如何工作

- [1. 为什么异步在 Rust 中与众不同](ch01-why-async-is-different-in-rust.md) 🟢 — 根本区别：Rust 没有内置运行时
- [2. The Future Trait](ch02-the-future-trait.md) 🟡 — `poll()`、`Waker`，以及使这一切工作的契约
- [3. How Poll Works](ch03-how-poll-works.md) 🟡 — 轮询状态机和最小化执行器
- [4. Pin and Unpin](ch04-pin-and-unpin.md) 🔴 — 为什么自引用结构体需要 pinning
- [5. The State Machine Reveal](ch05-the-state-machine-reveal.md) 🟢 — 编译器从 `async fn` 实际生成的代码

### 第二部分：生态系统

- [6. Building Futures by Hand](ch06-building-futures-by-hand.md) 🟡 — 从头编写 TimerFuture、Join、Select
- [7. Executors and Runtimes](ch07-executors-and-runtimes.md) 🟡 — tokio、smol、async-std、embassy — 如何选择
- [8. Tokio Deep Dive](ch08-tokio-deep-dive.md) 🟡 — 运行时类型、spawn、channel、同步原语
- [9. When Tokio Isn't the Right Fit](ch09-when-tokio-isnt-the-right-fit.md) 🟡 — LocalSet、FuturesUnordered、运行时无关设计
- [10. Async Traits](ch10-async-traits.md) 🟡 — RPITIT、dyn dispatch、trait_variant、async closures

### 第三部分：生产级异步

- [11. Streams and AsyncIterator](ch11-streams-and-asynciterator.md) 🟡 — 异步迭代、AsyncRead/Write、stream 组合子
- [12. Common Pitfalls](ch12-common-pitfalls.md) 🔴 — 9 个生产环境 bug 及如何避免
- [13. Production Patterns](ch13-production-patterns.md) 🔴 — 优雅关闭、背压（backpressure）、Tower 中间件
- [14. Async Is an Optimization, Not an Architecture](ch14-async-is-an-optimization-not-an-architecture.md) 🔴 — 同步核心/异步外壳、函数着色税

### 附录

- [Summary and Reference Card](ch16-summary-and-reference-card.md) — 快速查找表和决策树
- [Capstone Project: Async Chat Server](ch17-capstone-project.md) — 构建一个完整的异步应用

***
