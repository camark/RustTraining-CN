### Unsafe Rust

> **你将学到什么：** 何时以及如何使用 `unsafe` —— 原始指针解引用、FFI（外部函数接口）用于从 Rust 调用 C 以及反之亦然、`CString`/`CStr` 用于字符串互操作，以及如何围绕 unsafe 代码编写安全的包装器。

- ```unsafe``` 解锁访问通常被 Rust 编译器禁止的功能
    - 解引用原始指针
    - 访问*可变*静态变量
    - https://doc.rust-lang.org/book/ch19-01-unsafe-rust.html
- 能力越大，责任越大
    - ```unsafe``` 告诉编译器"我，程序员，负责维护编译器通常保证的不变量"
    - 必须保证没有混叠的可变和不可变引用、没有悬空指针、没有无效引用……
    - ```unsafe``` 的使用应限制在尽可能小的作用域
    - 所有使用 ```unsafe``` 的代码都应该有"安全"注释描述假设

### Unsafe Rust 示例
```rust
unsafe fn harmless() {}
fn main() {
    // 安全：我们调用一个无害的 unsafe 函数
    unsafe {
        harmless();
    }
    let a = 42u32;
    let p = &a as *const u32;
    // 安全：p 是指向有效变量的指针，它将在作用域内保持有效
    unsafe {
        println!("{}", *p);
    }
    // 安全：不安全；仅用于说明目的
    let dangerous_buffer = 0xb8000 as *mut u32;
    unsafe {
        println!("即将崩溃！！！");
        *dangerous_buffer = 0; // 这在大多数现代机器上会导致 SEGV
    }
}
```

### 简单 FFI 示例（Rust 库函数被 C 使用）

## FFI 字符串：CString 和 CStr

FFI 代表*外部函数接口* —— Rust 用于调用其他语言（如 C）编写的函数的机制，反之亦然。

在与 C 代码交互时，Rust 的 `String` 和 `&str` 类型（无 null 终止符的 UTF-8）与 C 字符串（null 终止的字节数组）不直接兼容。Rust 从 `std::ffi` 提供 `CString`（自有）和 `CStr`（借用）用于此目的：

| 类型 | 类似于 | 何时使用 |
|------|-------------|----------|
| `CString` | `String`（自有） | 从 Rust 数据创建 C 字符串 |
| `&CStr` | `&str`（借用） | 从外部代码接收 C 字符串 |

```rust
use std::ffi::{CString, CStr};
use std::os::raw::c_char;

fn demo_ffi_strings() {
    // 创建 C 兼容字符串（添加 null 终止符）
    let c_string = CString::new("Hello from Rust").expect("CString::new 失败");
    let ptr: *const c_char = c_string.as_ptr();

    // 将 C 字符串转换回 Rust（不安全因为我们信任指针）
    // 安全：ptr 有效且 null 终止（我们刚刚在上面创建了它）
    let back_to_rust: &CStr = unsafe { CStr::from_ptr(ptr) };
    let rust_str: &str = back_to_rust.to_str().expect("无效的 UTF-8");
    println!("{}", rust_str);
}
```

> **警告**：如果输入包含内部 null 字节（`\0`），`CString::new()` 将返回错误。始终处理 `Result`。你将在下面的 FFI 示例中看到 extensively 使用 `CStr`。

- ```FFI``` 方法必须用 ```#[no_mangle]``` 标记以确保编译器不修改名称
- 我们将 crate 编译为静态库
    ```
    #[no_mangle] 
    pub extern "C" fn add(left: u64, right: u64) -> u64 {
        left + right
    }
    ```
- 我们将编译以下 C 代码并链接到我们的静态库。
    ```
    #include <stdio.h>
    #include <stdint.h>
    extern uint64_t add(uint64_t, uint64_t);
    int main() {
        printf("Add 返回 %llu\n", add(21, 21));
    }
    ``` 

### 复杂 FFI 示例
- 在以下示例中，我们将创建一个 Rust 日志接口并将其暴露给
[PYTHON] 和 ```C```
    - 我们将看到如何从 Rust 和 C 原生使用相同的接口
    - 我们将探索使用 ```cbindgen``` 等工具为 ```C``` 生成头文件
    - 我们将看到 ```unsafe``` 包装器如何充当安全 Rust 代码的桥梁

## Logger 辅助函数
```rust
fn create_or_open_log_file(log_file: &str, overwrite: bool) -> Result<File, String> {
    if overwrite {
        File::create(log_file).map_err(|e| e.to_string())
    } else {
        OpenOptions::new()
            .write(true)
            .append(true)
            .open(log_file)
            .map_err(|e| e.to_string())
    }
}

fn log_to_file(file_handle: &mut File, message: &str) -> Result<(), String> {
    file_handle
        .write_all(message.as_bytes())
        .map_err(|e| e.to_string())
}
```

