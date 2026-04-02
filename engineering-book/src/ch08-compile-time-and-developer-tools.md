# 编译时和开发者工具 🟡

> **你将学到什么：**
> - 使用 `sccache` 进行本地和 CI 构建的编译缓存
> - 使用 `mold` 进行更快的链接（比默认链接器快 3-10 倍）
> - `cargo-nextest`：更快、信息量更大的测试运行器
> - 开发者可见性工具：`cargo-expand`、`cargo-geiger`、`cargo-watch`
> - 工作空间 lint、MSRV 策略和文档即 CI
>
> **交叉引用：** [发布配置文件](ch07-release-profiles-and-binary-size.md) —— LTO 和二进制大小优化 · [CI/CD 管道](ch11-putting-it-all-together-a-production-cic.md) —— 这些工具集成到你的管道中 · [依赖](ch06-dependency-management-and-supply-chain-s.md) —— 更少的依赖 = 更快的编译

### 编译时优化：sccache、mold、cargo-nextest

长编译时间是 Rust 中第一大开发者痛点。这些工具 collectively 可以将迭代时间减少 50-80%：

**`sccache` —— 共享编译缓存：**

```bash
# 安装
cargo install sccache

# 配置为 Rust 包装器
export RUSTC_WRAPPER=sccache

# 或在 .cargo/config.toml 中永久设置：
# [build]
# rustc-wrapper = "sccache"

# 第一次构建：正常速度（填充缓存）
cargo build --release  # 3 分钟

# 清理 + 重新构建：未更改的 crate 缓存命中
cargo clean && cargo build --release  # 45 秒

# 检查缓存统计
sccache --show-stats
# 编译请求        1,234
# 缓存命中           987 (80%)
# 缓存未命中         247
```

`sccache` 支持共享缓存（S3、GCS、Azure Blob）用于团队范围和 CI 缓存共享。

**`mold` —— 更快的链接器：**

链接通常是最慢的阶段。`mold` 比 `lld` 快 3-5 倍，比默认 GNU `ld` 快 10-20 倍：

```bash
# 安装
sudo apt install mold  # Ubuntu 22.04+
# 注意：mold 用于 ELF 目标（Linux）。macOS 使用 Mach-O，而非 ELF。
# macOS 链接器 (ld64) 已经相当快；如果你需要更快：
# brew install sold     # sold = mold for Mach-O（实验性，不太成熟）
# 在实践中，macOS 链接时间很少是瓶颈。
```

```toml
# 使用 mold 进行链接
# .cargo/config.toml
[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=mold"]
```

```bash
# 参见 https://github.com/rui314/mold/blob/main/docs/mold.md#environment-variables
export MOLD_JOBS=1

# 验证 mold 正在使用
cargo build -v 2>&1 | grep mold
```

**`cargo-nextest` —— 更快的测试运行器：**

```bash
# 安装
cargo install cargo-nextest

# 运行测试（默认并行，每测试超时，重试）
cargo nextest run

# 相对于 cargo test 的主要优势：
# - 每个测试在自己的进程中运行 → 更好的隔离
# - 带智能调度的并行执行
# - 每测试超时（不再有悬挂 CI）
# - 用于 CI 的 JUnit XML 输出
# - 重试失败的测试

# 配置
cargo nextest run --retries 2 --fail-fast

# 归档测试二进制文件（对 CI 有用：构建一次，在多台机器上测试）
cargo nextest archive --archive-file tests.tar.zst
cargo nextest run --archive-file tests.tar.zst
```

```toml
# .config/nextest.toml
[profile.default]
retries = 0
slow-timeout = { period = "60s", terminate-after = 3 }
fail-fast = true

[profile.ci]
retries = 2
fail-fast = false
junit = { path = "test-results.xml" }
```

**组合开发配置：**

```toml
# .cargo/config.toml —— 优化开发内部循环
[build]
rustc-wrapper = "sccache"       # 缓存编译产物

[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=mold"]  # 更快链接

# Dev 配置文件：优化依赖但不优化你的代码
# （放在 Cargo.toml 中）
# [profile.dev.package."*"]
# opt-level = 2
```

### cargo-expand 和 cargo-geiger —— 可见性工具

**`cargo-expand`** —— 查看宏生成什么：

