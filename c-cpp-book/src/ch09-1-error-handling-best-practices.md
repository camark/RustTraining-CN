# Rust Option 和 Result 核心要点

> **你将学到什么：** 惯用的错误处理模式 —— `unwrap()` 的安全替代方案、用于传播错误的 `?` 运算符、自定义错误类型，以及在生产代码中何时使用 `anyhow` vs `thiserror`。

- ```Option``` 和 ```Result``` 是惯用 Rust 不可或缺的部分
- **`unwrap()` 的安全替代方案**：
```rust
// Option<T> 的安全替代方案
let value = opt.unwrap_or(default);              // 提供回退值
let value = opt.unwrap_or_else(|| compute());    // 惰性计算回退值
let value = opt.unwrap_or_default();             // 使用 Default trait 实现
let value = opt.expect("描述性消息");   // 仅在 panic 可接受时使用

// Result<T, E> 的安全替代方案  
let value = result.unwrap_or(fallback);          // 忽略错误，使用回退值
let value = result.unwrap_or_else(|e| handle(e)); // 处理错误，返回回退值
let value = result.unwrap_or_default();          // 使用 Default trait
```
- **用于显式控制的模式匹配**：
```rust
match some_option {
    Some(value) => println!("得到：{}", value),
    None => println!("未找到值"),
}

match some_result {
    Ok(value) => process(value),
    Err(error) => log_error(error),
}
```
- **使用 `?` 运算符进行错误传播**：短路并向上冒泡错误
```rust
fn process_file(path: &str) -> Result<String, std::io::Error> {
    let content = std::fs::read_to_string(path)?; // 自动返回错误
    Ok(content.to_uppercase())
}
```
- **转换方法**：
    - `map()`：转换成功值 `Ok(T)` -> `Ok(U)` 或 `Some(T)` -> `Some(U)`
    - `map_err()`：转换错误类型 `Err(E)` -> `Err(F)`
    - `and_then()`：链接可能失败的操作