## Logger 结构体
```rust
struct SimpleLogger {
    log_level: LogLevel,
    file_handle: File,
}

impl SimpleLogger {
    fn new(log_file: &str, overwrite: bool, log_level: LogLevel) -> Result<Self, String> {
        let file_handle = create_or_open_log_file(log_file, overwrite)?;
        Ok(Self {
            file_handle,
            log_level,
        })
    }

    fn log_message(&mut self, log_level: LogLevel, message: &str) -> Result<(), String> {
        if log_level as u32 <= self.log_level as u32 {
            let timestamp = Local::now().format("%Y-%m-%d %H:%M:%S").to_string();
            let message = format!("Simple: {timestamp} {log_level} {message}\n");
            log_to_file(&mut self.file_handle, &message)
        } else {
            Ok(())
        }
    }
}
```

## 测试
- 使用 Rust 测试功能很简单
    - 测试方法用 ```#[test]``` 装饰，不是编译二进制文件的一部分
    - 很容易为测试目的创建模拟方法
```rust
#[test]
fn testfunc() -> Result<(), String> {
    let mut logger = SimpleLogger::new("test.log", false, LogLevel::INFO)?;
    logger.log_message(LogLevel::TRACELEVEL1, "Hello world")?;
    logger.log_message(LogLevel::CRITICAL, "Critical message")?;
    Ok(()) // 编译器在这里自动 drop logger
}
```
```bash
cargo test
```

## (C)-Rust FFI
- cbindgen 是为导出的 Rust 函数生成头文件的好工具
    - 可以使用 cargo 安装
```bash
cargo install cbindgen
cbindgen 
```
- 函数和结构体可以使用 ```#[no_mangle]``` 和 ```#[repr(C)]``` 导出
    - 我们假设通用的接口模式，传入 `**` 到实际实现，成功返回 0，错误返回非零
    - **不透明 vs 透明结构体**：我们的 `SimpleLogger` 作为*不透明指针*（`*mut SimpleLogger`）传递 —— C 端从不访问其字段，所以 `#[repr(C)]`**不**需要。当 C 代码需要直接读/写结构体字段时使用 `#[repr(C)]`：

```rust
// 不透明 —— C 仅持有指针，从不检查字段。不需要 #[repr(C)]。
struct SimpleLogger { /* 仅 Rust 字段 */ }

// 透明 —— C 直接读/写字段。必须使用 #[repr(C)]。
#[repr(C)]
pub struct Point {
    pub x: f64,
    pub y: f64,
}
```
```c
typedef struct SimpleLogger SimpleLogger;
uint32_t create_simple_logger(const char *file_name, struct SimpleLogger **out_logger);
uint32_t log_entry(struct SimpleLogger *logger, const char *message);
uint32_t drop_logger(struct SimpleLogger *logger);
```

- 注意我们需要大量健全性检查
- 我们必须显式泄漏内存以防止 Rust 自动释放
```rust
#[no_mangle] 
pub extern "C" fn create_simple_logger(file_name: *const std::os::raw::c_char, out_logger: *mut *mut SimpleLogger) -> u32 {
    use std::ffi::CStr;
    // 确保指针不为 NULL
    if file_name.is_null() || out_logger.is_null() {
        return 1;
    }
    // 安全：传入的指针根据约定为 NULL 或以 0 终止
    let file_name = unsafe {
        CStr::from_ptr(file_name)
    };
    let file_name = file_name.to_str();
    // 确保 file_name 没有垃圾字符
    if file_name.is_err() {
        return 1;
    }
    let file_name = file_name.unwrap();
    // 假设一些默认值；在现实生活中我们会传入它们
    let new_logger = SimpleLogger::new(file_name, false, LogLevel::CRITICAL);
    // 检查我们是否能够构造 logger
    if new_logger.is_err() {
        return 1;
    }
    let new_logger = Box::new(new_logger.unwrap());
    // 这防止 Box 在离开作用域时被 drop
    let logger_ptr: *mut SimpleLogger = Box::leak(new_logger);
    // 安全：logger 非空且 logger_ptr 有效
    unsafe {
        *out_logger = logger_ptr;
    }
    return 0;
}
```

