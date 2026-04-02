<div style="background-color: #d9d9d9; padding: 16px; border-radius: 6px; color: #000000;">

**许可证** 本项目采用 [MIT 许可证](LICENSE) 和 [Creative Commons Attribution 4.0 International (CC-BY-4.0)](LICENSE-DOCS) 双重许可。

</div>

<div style="background-color: #d9d9d9; padding: 16px; border-radius: 6px; color: #000000;">

**商标** 本项目可能包含项目、产品或服务的商标或标志。Microsoft 商标或标志的授权使用必须遵循 [Microsoft 商标和品牌指南](https://www.microsoft.com/en-us/legal/intellectualproperty/trademarks/usage/general)。在本项目的修改版本中使用 Microsoft 商标或标志不得造成混淆或暗示 Microsoft 赞助。任何第三方商标或标志的使用均须遵守这些第三方的政策。

</div>

# Rust 培训丛书

七套培训课程，涵盖从不同编程背景学习 Rust 的内容，另有异步编程、高级模式和工程实践的深入探讨。

本材料结合了原创内容与 Rust 生态系统中一些最佳资源启发而来的思想和示例。目标是将分散在书籍、博客、会议演讲和视频系列中的知识编织成一个连贯的、具有教学结构的体验。

> **免责声明：** 这些丛书是培训材料，而非权威性参考。虽然我们力求准确，但请务必针对 [官方 Rust 文档](https://doc.rust-lang.org/) 和 [Rust Reference](https://doc.rust-lang.org/reference/) 验证关键细节。

### 灵感来源与致谢

- [**The Rust Programming Language**](https://doc.rust-lang.org/book/) —— 一切构建的基础
- [**Jon Gjengset**](https://www.youtube.com/c/JonGjengset) —— 高级 Rust 内部原理深度直播，《Crust of Rust》系列
- [**withoutboats**](https://without.boats/blog/) —— 异步设计、`Pin` 和 Future 模型
- [**fasterthanlime (Amos)**](https://fasterthanli.me/) —— 从第一性原理出发的系统编程，引人入胜的长篇探索
- [**Mara Bos**](https://marabos.nl/) —— *Rust Atomics and Locks*，并发原语
- [**Aleksey Kladov (matklad)**](https://matklad.github.io/) —— Rust Analyzer 见解、API 设计、错误处理模式
- [**Niko Matsakis**](https://smallcultfollowing.com/babysteps/) —— 语言设计、借用检查器内部原理、Polonius
- [**Rust by Example**](https://doc.rust-lang.org/rust-by-example/) 和 [**Rustonomicon**](https://doc.rust-lang.org/nomicon/) —— 实用模式和 unsafe 深入探讨
- [**This Week in Rust**](https://this-week-in-rust.org/) —— 塑造了许多示例的社区发现
- ……以及 **Rust 社区广大成员** 的博客文章、会议演讲、RFC 和论坛讨论为本材料提供了信息——数量太多无法一一列举，但深表感激

## 📖 开始阅读

选择与你的背景匹配的丛书。书籍按复杂度分组，因此你可以规划学习路径：

| 级别 | 描述 |
|------|------|
| 🟢 **桥梁** | 从另一门语言学习 Rust —— 从这里开始 |
| 🔵 **深入探讨** | 聚焦探索 Rust 的主要子系统 |
| 🟡 **高级** | 面向经验丰富的 Rustacean 的模式和技巧 |
| 🟣 **专家** | 前沿的类型级和正确性技术 |
| 🟤 **实践** | 工程、工具和生产就绪 |

| 丛书 | 级别 | 适合谁 |
|------|------|--------|
| [**Rust for C/C++ Programmers**](c-cpp-book/src/SUMMARY.md) | 🟢 桥梁 | 移动语义、RAII、FFI、嵌入式、no_std |
| [**Rust for C# Programmers**](csharp-book/src/SUMMARY.md) | 🟢 桥梁 | Swift / C# / Java → 所有权和类型系统 |
| [**Rust for Python Programmers**](python-book/src/SUMMARY.md) | 🟢 桥梁 | 动态 → 静态类型，无 GIL 并发 |
| [**Async Rust**](async-book/src/SUMMARY.md) | 🔵 深入探讨 | Tokio、Streams、取消安全性 |
| [**Rust Patterns**](rust-patterns-book/src/SUMMARY.md) | 🟡 高级 | Pin、分配器、无锁结构、unsafe |
| [**Type-Driven Correctness**](type-driven-correctness-book/src/SUMMARY.md) | 🟣 专家 | Type-state、Phantom 类型、能力令牌 |
| [**Rust Engineering Practices**](engineering-book/src/SUMMARY.md) | 🟤 实践 | 构建脚本、跨平台编译、CI/CD、Miri |

每本丛书包含 15-16 章，配有 Mermaid 图表、可编辑的 Rust Playground、练习和全文搜索。

> **提示：** 你可以直接在 GitHub 上阅读 Markdown 源代码，或者在 [GitHub Pages 站点](https://microsoft.github.io/RustTraining/) 上浏览带侧边栏导航和搜索的渲染版本。
>
> **本地服务：** 为了获得最佳阅读体验（章节间的键盘导航、即时搜索、离线访问），克隆仓库并运行：
> ```
> # 如果还没有安装 Rust，通过 rustup 安装：
> # https://rustup.rs/
>
> cargo install mdbook@0.4.52 mdbook-mermaid@0.14.0
> cargo xtask serve          # 构建所有丛书并打开本地服务器
> ```

---

## 🔧 面向维护者

<details>
<summary>本地构建、服务和编辑丛书</summary>

### 前置要求

通过 [**rustup**](https://rustup.rs/) 安装 Rust（如果还没有），然后：

```bash
cargo install mdbook@0.4.52 mdbook-mermaid@0.14.0
```

### 构建和服务

```bash
cargo xtask build               # 将所有丛书构建到 site/（本地预览）
cargo xtask serve               # 构建并在 http://localhost:3000 提供服务
cargo xtask deploy              # 将所有丛书构建到 docs/（用于 GitHub Pages）
cargo xtask clean               # 删除 site/ 和 docs/
```

构建或服务单本丛书：

```bash
cd c-cpp-book && mdbook serve --open    # http://localhost:3000
```

### 部署

站点通过 `.github/workflows/pages.yml` 在推送到 `main` 时自动部署到 GitHub Pages。无需手动步骤。

</details>
