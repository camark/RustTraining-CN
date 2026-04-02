# C# 程序员的 Rust：完整培训指南

面向具有 C# 经验的开发者学习 Rust 的综合指南。本指南涵盖从基础语法到高级模式的所有内容，重点关注两种语言之间的概念转变和实际差异。

## 课程概述
- **为什么选择 Rust** —— Rust 为何对 C# 开发者重要：性能、安全性和正确性
- **入门** —— 安装、工具链和你的第一个程序
- **基础构建块** —— 类型、变量、控制流
- **数据结构** —— 数组、元组、结构体、集合
- **模式匹配和枚举** —— 代数数据类型和穷尽匹配
- **所有权和借用** —— Rust 的内存管理模型
- **模块和 crate** —— 代码组织和依赖
- **错误处理** —— 基于 Result 的错误传播
- **Traits 和泛型** —— Rust 的类型系统
- **闭包和迭代器** —— 函数式编程模式
- **并发** —— 带有类型系统保证的无畏并发，async/await 深入探讨
- **Unsafe Rust 和 FFI** —— 何时以及如何超越安全 Rust
- **迁移模式** —— 真实的 C# 到 Rust 模式和渐进式采用
- **最佳实践** —— 面向 C# 开发者的地道 Rust

---

# 自学指南

本材料既适用于讲师指导的课程，也适用于自学。如果你是自学，以下是如何充分利用它的建议。

**节奏建议：**

| 章节 | 主题 | 建议时间 | 检查点 |
|----------|-------|---------------|------------|
| 1–4 | 设置、类型、控制流 | 1 天 | 你可以用 Rust 编写一个 CLI 温度转换器 |
| 5–6 | 数据结构、枚举、模式匹配 | 1–2 天 | 你可以定义一个带数据的枚举并对它进行穷尽 `match` |
| 7 | 所有权和借用 | 1–2 天 | 你可以解释*为什么* `let s2 = s1` 会使 `s1` 失效 |
| 8–9 | 模块、错误处理 | 1 天 | 你可以创建一个多文件项目，用 `?` 传播错误 |
| 10–12 | Traits、泛型、闭包、迭代器 | 1–2 天 | 你可以将 LINQ 链转换为 Rust 迭代器 |
| 13 | 并发和 async | 1 天 | 你可以用 `Arc<Mutex<T>>` 编写线程安全的计数器 |
| 14 | Unsafe Rust、FFI、测试 | 1 天 | 你可以通过 P/Invoke 调用 Rust 函数 |
| 15–16 | 迁移、最佳实践、工具链 | 按自己的节奏 | 参考材料 —— 在编写真实代码时查阅 |
| 17 | 综合项目 | 1–2 天 | 你有一个可以获取天气数据的 CLI 工具 |

