# 构建脚本 —— `build.rs` 深入探讨 🟢

> **你将学到什么：**
> - `build.rs` 如何融入 Cargo 构建管道以及何时运行
> - 五种生产模式：编译时常量、C/C++ 编译、protobuf 代码生成、`pkg-config` 链接和功能检测
> - 减慢构建或破坏跨平台编译的反模式
> - 如何在可追溯性和可重现构建之间取得平衡
>
> **交叉引用：** [跨平台编译](ch02-cross-compilation-one-source-many-target.md) 使用构建脚本进行目标感知构建 · [`no_std` 和功能](ch09-no-std-and-feature-verification.md) 扩展此处设置的 `cfg` 标志 · [CI/CD 管道](ch11-putting-it-all-together-a-production-cic.md) 在自动化中编排构建脚本

每个 Cargo 包都可以在 crate 根目录包含一个名为 `build.rs` 的文件。
Cargo 在编译 crate *之前* 编译并执行此文件。构建脚本通过 stdout 上的 `println!` 指令与 Cargo 通信。

### build.rs 是什么以及何时运行

```text
┌─────────────────────────────────────────────────────────┐
│                    Cargo 构建管道                        │
│                                                         │
│  1. 解析依赖                                            │
│  2. 下载 crate                                          │
│  3. 编译 build.rs  ← 普通 Rust，在 HOST 上运行            │
│  4. 执行 build.rs  ← stdout → Cargo 指令                  │
│  5. 编译 crate（使用步骤 4 的指令）                        │
│  6. 链接                                                 │
└─────────────────────────────────────────────────────────┘
```

关键事实：
- `build.rs` 在**主机**上运行，不在目标机上。在跨平台编译期间，即使最终二进制文件针对不同架构，构建脚本也在你的开发机器上运行。
- 构建脚本的作用域限于其自己的包。它无法影响其他 crate 的编译方式 —— 除非包在 `Cargo.toml` 中声明 `links` 键，这通过 `cargo::metadata=KEY=VALUE` 实现将元数据传递给依赖 crate。
- **每次** Cargo 检测到更改时都会运行 —— 除非你发出 `cargo::rerun-if-changed` 指令来限制重新运行。

> **注意（Rust 1.71+）**：从 Rust 1.71 开始，Cargo 对编译的 `build.rs` 二进制文件进行指纹识别 —— 如果二进制文件相同，即使源时间戳更改也不会重新运行。但是，`cargo::rerun-if-changed=build.rs` 仍然很有价值：没有任何 `rerun-if-changed` 指令，Cargo 会在**包中任何文件**更改时重新运行 `build.rs`（不仅是 `build.rs`）。发出 `cargo::rerun-if-changed=build.rs` 将重新运行限制为仅当 `build.rs` 本身更改时 —— 在大型 crate 中显著节省编译时间。
- 它可以发出 *cfg 标志*、*环境变量*、*链接器参数* 和 *文件路径*，主 crate 使用这些。

最小的 `Cargo.toml` 条目：

```toml
[package]
name = "my-crate"
version = "0.1.0"
edition = "2021"
build = "build.rs"       # 默认 —— Cargo 自动查找 build.rs
# build = "src/build.rs" # 或放在其他位置
```

### Cargo 指令协议

你的构建脚本通过在 stdout 上打印指令与 Cargo 通信。从 Rust 1.77 开始，首选前缀是 `cargo::`（替换旧的 `cargo:` 单冒号形式）。

| 指令 | 用途 |
|-------------|---------|
| `cargo::rerun-if-changed=PATH` | 仅当 PATH 更改时重新运行 build.rs |
| `cargo::rerun-if-env-changed=VAR` | 仅当环境变量 VAR 更改时重新运行 |
| `cargo::rustc-link-lib=NAME` | 链接到原生库 NAME |
| `cargo::rustc-link-search=PATH` | 将 PATH 添加到库搜索路径 |
| `cargo::rustc-cfg=KEY` | 设置 `#[cfg(KEY)]` 标志用于条件编译 |
| `cargo::rustc-cfg=KEY="VALUE"` | 设置 `#[cfg(KEY = "VALUE")]` 标志 |
| `cargo::rustc-env=KEY=VALUE` | 设置环境变量，可通过 `env!()` 访问 |
| `cargo::rustc-cdylib-link-arg=FLAG` | 将 FLAG 传递给 cdylib 目标的链接器 |
| `cargo::warning=MESSAGE` | 编译期间显示警告 |
| `cargo::metadata=KEY=VALUE` | 存储元数据，依赖 crate 可读 |