- 我们在 ```log_entry()``` 中有类似的错误检查
```rust
#[no_mangle]
pub extern "C" fn log_entry(logger: *mut SimpleLogger, message: *const std::os::raw::c_char) -> u32 {
    use std::ffi::CStr;
    if message.is_null() || logger.is_null() {
        return 1;
    }
    // 安全：message 非空
    let message = unsafe {
        CStr::from_ptr(message)
    };
    let message = message.to_str();
    // 确保 file_name 没有垃圾字符
    if message.is_err() {
        return 1;
    }
    // 安全：logger 是之前由 create_simple_logger() 构造的有效指针
    unsafe {
        (*logger).log_message(LogLevel::CRITICAL, message.unwrap()).is_err() as u32
    }
}

#[no_mangle]
pub extern "C" fn drop_logger(logger: *mut SimpleLogger) -> u32 {
    if logger.is_null() {
        return 1;
    }
    // 安全：logger 是之前由 create_simple_logger() 构造的有效指针
    unsafe {
        // 这构造一个 Box<SimpleLogger>，它在离开作用域时被 drop
        let _ = Box::from_raw(logger);
    }
    0
}
```

- 我们可以使用 Rust 或通过编写 (C) 程序来测试我们的 (C)-FFI
```rust
#[test]
fn test_c_logger() {
    // c".." 创建 NULL 终止字符串
    let file_name = c"test.log".as_ptr() as *const std::os::raw::c_char;
    let mut c_logger: *mut SimpleLogger = std::ptr::null_mut();
    assert_eq!(create_simple_logger(file_name, &mut c_logger), 0);
    // 这是手动创建 c"..." 字符串的方式
    let message = b"message from C\0".as_ptr() as *const std::os::raw::c_char;
    assert_eq!(log_entry(c_logger, message), 0);
    drop_logger(c_logger);
}
```
```c
#include "logger.h"
...
int main() {
    SimpleLogger *logger = NULL;
    if (create_simple_logger("test.log", &logger) == 0) {
        log_entry(logger, "Hello from C");
        drop_logger(logger); /*需要关闭句柄等*/
    } 
    ...
}
```

## 确保 unsafe 代码的正确性
- TL;DR 版本是使用 ```unsafe``` 需要深思熟虑
    - 始终记录代码的安全假设并与专家一起审查
    - 使用 cbindgen、Miri、Valgrind 等工具帮助验证正确性
    - **永远不要让 panic 跨越 FFI 边界展开** —— 这是 UB。在 FFI 入口点使用 `std::panic::catch_unwind`，或在你的 profile 中配置 `panic = "abort"`
    - 如果结构体在 FFI 间共享，标记为 `#[repr(C)]` 以保证 C 兼容的内存布局
    - 参考 https://doc.rust-lang.org/nomicon/intro.html（"Rustonomicon" —— unsafe Rust 的黑魔法）
    - 寻求内部专家的帮助

### 验证工具：Miri vs Valgrind

C++ 开发者熟悉 Valgrind 和 sanitizers。Rust 有这些工具**加上** Miri，它对 Rust 特定的 UB 更精确：

| | **Miri** | **Valgrind** | **C++ sanitizers（ASan/MSan/UBSan）** |
|---|---------|-------------|--------------------------------------|
| **它捕获什么** | Rust 特定的 UB：堆叠借用、无效的 `enum` 判别式、未初始化读取、混叠违规 | 内存泄漏、释放后使用、无效读/写、未初始化内存 | 缓冲区溢出、释放后使用、数据竞争、UB |
| **如何工作** | 解释 MIR（Rust 的中级 IR） —— 不执行原生代码 | 在运行时检测编译的二进制文件 | 编译时检测 |
| **FFI 支持** | ❌ 不能跨越 FFI 边界（跳过 C 调用） | ✅ 适用于任何编译的二进制文件，包括 FFI | ✅ 如果 C 代码也用 sanitizers 编译 |
| **速度** | 比原生慢 ~100 倍 | 慢 ~10-50 倍 | 慢 ~2-5 倍 |
| **何时使用** | 纯 Rust unsafe 代码、数据结构不变量 | FFI 代码、完整二进制集成测试 | FFI 的 C/C++ 端、性能敏感测试 |
| **捕获混叠 bug** | ✅ 堆叠借用模型 | ❌ | 部分（TSan 用于数据竞争） |

**推荐**：**两者都使用** —— Miri 用于纯 Rust unsafe，Valgrind 用于 FFI 集成：

- **Miri** —— 捕获 Valgrind 看不到的 Rust 特定 UB（混叠违规、无效 enum 值、堆叠借用）：
    ```
    rustup +nightly component add miri
    cargo +nightly miri test                    # 在 Miri 下运行所有测试
    cargo +nightly miri test -- test_name       # 运行特定测试
    ```
    > ⚠️ Miri 需要 nightly 且不能执行 FFI 调用。将 unsafe Rust 逻辑隔离到可测试单元中。

- **Valgrind** —— 你已经知道的工具，适用于包括 FFI 的编译二进制文件：
    ```
    sudo apt install valgrind
    cargo install cargo-valgrind
    cargo valgrind test                         # 在 Valgrind 下运行所有测试
    ```
    > 捕获 FFI 代码中常见的 `Box::leak` / `Box::from_raw` 模式的泄漏。