```bash
cargo install cargo-expand

# 展开特定模块中的所有宏
cargo expand --lib accel_diag::vendor

# 展开特定的 derive
# 给定：#[derive(Debug, Serialize, Deserialize)]
# cargo expand 显示生成的 impl 块
cargo expand --lib --tests
```

对于调试 `#[derive]` 宏输出、`macro_rules!` 展开和理解 `serde` 为你的类型生成什么非常宝贵。

除了 `cargo-expand`，你还可以使用 rust-analyzer 展开宏：

1. 将光标移动到你想要检查的宏。
2. 打开命令面板（例如，在 VSCode 上按 `F1`）。
3. 搜索 `rust-analyzer: Expand macro recursively at caret`。

**`cargo-geiger`** —— 统计你的依赖树中的 `unsafe` 使用：

```bash
cargo install cargo-geiger

cargo geiger
# 输出：
# 指标输出格式：x/y
#   x = 构建使用的不安全代码
#   y = 在 crate 中找到的总不安全代码
#
# Functions  Expressions  Impls  Traits  Methods
# 0/0        0/0          0/0    0/0     0/0      ✅ my_crate
# 0/5        0/23         0/2    0/0     0/3      ✅ serde
# 3/3        14/14        0/0    0/0     2/2      ❗ libc
# 15/15      142/142      4/4    0/0     12/12    ☢️ ring

# 符号：
# ✅ = 无使用 unsafe
# ❗ = 使用一些 unsafe
# ☢️ = 重度 unsafe
```

对于项目的零 unsafe 策略，`cargo geiger` 验证没有依赖将 unsafe 代码引入你的代码实际练习的调用图中。

### 工作空间 Lints —— `[workspace.lints]`

从 Rust 1.74 开始，你可以在 `Cargo.toml` 中集中配置 Clippy 和编译器 lints —— 不再需要在每个 crate 顶部添加 `#![deny(...)]`：

```toml
# 根 Cargo.toml —— 所有 crate 的 lint 配置
[workspace.lints.clippy]
unwrap_used = "warn"         # 优先使用 ? 或 expect("reason")
dbg_macro = "deny"           # 提交的代码中无 dbg!()
todo = "warn"                # 跟踪不完整的实现
large_enum_variant = "warn"  # 捕获意外的大小膨胀

[workspace.lints.rust]
unsafe_code = "deny"         # 执行零 unsafe 策略
missing_docs = "warn"        # 鼓励文档
```

```toml
# 每个 crate 的 Cargo.toml —— 选择加入工作空间 lints
[lints]
workspace = true
```

这替换了分散的 `#![deny(clippy::unwrap_used)]` 属性，并确保整个工作空间的一致策略。

**自动修复 Clippy 警告：**

```bash
# 让 Clippy 自动修复机器可应用的建议
cargo clippy --fix --workspace --all-targets --allow-dirty

# 修复并应用可能改变行为的建议（仔细审查！）
cargo clippy --fix --workspace --all-targets --allow-dirty -- -W clippy::pedantic
```

> **提示**：在提交前运行 `cargo clippy --fix`。它处理繁琐的琐碎问题（未使用的导入、冗余克隆、类型简化），手动修复很麻烦。

### MSRV 策略和 rust-version

最低支持 Rust 版本（MSRV）确保你的 crate 在旧工具链上编译。这在部署到具有冻结 Rust 版本的系统时很重要。

```toml
# Cargo.toml
[package]
name = "diag_tool"
version = "0.1.0"
rust-version = "1.75"    # 所需的最低 Rust 版本
```

```bash
# 验证 MSRV 合规性
cargo +1.75.0 check --workspace

# 自动 MSRV 发现
cargo install cargo-msrv
cargo msrv find
# 输出：最低支持 Rust 版本是 1.75.0

# 在 CI 中验证
cargo msrv verify
```

**CI 中的 MSRV：**

```yaml
jobs:
  msrv:
    name: 检查 MSRV
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: dtolnay/rust-toolchain@master
        with:
          toolchain: "1.75.0"    # 匹配 Cargo.toml 中的 rust-version
      - run: cargo check --workspace
```

