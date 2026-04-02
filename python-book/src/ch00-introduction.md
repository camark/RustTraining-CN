# Python 程序员的 Rust：完整培训指南

面向具有 Python 经验的开发者学习 Rust 的综合指南。本指南涵盖从基础语法到高级模式的所有内容，重点关注从动态类型、垃圾回收语言转移到具有编译时内存安全保证的静态类型系统语言时所需的概念转变。

## 如何使用本书

**自学格式**：首先完成第一部分（第 1-6 章）—— 这些章节与你已经了解的 Python 概念 closely 对应。第二部分（第 7-12 章）介绍 Rust 特有的概念，如所有权和 traits。第三部分（第 13-16 章）涵盖高级主题和迁移。

**节奏建议：**

| 章节 | 主题 | 建议时间 | 检查点 |
|----------|-------|---------------|------------|
| 1–4 | 设置、类型、控制流 | 1 天 | 你可以用 Rust 编写一个 CLI 温度转换器 |
| 5–6 | 数据结构、枚举、模式匹配 | 1–2 天 | 你可以定义一个带数据的枚举并对它进行穷尽 `match` |
| 7 | 所有权和借用 | 1–2 天 | 你可以解释*为什么* `let s2 = s1` 会使 `s1` 失效 |
| 8–9 | 模块、错误处理 | 1 天 | 你可以创建一个多文件项目，用 `?` 传播错误 |
| 10–12 | Traits、泛型、闭包、迭代器 | 1–2 天 | 你可以将列表推导式转换为迭代器链 |
| 13 | 并发 | 1 天 | 你可以用 `Arc<Mutex<T>>` 编写线程安全的计数器 |
| 14 | Unsafe、PyO3、测试 | 1 天 | 你可以通过 PyO3 从 Python 调用 Rust 函数 |
| 15–16 | 迁移、最佳实践 | 按自己的节奏 | 参考材料 —— 在编写真实代码时查阅 |
| 17 | 综合项目 | 2–3 天 | 构建一个完整的 CLI 应用程序，将所有内容整合在一起 |