- **cargo-careful** —— 运行启用额外运行时检查的测试（在常规测试和 Miri 之间）：
    ```
    cargo install cargo-careful
    cargo +nightly careful test
    ```

## Unsafe Rust 总结
- ```cbindgen``` 是 (C) FFI 到 Rust 的好工具
    - 使用 ```bindgen``` 用于相反方向的 FFI 接口（参考广泛的文档）
- **不要假设你的 unsafe 代码是正确的，或者从安全 Rust 使用没问题。很容易出错，甚至看似正确工作的代码可能因微妙原因而错误**
    - 使用工具验证正确性
    - 如果仍然有疑问，寻求专家建议
- 确保你的 ```unsafe``` 代码有关于假设和为什么正确的显式文档注释
    - ```unsafe``` 代码的调用者也应该有关于安全的相应注释，并遵守限制

# 练习：编写安全的 FFI 包装器

🔴 **挑战** —— 需要理解 unsafe 块、原始指针和安全 API 设计

- 编写一个安全 Rust 包装器，包装一个 `unsafe` FFI 风格函数。此练习模拟调用一个 C 函数，该函数将格式化的字符串写入调用者提供的缓冲区。
- **步骤 1**：实现 unsafe 函数 `unsafe_greet`，将问候语写入原始 `*mut u8` 缓冲区
- **步骤 2**：编写一个安全的包装器 `safe_greet`，分配一个 `Vec<u8>`，调用 unsafe 函数，并返回一个 `String`
- **步骤 3**：为每个 unsafe 块添加适当的 `// Safety:` 注释

**起始代码：**
```rust
use std::fmt::Write as _;

/// 模拟 C 函数：将 "Hello, <name>!" 写入缓冲区。
/// 返回写入的字节数（不包括 null 终止符）。
/// # 安全
/// - `buf` 必须指向至少 `buf_len` 可写字节
/// - `name` 必须是指向 null 终止 C 字符串的有效指针
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // 待完成：构建问候语，复制字节到 buf，返回长度
    // 提示：使用 std::ffi::CStr::from_ptr 或手动迭代字节
    todo!()
}

/// 安全包装器 —— 公共 API 中没有 unsafe
fn safe_greet(name: &str) -> Result<String, String> {
    // 待完成：分配 Vec<u8> 缓冲区，创建 null 终止的名称，
    // 在带安全注释的 unsafe 块中调用 unsafe_greet，
    // 将结果转换回 String
    todo!()
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("错误：{e}"),
    }
    // 预期输出：Hello, Rustacean!
}
```

<details><summary>答案（点击展开）</summary>

```rust
use std::ffi::CStr;

/// 模拟 C 函数：将 "Hello, <name>!" 写入缓冲区。
/// 返回写入的字节数，如果缓冲区太小则返回 -1。
/// # 安全
/// - `buf` 必须指向至少 `buf_len` 可写字节
/// - `name` 必须是指向 null 终止 C 字符串的有效指针
unsafe fn unsafe_greet(buf: *mut u8, buf_len: usize, name: *const u8) -> isize {
    // 安全：调用者保证 name 是有效的 null 终止字符串
    let name_cstr = unsafe { CStr::from_ptr(name as *const std::os::raw::c_char) };
    let name_str = match name_cstr.to_str() {
        Ok(s) => s,
        Err(_) => return -1,
    };
    let greeting = format!("Hello, {}!", name_str);
    if greeting.len() > buf_len {
        return -1;
    }
    // 安全：buf 指向至少 buf_len 可写字节（调用者保证）
    unsafe {
        std::ptr::copy_nonoverlapping(greeting.as_ptr(), buf, greeting.len());
    }
    greeting.len() as isize
}

/// 安全包装器 —— 公共 API 中没有 unsafe
fn safe_greet(name: &str) -> Result<String, String> {
    let mut buffer = vec![0u8; 256];
    // 为 C API 创建 null 终止版本的 name
    let name_with_null: Vec<u8> = name.bytes().chain(std::iter::once(0)).collect();

    // 安全：缓冲区有 256 可写字节，name_with_null 是 null 终止的
    let bytes_written = unsafe {
        unsafe_greet(buffer.as_mut_ptr(), buffer.len(), name_with_null.as_ptr())
    };

    if bytes_written < 0 {
        return Err("缓冲区太小或无效的名称".to_string());
    }

    String::from_utf8(buffer[..bytes_written as usize].to_vec())
        .map_err(|e| format!("无效的 UTF-8: {e}"))
}

fn main() {
    match safe_greet("Rustacean") {
        Ok(msg) => println!("{msg}"),
        Err(e) => eprintln!("错误：{e}"),
    }
}
// 输出：
// Hello, Rustacean!
```

</details>

----