**MSRV 策略：**
- **二进制应用程序**（如此大型项目）：使用最新稳定版。不需要 MSRV。
- **库 crate**（发布到 crates.io）：设置 MSRV 为支持你使用的所有功能的最旧 Rust 版本。通常为 `N-2`（落后当前两个版本）。
- **企业部署**：设置 MSRV 以匹配你的集群上安装的最旧 Rust 版本。

### 应用：生产二进制配置文件

项目已经有极好的 [发布配置文件](ch07-release-profiles-and-binary-size.md)：

```toml
# 当前工作空间 Cargo.toml
[profile.release]
lto = true           # ✅ 完整跨 crate 优化
codegen-units = 1    # ✅ 最大优化
panic = "abort"      # ✅ 无展开开销
strip = true         # ✅ 部署时移除符号

[profile.dev]
opt-level = 0        # ✅ 快速编译
debug = true         # ✅ 完整调试信息
```

**推荐的补充：**

```toml
# 在 dev 模式下优化依赖（更快的测试执行）
[profile.dev.package."*"]
opt-level = 2

# Test 配置文件：一些优化以防止慢测试超时
[profile.test]
opt-level = 1

# 在发布版本中保留溢出检查（安全）
[profile.release]
lto = true
codegen-units = 1
panic = "abort"
strip = true
overflow-checks = true    # ← 添加这个：捕获整数溢出
debug = "line-tables-only" # ← 添加这个：回溯无完整 DWARF
```

**推荐的开发员工具：**

```toml
# .cargo/config.toml（建议）
[build]
rustc-wrapper = "sccache"  # 第一次构建后 80%+ 缓存命中率

[target.x86_64-unknown-linux-gnu]
rustflags = ["-C", "link-arg=-fuse-ld=mold"]  # 3-5 倍更快链接
```

**对项目的预期影响：**

| 指标 | 当前 | 补充后 |
|--------|---------|----------------|
| 发布二进制 | ~10 MB（剥离，LTO） | 相同 |
| Dev 构建时间 | ~45 秒 | ~25 秒（sccache + mold） |
| 重新构建（1 个文件更改） | ~15 秒 | ~5 秒（sccache + mold） |
| 测试执行 | `cargo test` | `cargo nextest` —— 2 倍更快 |
| 依赖漏洞扫描 | 无 | CI 中的 `cargo audit` |
| 许可证合规 | 手动 | `cargo deny` 自动化 |
| 未使用依赖检测 | 手动 | CI 中的 `cargo udeps` |

### `cargo-watch` —— 文件更改时自动重新构建

