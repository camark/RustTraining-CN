# Rust 模式与工程实践指南

## 演讲者介绍

- 微软 SCHIE（芯片和云硬件基础设施工程）团队首席固件架构师
- 行业专家，精通安全、系统编程（固件、操作系统、虚拟机监控器）、CPU 和平台架构以及 C++ 系统
- 2017 年开始使用 Rust 编程（@AWS EC2），从此爱上这门语言

---

这是一本关于真实代码库中出现的中级及以上 Rust 模式的实践指南。这不是语言教程 —— 它假设你已经能编写基本的 Rust 代码并希望提升水平。每章隔离一个概念，解释何时以及为何使用它，并提供可编译的示例和内联练习。

## 目标读者

- 完成了《The Rust Programming Language》但仍在纠结"我该如何设计？"的开发者
- 将生产系统从 C++/C# 翻译到 Rust 的工程师
- 在泛型、trait 约束或生命周期错误中遇到障碍并希望获得系统化工具包的任何人

## 前置要求

开始之前，你应该熟悉：
- 所有权、借用和生命周期（基础级别）
- 枚举、模式匹配和 `Option`/`Result`
- 结构体、方法和基础 traits（`Display`、`Debug`、`Clone`）
- Cargo 基础：`cargo build`、`cargo test`、`cargo run`

## 如何使用本书

### 难度图例

每章都标有难度级别：

| 符号 | 级别 | 含义 |
|------|------|------|
| 🟢 | 基础 | 每个 Rust 开发者都需要掌握的核心概念 |
| 🟡 | 中级 | 生产代码库中使用的模式 |
| 🔴 | 高级 | 深度语言机制 —— 根据需要 revisited |

### 进度指南

| 章节 | 主题 | 建议时间 | 检查点 |
|------|------|----------|--------|
| **第一部分：类型级模式** | | | |
| 1. 泛型 🟢 | 单态化、const 泛型、`const fn` | 1-2 小时 | 能解释何时 `dyn Trait` 优于泛型 |
| 2. Traits 🟡 | 关联类型、GATs、覆盖实现、vtables | 3-4 小时 | 能设计带有抽象类型的 trait |
| 3. Newtype 和类型状态 🟡 | 零成本安全、编译时 FSMs | 2-3 小时 | 能构建类型状态的 builder 模式 |
| 4. PhantomData 🔴 | 生命周期标记、variance、drop check | 2-3 小时 | 能解释为何 `PhantomData<fn(T)>` 不同于 `PhantomData<T>` |
| **第二部分：并发与运行时** | | | |
| 5. Channels 🟢 | `mpsc`、crossbeam、`select!`、actors | 1-2 小时 | 能实现基于 channel 的工作池 |
| 6. 并发 🟡 | 线程、rayon、Mutex、RwLock、atomics | 2-3 小时 | 能为场景选择正确的同步原语 |
| 7. 闭包 🟢 | `Fn`/`FnMut`/`FnOnce`、组合子 | 1-2 小时 | 能编写接受闭包的高阶函数 |
| 8. 函数式 vs 命令式 🟡 | 组合子、迭代器适配器、函数式模式 | 2-3 小时 | 能解释何时函数式风格优于命令式 |
| 9. 智能指针 🟡 | Box、Rc、Arc、Cow、Pin | 2-3 小时 | 能解释何时使用每个智能指针 |
| **第三部分：系统与生产** | | | |
| 10. 错误处理 🟢 | thiserror、anyhow、`?` 操作符 | 1-2 小时 | 能设计错误类型层次结构 |
| 11. 序列化 🟡 | serde、零拷贝、二进制数据 | 2-3 小时 | 能编写自定义 serde 反序列化器 |
| 12. Unsafe 🔴 | 超能力、FFI、UB 陷阱、分配器 | 2-3 小时 | 能用安全的 API 封装不安全代码 |
| 13. 宏 🟡 | `macro_rules!`、过程宏、`syn`/`quote` | 2-3 小时 | 能编写带有 `tt` 递归的声明式宏 |
| 14. 测试 🟢 | 单元/集成/doc 测试、proptest、criterion | 1-2 小时 | 能设置基于属性的测试 |
| 15. API 设计 🟡 | 模块布局、直观的 API、feature 标志 | 2-3 小时 | 能应用"解析，不要验证"模式 |
| 16. Async 🔴 | Futures、Tokio、常见陷阱 | 1-2 小时 | 能识别异步反模式 |
| **附录** | | | |
| 参考卡片 | 快速查看 trait 约束、生命周期、模式 | 根据需要 | — |
| 综合项目 | 类型安全的任务调度器 | 4-6 小时 | 提交可工作的实现 |