**如何使用练习：**
- 章节包含带有解决方案的可折叠 `<details>` 块中的实践练习
- **在查看答案之前始终先尝试练习。** 与借用检查器斗争是学习的一部分 —— 编译器的错误信息是你的老师
- 如果卡住超过 15 分钟，展开解决方案，学习它，然后关闭它并从头再试一次
- [Rust Playground](https://play.rust-lang.org/) 让你无需本地安装即可运行代码

**难度指示器：**
- 🟢 **初学者** —— 直接来自 C# 概念的转换
- 🟡 **中级** —— 需要理解所有权或 traits
- 🔴 **高级** —— 生命周期、async 内部原理或 unsafe 代码

**当你遇到困难时：**
- 仔细阅读编译器错误信息 —— Rust 的错误异常有帮助
- 重读相关部分；像所有权（第 7 章）这样的概念通常在第二遍时豁然开朗
- [Rust 标准库文档](https://doc.rust-lang.org/std/) 非常出色 —— 搜索任何类型或方法
- 对于更深的 async 模式，请参阅伴侣 [Async Rust 培训](../async-book/)

---

# 目录

## 第一部分 —— 基础

### 1. 介绍和动机 🟢
- [C# 开发者为什么选择 Rust](ch01-introduction-and-motivation.md#the-case-for-rust-for-c-developers)
- [Rust 解决的常见 C# 痛点](ch01-introduction-and-motivation.md#common-c-pain-points-that-rust-addresses)
- [何时选择 Rust 而非 C#](ch01-introduction-and-motivation.md#when-to-choose-rust-over-c)
- [语言哲学对比](ch01-introduction-and-motivation.md#language-philosophy-comparison)
- [快速参考：Rust vs C#](ch01-introduction-and-motivation.md#quick-reference-rust-vs-c)

### 2. 入门 🟢
- [安装和设置](ch02-getting-started.md#installation-and-setup)
- [你的第一个 Rust 程序](ch02-getting-started.md#your-first-rust-program)
- [Cargo vs NuGet/MSBuild](ch02-getting-started.md#cargo-vs-nugetmsbuild)
- [读取输入和 CLI 参数](ch02-getting-started.md#reading-input-and-cli-arguments)
- [基本 Rust 关键字 *(可选参考 —— 按需查阅)*](ch02-1-essential-keywords-reference.md#essential-rust-keywords-for-c-developers)

### 3. 内置类型和变量 🟢
- [变量和可变性](ch03-built-in-types-and-variables.md#variables-and-mutability)
- [原始类型对比](ch03-built-in-types-and-variables.md#primitive-types)
- [字符串类型：String vs &str](ch03-built-in-types-and-variables.md#string-types-string-vs-str)
- [打印和字符串格式化](ch03-built-in-types-and-variables.md#printing-and-string-formatting)
- [类型转换](ch03-built-in-types-and-variables.md#type-casting-and-conversions)
- [真正的不可变性 vs Record 幻觉](ch03-1-true-immutability-vs-record-illusions.md#true-immutability-vs-record-illusions)

### 4. 控制流 🟢
- [函数 vs 方法](ch04-control-flow.md#functions-vs-methods)
- [表达式 vs 语句（重要！）](ch04-control-flow.md#expression-vs-statement-important)
- [条件语句](ch04-control-flow.md#conditional-statements)
- [循环和迭代](ch04-control-flow.md#loops)

### 5. 数据结构和集合 🟢
- [元组和解构](ch05-data-structures-and-collections.md#tuples-and-destructuring)
- [数组和切片](ch05-data-structures-and-collections.md#arrays-and-slices)
- [结构体 vs 类](ch05-data-structures-and-collections.md#structs-vs-classes)
- [构造函数模式](ch05-1-constructor-patterns.md#constructor-patterns)
- [`Vec<T>` vs `List<T>`](ch05-2-collections-vec-hashmap-and-iterators.md#vect-vs-listt)
- [HashMap vs Dictionary](ch05-2-collections-vec-hashmap-and-iterators.md#hashmap-vs-dictionary)

### 6. 枚举和模式匹配 🟡
- [代数数据类型 vs C# Unions](ch06-enums-and-pattern-matching.md#algebraic-data-types-vs-c-unions)
- [穷尽模式匹配](ch06-1-exhaustive-matching-and-null-safety.md#exhaustive-pattern-matching-compiler-guarantees-vs-runtime-errors)
- [`Option<T>` 用于空值安全](ch06-1-exhaustive-matching-and-null-safety.md#null-safety-nullablet-vs-optiont)
- [守卫和高级模式](ch06-enums-and-pattern-matching.md#guards-and-advanced-patterns)

### 7. 所有权和借用 🟡
- [理解所有权](ch07-ownership-and-borrowing.md#understanding-ownership)
- [移动语义 vs 引用语义](ch07-ownership-and-borrowing.md#move-semantics)
- [借用和引用](ch07-ownership-and-borrowing.md#borrowing-basics)
- [内存安全深入探讨](ch07-1-memory-safety-deep-dive.md#references-vs-pointers)
- [生命周期深入探讨](ch07-2-lifetimes-deep-dive.md#lifetimes-telling-the-compiler-how-long-references-live) 🔴
- [智能指针、Drop 和 Deref](ch07-3-smart-pointers-beyond-single-ownership.md#smart-pointers-when-single-ownership-isnt-enough) 🔴

### 8. Crate 和模块 🟢
- [Rust 模块 vs C# 命名空间](ch08-crates-and-modules.md#rust-modules-vs-c-namespaces)
- [Crate vs .NET 程序集](ch08-crates-and-modules.md#crates-vs-net-assemblies)
- [包管理：Cargo vs NuGet](ch08-1-package-management-cargo-vs-nuget.md#package-management-cargo-vs-nuget)

### 9. 错误处理 🟡
- [异常 vs `Result<T, E>`](ch09-error-handling.md#exceptions-vs-resultt-e)
- [? 运算符](ch09-error-handling.md#the--operator-propagating-errors-concisely)
- [自定义错误类型](ch06-1-exhaustive-matching-and-null-safety.md#custom-error-types)
- [Crate 级错误类型和 Result 别名](ch09-1-crate-level-error-types-and-result-alias.md#crate-level-error-types-and-result-aliases)
- [错误恢复模式](ch09-1-crate-level-error-types-and-result-alias.md#error-recovery-patterns)

### 10. Traits 和泛型 🟡
- [Traits vs 接口](ch10-traits-and-generics.md#traits---rusts-interfaces)
- [继承 vs 组合](ch10-2-inheritance-vs-composition.md#inheritance-vs-composition)
- [泛型约束：where vs trait bounds](ch10-1-generic-constraints.md#generic-constraints-where-vs-trait-bounds)
- [常见标准库 Traits](ch10-traits-and-generics.md#common-standard-library-traits)

### 11. From 和 Into Traits 🟡
- [Rust 中的类型转换](ch11-from-and-into-traits.md#type-conversions-in-rust)
- [为自定义类型实现 From](ch11-from-and-into-traits.md#rust-from-and-into)

### 12. 闭包和迭代器 🟡
- [Rust 闭包](ch12-closures-and-iterators.md#rust-closures)
- [LINQ vs Rust 迭代器](ch12-closures-and-iterators.md#linq-vs-rust-iterators)
- [宏入门](ch12-1-macros-primer.md#macros-code-that-writes-code)

---

## 第二部分 —— 并发和系统

### 13. 并发 🔴
- [线程安全：约定 vs 类型系统保证](ch13-concurrency.md#thread-safety-convention-vs-type-system-guarantees)
- [async/await：C# Task vs Rust Future](ch13-1-asyncawait-deep-dive.md#async-programming-c-task-vs-rust-future)
- [取消模式](ch13-1-asyncawait-deep-dive.md#cancellation-cancellationtoken-vs-drop--select)
- [Pin 和 tokio::spawn](ch13-1-asyncawait-deep-dive.md#pin-why-rust-async-has-a-concept-c-doesnt)

### 14. Unsafe Rust、FFI 和测试 🟡
- [何时以及为什么使用 Unsafe](ch14-unsafe-rust-and-ffi.md#when-you-need-unsafe)
- [通过 FFI 与 C# 互操作](ch14-unsafe-rust-and-ffi.md#interop-with-c-via-ffi)
- [Rust vs C# 中的测试](ch14-1-testing.md#testing-in-rust-vs-c)
- [属性测试和 Mocking](ch14-1-testing.md#property-testing-proving-correctness-at-scale)

---

## 第三部分 —— 迁移和最佳实践

### 15. 迁移模式和案例研究 🟡
- [Rust 中的常见 C# 模式](ch15-migration-patterns-and-case-studies.md#common-c-patterns-in-rust)
- [C# 开发者基本 Crate](ch15-1-essential-crates-for-c-developers.md#essential-crates-for-c-developers)
- [渐进式采用策略](ch15-2-incremental-adoption-strategy.md#incremental-adoption-strategy)

### 16. 最佳实践和参考 🟡
- [面向 C# 开发者的地道 Rust](ch16-best-practices.md#best-practices-for-c-developers)
- [性能对比：托管 vs 原生](ch16-1-performance-comparison-and-migration.md#performance-comparison-managed-vs-native)
- [常见陷阱和解决方案](ch16-2-learning-path-and-resources.md#common-pitfalls-for-c-developers)
- [学习路径和资源](ch16-2-learning-path-and-resources.md#learning-path-and-next-steps)
- [Rust 工具链生态系统](ch16-3-rust-tooling-ecosystem.md#essential-rust-tooling-for-c-developers)

---

## 综合项目

### 17. 综合项目 🟡
- [构建一个 CLI 天气工具](ch17-capstone-project.md#capstone-project-build-a-cli-weather-tool) —— 将结构体、traits、错误处理、async、模块、serde 和测试整合到一个可运行的应用程序中


