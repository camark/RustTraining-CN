# Rust 工程实践 —— 超越 `cargo build`

## 讲师介绍

- 微软 SCHIE（硅和云硬件基础设施工程）团队首席固件架构师
- 行业资深专家，专长于安全、系统编程（固件、操作系统、虚拟化器）、CPU 和平台架构，以及 C++ 系统
- 2017 年开始使用 Rust 编程（@AWS EC2），从此爱上这门语言

---

> 一本关于 Rust 工具链功能的实用指南，大多数团队发现得太晚：
> 构建脚本、跨平台编译、基准测试、代码覆盖率，以及使用 Miri 和 Valgrind 进行安全验证。
> 每章使用来自真实硬件诊断代码库的具体示例 ——
> 一个大型多 crate 工作空间 —— 所以每种技术直接映射到生产代码。

## 如何使用本书

本书设计用于**自定进度学习或团队研讨会**。每章基本独立 —— 按顺序阅读或跳转到你需要的话题。

### 难度图例

| 符号 | 级别 | 含义 |
|:------:|-------|---------|
| 🟢 | 入门 | 直接了当的工具，清晰的模式 —— 第一天就有用 |
| 🟡 | 中级 | 需要了解工具链内部或平台概念 |
| 🔴 | 高级 | 深度工具链知识、nightly 功能或多工具编排 |

### 进度指南

| 部分 | 章节 | 预计时间 | 关键成果 |
|------|----------|:---------:|-------------|
| **I — 构建和发布** | ch01–02 | 3–4 小时 | 构建元数据、跨平台编译、静态二进制文件 |
| **II — 测量和验证** | ch03–05 | 4–5 小时 | 统计基准测试、覆盖率门禁、Miri/消毒器 |
| **III — 强化和优化** | ch06–10 | 6–8 小时 | 供应链安全、发布配置文件、编译时工具、`no_std`、Windows |
| **IV — 集成** | ch11–13 | 3–4 小时 | 生产 CI/CD 管道、技巧、综合练习 |
| | | **16–21 小时** | **完整的生产工程管道** |

### 完成练习

每章包含带难度指示器的**🏋️ 练习**。解决方案在可展开的 `<details>` 块中 —— 先尝试练习，然后检查你的工作。

- 🟢 练习可以在 10–15 分钟内完成
- 🟡 练习需要 20–40 分钟，可能涉及在本地运行工具
- 🔴 练习需要大量设置和实验（1+ 小时）

## 前置要求

| 概念 | 在哪里学习 |
|---------|-------------------|
| Cargo 工作空间布局 | [Rust Book ch14.3](https://doc.rust-lang.org/book/ch14-03-cargo-workspaces.html) |
| 功能标志 | [Cargo Reference — Features](https://doc.rust-lang.org/cargo/reference/features.html) |
| `#[cfg(test)]` 和基础测试 | Rust Patterns ch12 |
| `unsafe` 块和 FFI 基础 | Rust Patterns ch10 |

## 章节依赖图

```text
                 ┌──────────┐
                 │ ch00     │
                 │  简介    │
                 └────┬─────┘
        ┌─────┬───┬──┴──┬──────┬──────┐
        ▼     ▼   ▼     ▼      ▼      ▼
      ch01  ch03 ch04  ch05   ch06   ch09
      构建  基准 覆盖率 Miri   依赖   no_std
        │     └────┴────┘      │      │
        │          │           │      ▼
        │          │           │    ch10
        │          │           │   Windows
        ▼          ▼           ▼     │
       ch02      ch07        ch07    │
       跨平台    发布配置     发布配置 │
        │          │           │     │
        │          ▼           │     │
        │        ch08          │     │
        │      编译时          │     │
        └──────────┴───────────┴─────┘
                   │
                   ▼
                 ch11
               CI/CD 管道
                   │
                   ▼
                ch12 ─── ch13
               技巧      快速参考
```

**按任意顺序阅读**：ch01、ch03、ch04、ch05、ch06、ch09 是独立的。
**在前置要求后阅读**：ch02（需要 ch01）、ch07–ch08（从 ch03–ch06 受益）、ch10（从 ch09 受益）。
**最后阅读**：ch11（整合所有内容）、ch12（技巧）、ch13（参考）。

## 带注释的目录

### 第一部分 —— 构建和发布

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 1 | [构建脚本 —— `build.rs` 深入探讨](ch01-build-scripts-buildrs-in-depth.md) | 🟢 | 编译时常量、编译 C 代码、protobuf 生成、系统库链接、反模式 |
| 2 | [跨平台编译 —— 一个源码，多个目标](ch02-cross-compilation-one-source-many-target.md) | 🟡 | 目标三元组、musl 静态二进制文件、ARM 跨平台编译、`cross` 工具、`cargo-zigbuild`、GitHub Actions |

### 第二部分 —— 测量和验证

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 3 | [基准测试 —— 测量重要的内容](ch03-benchmarking-measuring-what-matters.md) | 🟡 | Criterion.rs、Divan、`perf` 火焰图、PGO、CI 中的持续基准测试 |
| 4 | [代码覆盖率 —— 看到测试遗漏的内容](ch04-code-coverage-seeing-what-tests-miss.md) | 🟢 | `cargo-llvm-cov`、`cargo-tarpaulin`、`grcov`、Codecov/Coveralls CI 集成 |
| 5 | [Miri、Valgrind 和消毒器](ch05-miri-valgrind-and-sanitizers-verifying-u.md) | 🔴 | MIR 解释器、Valgrind memcheck/Helgrind、ASan/MSan/TSan、cargo-fuzz、loom |

### 第三部分 —— 强化和优化

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 6 | [依赖管理和供应链安全](ch06-dependency-management-and-supply-chain-s.md) | 🟢 | `cargo-audit`、`cargo-deny`、`cargo-vet`、`cargo-outdated`、`cargo-semver-checks` |
| 7 | [发布配置文件和二进制文件大小](ch07-release-profiles-and-binary-size.md) | 🟡 | 发布配置文件解剖、LTO 权衡、`cargo-bloat`、`cargo-udeps` |
| 8 | [编译时和开发者工具](ch08-compile-time-and-developer-tools.md) | 🟡 | `sccache`、`mold`、`cargo-nextest`、`cargo-expand`、`cargo-geiger`、工作空间 lint、MSRV |
| 9 | [`no_std` 和功能验证](ch09-no-std-and-feature-verification.md) | 🔴 | `cargo-hack`、`core`/`alloc`/`std` 层、自定义 panic 处理程序、测试 `no_std` 代码 |
| 10 | [Windows 和条件编译](ch10-windows-and-conditional-compilation.md) | 🟡 | `#[cfg]` 模式、`windows-sys`/`windows` crate、`cargo-xwin`、平台抽象 |

### 第四部分 —— 集成

| # | 章节 | 难度 | 描述 |
|---|---------|:----------:|-------------|
| 11 | [整合所有内容 —— 生产 CI/CD 管道](ch11-putting-it-all-together-a-production-cic.md) | 🟡 | GitHub Actions 工作流、`cargo-make`、pre-commit 钩子、`cargo-dist`、综合项目 |
| 12 | [来自实践的技巧](ch12-tricks-from-the-trenches.md) | 🟡 | 10 个经过实战验证的模式：`deny(warnings)` 陷阱、缓存调优、依赖去重、RUSTFLAGS 等 |
| 13 | [快速参考卡片](ch13-quick-reference-card.md) | — | 快速浏览命令、60+ 决策表条目、进一步阅读链接 |