[`cargo-watch`](https://github.com/watchexec/cargo-watch) 每次源文件更改时重新运行命令 —— 对于紧密反馈循环至关重要：

```bash
# 安装
cargo install cargo-watch

# 每次保存时重新检查（即时反馈）
cargo watch -x check

# 更改时运行 clippy + 测试
cargo watch -x 'clippy --workspace --all-targets' -x 'test --workspace --lib'

# 仅监视特定 crate（对大型工作空间更快）
cargo watch -w accel_diag/src -x 'test -p accel_diag'

# 在运行之间清屏
cargo watch -c -x check
```

> **提示**：与上面的 `mold` + `sccache` 结合，在增量更改时实现亚秒级重新检查时间。

### `cargo doc` 和工作空间文档

对于大型工作空间，生成的文档对于可发现性至关重要。`cargo doc` 使用 rustdoc 从文档注释和类型签名生成 HTML 文档：

```bash
# 为所有工作空间 crate 生成文档（在浏览器中打开）
cargo doc --workspace --no-deps --open

# 包括私有项（在开发期间有用）
cargo doc --workspace --no-deps --document-private-items

# 检查文档链接而不生成 HTML（快速 CI 检查）
cargo doc --workspace --no-deps 2>&1 | grep -E 'warning|error'
```

**内部文档链接** —— 跨 crate 链接类型而无需 URL：

```rust
/// 使用 [`GpuConfig`] 设置运行 GPU 诊断。
///
/// 参见 [`crate::accel_diag::run_diagnostics`] 获取实现。
/// 返回 [`DiagResult`] 可序列化为
/// [`DerReport`](crate::core_lib::DerReport) 格式。
pub fn run_accel_diag(config: &GpuConfig) -> DiagResult {
    // ...
}
```

**在文档中显示平台特定 API：**

```rust
// Cargo.toml: [package.metadata.docs.rs]
// all-features = true
// rustdoc-args = ["--cfg", "docsrs"]

/// 仅限 Windows：通过 Win32 API 读取电池状态。
///
/// 仅在 `cfg(windows)` 构建上可用。
#[cfg(windows)]
#[doc(cfg(windows))]  // 在文档中显示"仅在 Windows 上可用"徽章
pub fn get_battery_status() -> Option<u8> {
    // ...
}
```

**CI 文档检查：**

```yaml
# 添加到 CI 工作流
- name: 检查文档
  run: RUSTDOCFLAGS="-D warnings" cargo doc --workspace --no-deps
  # 将损坏的内部文档链接视为错误
```

> **对于项目**：对于许多 crate，`cargo doc --workspace` 是新团队成员发现 API 表面的最佳方式。在 CI 中添加 `RUSTDOCFLAGS="-D warnings"` 以在合并前捕获损坏的文档链接。

### 编译时决策树

```mermaid
flowchart TD
    START["编译太慢？"] --> WHERE{"时间在哪里？"}

    WHERE -->|"重新编译\n未更改的 crate"| SCCACHE["sccache\n共享编译缓存"]
    WHERE -->|"链接阶段"| MOLD["mold 链接器\n3-10 倍更快链接"]
    WHERE -->|"运行测试"| NEXTEST["cargo-nextest\n并行测试运行器"]
    WHERE -->|"一切"| COMBO["以上所有内容 +\ncargo-udeps 修剪依赖"]

    SCCACHE --> CI_CACHE{"CI 或本地？"}
    CI_CACHE -->|"CI"| S3["S3/GCS 共享缓存"]
    CI_CACHE -->|"本地"| LOCAL["本地磁盘缓存\n自动配置"]

    style SCCACHE fill:#91e5a3,color:#000
    style MOLD fill:#e3f2fd,color:#000
    style NEXTEST fill:#ffd43b,color:#000
    style COMBO fill:#b39ddb,color:#000
```

### 🏋️ 练习

#### 🟢 练习 1：设置 sccache + mold

安装 `sccache` 和 `mold`，在 `.cargo/config.toml` 中配置它们，然后测量干净重新构建上的编译时间改进。

<details>
<summary>答案</summary>

```bash
# 安装
cargo install sccache
sudo apt install mold  # Ubuntu 22.04+

# 配置 .cargo/config.toml：
cat > .cargo/config.toml << 'EOF'
[build]
rustc-wrapper = "sccache"

[target.x86_64-unknown-linux-gnu]
linker = "clang"
rustflags = ["-C", "link-arg=-fuse-ld=mold"]
EOF

# 第一次构建（填充缓存）
time cargo build --release  # 例如，180 秒

# 清理 + 重新构建（缓存命中）
cargo clean
time cargo build --release  # 例如，45 秒

sccache --show-stats
# 缓存命中率应该为 60-80%+
```
</details>

#### 🟡 练习 2：切换到 cargo-nextest

安装 `cargo-nextest` 并运行你的测试套件。与 `cargo test` 比较挂钟时间。加速多少？

<details>
<summary>答案</summary>

```bash
cargo install cargo-nextest

# 标准测试运行器
time cargo test --workspace 2>&1 | tail -5

# nextest（每测试二进制并行执行）
time cargo nextest run --workspace 2>&1 | tail -5

# 大型工作空间的典型加速：2-5 倍
# nextest 还提供：
# - 每测试计时
# - 对不稳定测试的重试
# - 用于 CI 的 JUnit XML 输出
cargo nextest run --workspace --retries 2
```
</details>

### 关键要点

- 带 S3/GCS 后端的 `sccache` 在团队和 CI 之间共享编译缓存
- `mold` 是最快的 ELF 链接器 —— 链接时间从秒降到毫秒
- `cargo-nextest` 每二进制并行运行测试，带更好的输出和重试支持
- `cargo-geiger` 统计 `unsafe` 使用 —— 在接受新依赖前运行它
- `[workspace.lints]` 在多 crate 工作空间中集中 Clippy 和 rustc lint 配置

---