- **在你自己的 API 中使用**：优先使用 `Result<T, E>` 而不是异常或错误码
- **参考**：[Option 文档](https://doc.rust-lang.org/std/option/enum.Option.html) | [Result 文档](https://doc.rust-lang.org/std/result/enum.Result.html)

# Rust 常见陷阱和调试技巧
- **借用问题**：最常见的初学者错误
    - "cannot borrow as mutable"（无法可变借用）→ 同时只允许一个可变引用
    - "borrowed value does not live long enough"（借用值存活时间不够长）→ 引用比其指向的数据活得更长
    - **修复**：使用作用域 `{}` 限制引用生命周期，或在需要时克隆数据
- **缺少 trait 实现**："method not found"（方法未找到）错误
    - **修复**：添加 `#[derive(Debug, Clone, PartialEq)]` 获取通用 traits
    - 使用 `cargo check` 获取比 `cargo run` 更好的错误消息
- **调试模式下的整数溢出**：Rust 在溢出时 panic
    - **修复**：使用 `wrapping_add()`、`saturating_add()` 或 `checked_add()` 获取显式行为
- **String vs &str 混淆**：不同用途的不同类型
    - 使用 `&str` 表示字符串切片（借用），`String` 表示自有字符串
    - **修复**：使用 `.to_string()` 或 `String::from()` 将 `&str` 转换为 `String`
- **与借用检查器斗争**：不要试图智胜它
    - **修复**：重构代码以与所有权规则合作而不是对抗
    - 考虑使用 `Rc<RefCell<T>>` 处理复杂的共享场景（谨慎使用）

## 错误处理示例：好 vs 坏
```rust
// [错误] 坏：可能意外 panic
fn bad_config_reader() -> String {
    let config = std::env::var("CONFIG_FILE").unwrap(); // 如果未设置则 panic！
    std::fs::read_to_string(config).unwrap()           // 如果文件缺失则 panic！
}

// [好] 好：优雅处理错误
fn good_config_reader() -> Result<String, ConfigError> {
    let config_path = std::env::var("CONFIG_FILE")
        .unwrap_or_else(|_| "default.conf".to_string()); // 回退到默认值
    
    let content = std::fs::read_to_string(config_path)
        .map_err(ConfigError::FileRead)?;                // 转换并传播错误
    
    Ok(content)
}

// [好] 更好：带适当的错误类型
use thiserror::Error;

#[derive(Error, Debug)]
enum ConfigError {
    #[error("读取配置文件失败：{0}")]
    FileRead(#[from] std::io::Error),
    
    #[error("配置无效：{message}")]
    Invalid { message: String },
}
```

让我们分解这里发生的事情。`ConfigError` 只有**两个变体** —— 一个用于 I/O 错误，一个用于验证错误。这是大多数模块的正确起点：

| `ConfigError` 变体 | 包含 | 创建者 |
|----------------------|-------|-----------|
| `FileRead(io::Error)` | 原始 I/O 错误 | `#[from]` 通过 `?` 自动转换 |
| `Invalid { message }` | 人类可读的解释 | 你的验证代码 |

现在你可以编写返回 `Result<T, ConfigError>` 的函数：

```rust
fn read_config(path: &str) -> Result<String, ConfigError> {
    let content = std::fs::read_to_string(path)?;  // io::Error → ConfigError::FileRead
    if content.is_empty() {
        return Err(ConfigError::Invalid {
            message: "配置文件为空".to_string(),
        });
    }
    Ok(content)
}
```

> **🟢 自学检查点：** 在继续之前，确保你能回答：
> 1. 为什么 `read_to_string` 调用上的 `?` 能工作？（因为 `#[from]` 生成 `impl From<io::Error> for ConfigError`）
> 2. 如果添加第三个变体 `MissingKey(String)` 会发生什么变化？（只需添加变体；现有代码仍能编译）

## Crate 级错误类型和 Result 别名

随着项目发展到单个文件之外，你会将多个模块级错误组合成 **crate 级错误类型**。这是生产 Rust 中的标准模式。让我们从上面的 `ConfigError` 开始构建。

在现实世界的 Rust 项目中，每个 crate（或重要模块）定义自己的 `Error`
枚举和 `Result` 类型别名。这是惯用模式 —— 类似于在 C++ 中
你定义每个库的异常层次结构和 `using Result = std::expected<T, Error>`。

### 模式

```rust
// src/error.rs（或在 lib.rs 顶部）
use thiserror::Error;

/// 这个 crate 可以产生的每个错误。
#[derive(Error, Debug)]
pub enum Error {
    #[error("I/O 错误：{0}")]
    Io(#[from] std::io::Error),          // 通过 From 自动转换

    #[error("JSON 解析错误：{0}")]
    Json(#[from] serde_json::Error),     // 通过 From 自动转换

    #[error("传感器 id 无效：{0}")]
    InvalidSensor(u32),                  // 领域特定变体

    #[error("{ms} ms 后超时")]
    Timeout { ms: u64 },
}

/// Crate 范围的 Result 别名 —— 节省整个 crate 中的输入。
pub type Result<T> = core::result::Result<T, Error>;
```

### 它如何简化每个函数

没有别名你这样写：

```rust
// 冗长 —— 错误类型在各处重复
fn read_sensor(id: u32) -> Result<f64, crate::Error> { ... }
fn parse_config(path: &str) -> Result<Config, crate::Error> { ... }
```

有别名：

```rust
// 简洁 —— 只需 `Result<T>`
use crate::{Error, Result};

fn read_sensor(id: u32) -> Result<f64> {
    if id > 128 {
        return Err(Error::InvalidSensor(id));
    }
    let raw = std::fs::read_to_string(format!("/dev/sensor/{id}"))?; // io::Error → Error::Io
    let value: f64 = raw.trim().parse()
        .map_err(|_| Error::InvalidSensor(id))?;
    Ok(value)
}
```

`#[from]` 属性在 `Io` 上免费生成这个 `impl`：

```rust
// 由 thiserror 的 #[from] 自动生成
impl From<std::io::Error> for Error {
    fn from(source: std::io::Error) -> Self {
        Error::Io(source)
    }
}
```

这就是 `?` 工作的原因：当函数返回 `std::io::Error` 而你的函数
返回 `Result<T>`（你的别名）时，编译器自动调用 `From::from()` 来转换它。

### 组合模块级错误

更大的 crate 按模块拆分错误，然后在 crate 根目录组合它们：

```rust
// src/config/error.rs
#[derive(thiserror::Error, Debug)]
pub enum ConfigError {
    #[error("缺少键：{0}")]
    MissingKey(String),
    #[error("'{key}' 的值无效：{reason}")]
    InvalidValue { key: String, reason: String },
}

// src/error.rs（crate 级）
#[derive(thiserror::Error, Debug)]
pub enum Error {
    #[error(transparent)]               // 委托 Display 给内部错误
    Config(#[from] crate::config::ConfigError),

    #[error("I/O 错误：{0}")]
    Io(#[from] std::io::Error),
}
pub type Result<T> = core::result::Result<T, Error>;
```

调用者仍然可以匹配特定的配置错误：

```rust
match result {
    Err(Error::Config(ConfigError::MissingKey(k))) => eprintln!("添加 '{k}' 到配置"),
    Err(e) => eprintln!("其他错误：{e}"),
    Ok(v) => use_value(v),
}
```

### C++ 对比

| 概念 | C++ | Rust |
|---------|-----|------|
| 错误层次结构 | `class AppError : public std::runtime_error` | `#[derive(thiserror::Error)] enum Error { ... }` |
| 返回错误 | `std::expected<T, Error>` 或 `throw` | `fn foo() -> Result<T>` |
| 转换错误 | 手动 `try/catch` + 重新抛出 | `#[from]` + `?` —— 零样板代码 |
| Result 别名 | `template<class T> using Result = std::expected<T, Error>;` | `pub type Result<T> = core::result::Result<T, Error>;` |
| 错误消息 | 覆盖 `what()` | `#[error("...")]` —— 编译进 `Display` 实现 |