**如何使用练习：**
- 章节包含带有解决方案的可折叠 `<details>` 块中的实践练习
- **在查看答案之前始终先尝试练习。** 与借用检查器斗争是学习的一部分 —— 编译器的错误信息是你的老师
- 如果卡住超过 15 分钟，展开解决方案，学习它，然后关闭它并从头再试一次
- [Rust Playground](https://play.rust-lang.org/) 让你无需本地安装即可运行代码

**难度指示器：**
- 🟢 **初学者** —— 直接来自 Python 概念的转换
- 🟡 **中级** —— 需要理解所有权或 traits
- 🔴 **高级** —— 生命周期、async 内部原理或 unsafe 代码

**当你遇到困难时：**
- 仔细阅读编译器错误信息 —— Rust 的错误异常有帮助
- 重读相关部分；像所有权（第 7 章）这样的概念通常在第二遍时豁然开朗
- [Rust 标准库文档](https://doc.rust-lang.org/std/) 非常出色 —— 搜索任何类型或方法
- 对于更深的 async 模式，请参阅伴侣 [Async Rust 培训](../async-book/)

---

## 目录

### 第一部分 —— 基础

#### 1. 介绍和动机 🟢
- [Python 开发者为什么选择 Rust](ch01-introduction-and-motivation.md#the-case-for-rust-for-python-developers)
- [Rust 解决的常见 Python 痛点](ch01-introduction-and-motivation.md#common-python-pain-points-that-rust-addresses)
- [何时选择 Rust 而非 Python](ch01-introduction-and-motivation.md#when-to-choose-rust-over-python)

#### 2. 入门 🟢
- [安装和设置](ch02-getting-started.md#installation-and-setup)
- [你的第一个 Rust 程序](ch02-getting-started.md#your-first-rust-program)
- [Cargo vs pip/Poetry](ch02-getting-started.md#cargo-vs-pippoetry)

#### 3. 内置类型和变量 🟢
- [变量和可变性](ch03-built-in-types-and-variables.md#variables-and-mutability)
- [原始类型对比](ch03-built-in-types-and-variables.md#primitive-types-comparison)
- [字符串类型：String vs &str](ch03-built-in-types-and-variables.md#string-types-string-vs-str)

#### 4. 控制流 🟢
- [条件语句](ch04-control-flow.md#conditional-statements)
- [循环和迭代](ch04-control-flow.md#loops-and-iteration)
- [表达式块](ch04-control-flow.md#expression-blocks)
- [函数和类型签名](ch04-control-flow.md#functions-and-type-signatures)

#### 5. 数据结构和集合 🟢
- [元组、数组、切片](ch05-data-structures-and-collections.md#tuples-and-destructuring)
- [结构体 vs 类](ch05-data-structures-and-collections.md#structs-vs-classes)
- [Vec vs list, HashMap vs dict](ch05-data-structures-and-collections.md#vec-vs-list)

#### 6. 枚举和模式匹配 🟡
- [代数数据类型 vs Union 类型](ch06-enums-and-pattern-matching.md#algebraic-data-types-vs-union-types)
- [穷尽模式匹配](ch06-enums-and-pattern-matching.md#exhaustive-pattern-matching)
- [Option 用于 None 安全](ch06-enums-and-pattern-matching.md#option-for-none-safety)

### 第二部分 —— 核心概念

#### 7. 所有权和借用 🟡
- [理解所有权](ch07-ownership-and-borrowing.md#understanding-ownership)
- [移动语义 vs 引用计数](ch07-ownership-and-borrowing.md#move-semantics-vs-reference-counting)
- [借用和生命周期](ch07-ownership-and-borrowing.md#borrowing-and-lifetimes)
- [智能指针](ch07-ownership-and-borrowing.md#smart-pointers)

#### 8. Crate 和模块 🟢
- [Rust 模块 vs Python 包](ch08-crates-and-modules.md#rust-modules-vs-python-packages)
- [Crate vs PyPI 包](ch08-crates-and-modules.md#crates-vs-pypi-packages)

#### 9. 错误处理 🟡
- [异常 vs Result](ch09-error-handling.md#exceptions-vs-result)
- [? 运算符](ch09-error-handling.md#the--operator)
- [使用 thiserror 自定义错误类型](ch09-error-handling.md#custom-error-types-with-thiserror)

#### 10. Traits 和泛型 🟡
- [Traits vs 鸭子类型](ch10-traits-and-generics.md#traits-vs-duck-typing)
- [Protocols (PEP 544) vs Traits](ch10-traits-and-generics.md#protocols-pep-544-vs-traits)
- [泛型约束](ch10-traits-and-generics.md#generic-constraints)

#### 11. From 和 Into Traits 🟡
- [Rust 中的类型转换](ch11-from-and-into-traits.md#type-conversions-in-rust)
- [From、Into、TryFrom](ch11-from-and-into-traits.md#rust-frominto)
- [字符串转换模式](ch11-from-and-into-traits.md#string-conversions)

#### 12. 闭包和迭代器 🟡
- [闭包 vs Lambdas](ch12-closures-and-iterators.md#rust-closures-vs-python-lambdas)
- [迭代器 vs 生成器](ch12-closures-and-iterators.md#iterators-vs-generators)
- [宏：编写代码的代码](ch12-closures-and-iterators.md#why-macros-exist-in-rust)

### 第三部分 —— 高级主题和迁移

#### 13. 并发 🔴
- [无 GIL：真正的并行](ch13-concurrency.md#no-gil-true-parallelism)
- [线程安全：类型系统保证](ch13-concurrency.md#thread-safety-type-system-guarantees)
- [async/await 对比](ch13-concurrency.md#asyncawait-comparison)

#### 14. Unsafe Rust、FFI 和测试 🔴
- [何时以及为什么使用 Unsafe](ch14-unsafe-rust-and-ffi.md#when-and-why-to-use-unsafe)
- [PyO3：用于 Python 的 Rust 扩展](ch14-unsafe-rust-and-ffi.md#pyo3-rust-extensions-for-python)
- [单元测试 vs pytest](ch14-unsafe-rust-and-ffi.md#unit-tests-vs-pytest)

#### 15. 迁移模式 🟡
- [Rust 中的常见 Python 模式](ch15-migration-patterns.md#common-python-patterns-in-rust)
- [Python 开发者基本 Crate](ch08-crates-and-modules.md#essential-crates-for-python-developers)
- [渐进式采用策略](ch15-migration-patterns.md#incremental-adoption-strategy)

#### 16. 最佳实践 🟡
- [面向 Python 开发者的地道 Rust](ch16-best-practices.md#idiomatic-rust-for-python-developers)
- [常见陷阱和解决方案](ch16-best-practices.md#common-pitfalls-and-solutions)
- [Python→Rust Rosetta 石碑](ch16-best-practices.md#rosetta-stone-python-to-rust)
- [学习路径和资源](ch16-best-practices.md#learning-path-and-resources)

---

### 第四部分 —— 综合项目

#### 17. 综合项目：CLI 任务管理器 🔴
- [项目：`rustdo`](ch17-capstone-project.md#the-project-rustdo)
- [数据模型、存储、命令、业务逻辑](ch17-capstone-project.md#step-1-define-the-data-model-ch-3-6-10-11)
- [测试和扩展目标](ch17-capstone-project.md#step-7-tests-ch-14)

***