**总估计时间**：30-45 小时（包含练习的深入学习）。

### 完成练习

每章都以动手练习结束。为了最大化学习效果：

1. **先自己尝试** —— 在查看答案前至少花 15 分钟
2. **亲手输入代码** —— 不要复制粘贴；输入能建立肌肉记忆
3. **修改解决方案** —— 添加功能、改变约束、故意弄坏某些东西
4. **检查交叉引用** —— 大多数练习结合了来自多章的模式

附录中的综合项目将书中各处的模式整合到一个单一的生产质量系统中。

## 目录

### 第一部分：类型级模式

**[1. 泛型 —— 完整视图](ch01-generics-the-full-picture.md)** 🟢
单态化、代码膨胀权衡、泛型 vs 枚举 vs trait 对象、const 泛型、`const fn`。

**[2. Traits 深入探讨](ch02-traits-in-depth.md)** 🟡
关联类型、GATs、覆盖实现、marker traits、vtables、HRTBs、扩展 traits、枚举分发。

**[3. Newtype 和类型状态模式](ch03-the-newtype-and-type-state-patterns.md)** 🟡
零成本类型安全、编译时状态机、builder 模式、配置 traits。

**[4. PhantomData —— 不携带数据的类型](ch04-phantomdata-types-that-carry-no-data.md)** 🔴
生命周期标记、度量单位模式、drop check、variance。

### 第二部分：并发与运行时

**[5. Channels 和消息传递](ch05-channels-and-message-passing.md)** 🟢
`std::sync::mpsc`、crossbeam、`select!`、背压、actor 模式。

**[6. 并发 vs 并行 vs 线程](ch06-concurrency-vs-parallelism-vs-threads.md)** 🟡
OS 线程、scopped 线程、rayon、Mutex/RwLock/Atomics、Condvar、OnceLock、无锁模式。

**[7. 闭包和高阶函数](ch07-closures-and-higher-order-functions.md)** 🟢
`Fn`/`FnMut`/`FnOnce`、闭包作为参数/返回值、组合子、高阶 APIs。

**[8. 函数式 vs 命令式：何时优雅获胜（以及何时不会）](ch08-functional-vs-imperative-when-elegance-wins.md)** 🟡
组合子、迭代器适配器、函数式模式。

**[9. 智能指针和内部可变性](ch09-smart-pointers-and-interior-mutability.md)** 🟡
Box、Rc、Arc、Weak、Cell/RefCell、Cow、Pin、ManuallyDrop。

### 第三部分：系统与生产

**[10. 错误处理模式](ch10-error-handling-patterns.md)** 🟢
thiserror vs anyhow、`#[from]`、`.context()`、`?` 操作符、panics。

**[11. 序列化、零拷贝和二进制数据](ch11-serialization-zero-copy-and-binary-data.md)** 🟡
serde 基础、枚举表示、零拷贝反序列化、`repr(C)`、`bytes::Bytes`。

**[12. Unsafe Rust —— 受控的危险](ch12-unsafe-rust-controlled-danger.md)** 🔴
五种超能力、可靠的抽象、FFI、UB 陷阱、arena/slab 分配器。

**[13. 宏 —— 编写代码的代码](ch13-macros-code-that-writes-code.md)** 🟡
`macro_rules!`、何时（不）使用宏、过程宏、derive 宏、`syn`/`quote`。

**[14. 测试和基准测试模式](ch14-testing-and-benchmarking-patterns.md)** 🟢
单元/集成/doc 测试、proptest、criterion、mock 策略。

**[15. Crate 架构和 API 设计](ch15-crate-architecture-and-api-design.md)** 🟡
模块布局、API 设计清单、直观的参数、feature 标志、workspaces。

**[16. Async/Await 要点](ch16-asyncawait-essentials.md)** 🔴
Futures、Tokio 快速入门、常见陷阱。（深入异步覆盖，参见我们的 Async Rust 培训。）

### 附录

**[总结和参考卡片](ch18-summary-and-reference-card.md)**
模式决策指南、trait 约束速查表、生命周期省略规则、进一步阅读。

**[综合项目：类型安全的任务调度器](ch19-capstone-project.md)**
将泛型、traits、类型状态、channels、错误处理和测试整合到一个完整的系统中。

***
