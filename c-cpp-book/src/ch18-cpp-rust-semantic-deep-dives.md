## C++ → Rust 语义深入探讨

> **你将学到什么：** C++ 概念到 Rust 等价物的详细映射 —— 四个命名转换、SFINAE vs trait 约束、CRTP vs 关联类型，以及其他翻译过程中的常见摩擦点。

下面的章节映射 C++ 中没有明显 1:1 Rust 等价物的概念。这些差异在翻译工作期间经常绊倒 C++ 程序员。

### 转换层次结构：四个 C++ 转换 → Rust 等价物

C++ 有四个命名转换。Rust 用不同的、更显式的机制替换它们：

```cpp
// C++ 转换层次结构
int i = static_cast<int>(3.14);            // 1. 数值转换 / 向上转换
Derived* d = dynamic_cast<Derived*>(base); // 2. 运行时向下转换
int* p = const_cast<int*>(cp);              // 3. 移除 const
auto* raw = reinterpret_cast<char*>(&obj); // 4. 位级重新解释
```

| C++ 转换 | Rust 等价物 | 安全性 | 注释 |
|----------|----------------|--------|-------|
| `static_cast` (数值) | `as` 关键字 | 安全但可能截断/回绕 | `let i = 3.14_f64 as i32;` —— 截断为 3 |
| `static_cast` (数值，检查) | `From`/`Into` | 安全，编译时验证 | `let i: i32 = 42_u8.into();` —— 仅 widening |
| `static_cast` (数值，可失败) | `TryFrom`/`TryInto` | 安全，返回 `Result` | `let i: u8 = 300_u16.try_into()?;` —— 返回 Err |
| `dynamic_cast` (向下转换) | enum 上的 `match` / `Any::downcast_ref` | 安全 | enum 用模式匹配；trait 对象用 `Any` |
| `const_cast` | 无等价物 | | Rust 在安全代码中无法将 `&` 转换为 `&mut`。使用 `Cell`/`RefCell` 实现内部可变性 |
| `reinterpret_cast` | `std::mem::transmute` | **`unsafe`** | 重新解释位模式。几乎总是错误的 —— 优先使用 `from_le_bytes()` 等 |

```rust
// Rust 等价物：

// 1. 数值转换 —— 优先使用 From/Into 而不是 `as`
let widened: u32 = 42_u8.into();             // 不可失败的 widening —— 总是优先
let truncated = 300_u16 as u8;                // ⚠ 回绕到 44！静默数据丢失
let checked: Result<u8, _> = 300_u16.try_into(); // Err —— 安全的可失败转换

// 2. 向下转换：enum（首选）或 Any（需要类型擦除时使用）
use std::any::Any;

fn handle_any(val: &dyn Any) {
    if let Some(s) = val.downcast_ref::<String>() {
        println!("Got string: {s}");
    } else if let Some(n) = val.downcast_ref::<i32>() {
        println!("Got int: {n}");
    }
}

// 3. "const_cast" → 内部可变性（无需 unsafe）
use std::cell::Cell;
struct Sensor {
    read_count: Cell<u32>,  // 通过 &self 可变
}
impl Sensor {
    fn read(&self) -> f64 {
        self.read_count.set(self.read_count.get() + 1); // &self，不是 &mut self
        42.0
    }
}

// 4. reinterpret_cast → transmute（几乎从不需要）
// 优先使用安全的替代方案：
let bytes: [u8; 4] = 0x12345678_u32.to_ne_bytes();  // ✅ 安全
let val = u32::from_ne_bytes(bytes);                   // ✅ 安全
// unsafe { std::mem::transmute::<u32, [u8; 4]>(val) } // ❌ 避免
```

> **指南**：在惯用 Rust 中，`as` 应该是罕见的（对 widening 使用 `From`/`Into`，对 narrowing 使用 `TryFrom`/`TryInto`），`transmute` 应该是例外的，而 `const_cast` 没有等价物，因为内部可变性类型使其不必要。

---

### 预处理器 → `cfg`、功能标志和 `macro_rules!`

C++ 严重依赖预处理器进行条件编译、常量和代码生成。Rust 用一流的语言功能替换所有这些。

#### `#define` 常量 → `const` 或 `const fn`

```cpp
// C++
#define MAX_RETRIES 5
#define BUFFER_SIZE (1024 * 64)
#define SQUARE(x) ((x) * (x))  // 宏 —— 文本替换，无类型安全
```

```rust
// Rust —— 类型安全、作用域、无文本替换
const MAX_RETRIES: u32 = 5;
const BUFFER_SIZE: usize = 1024 * 64;
const fn square(x: u32) -> u32 { x * x }  // 编译时计算

// 可在 const 上下文中使用：
const AREA: u32 = square(12);  // 编译时计算
static BUFFER: [u8; BUFFER_SIZE] = [0; BUFFER_SIZE];
```

#### `#ifdef` / `#if` → `#[cfg()]` 和 `cfg!()`

```cpp
// C++
#ifdef DEBUG
    log_verbose("Step 1 complete");
#endif

#if defined(LINUX) && !defined(ARM)
    use_x86_path();
#else
    use_generic_path();
#endif
```

```rust
// Rust —— 基于属性的条件编译
#[cfg(debug_assertions)]
fn log_verbose(msg: &str) { eprintln!("[VERBOSE] {msg}"); }

#[cfg(not(debug_assertions))]
fn log_verbose(_msg: &str) { /* release 时编译为空 */ }

// 组合条件：
#[cfg(all(target_os = "linux", target_arch = "x86_64"))]
fn use_x86_path() { /* ... */ }

#[cfg(not(all(target_os = "linux", target_arch = "x86_64")))]
fn use_generic_path() { /* ... */ }

// 运行时检查（条件仍是编译时的，但可在表达式中使用）：
if cfg!(target_os = "windows") {
    println!("Running on Windows");
}
```

#### `Cargo.toml` 中的功能标志

```toml
// Cargo.toml —— 替换 #ifdef FEATURE_FOO
[features]
default = ["json"]
json = ["dep:serde_json"]       # 可选依赖
verbose-logging = []            # 无额外依赖的标志
gpu-support = ["dep:cuda-sys"]  # 可选 GPU 支持
```

```rust
// 基于功能标志的条件代码：
#[cfg(feature = "json")]
pub fn parse_config(data: &str) -> Result<Config, Error> {
    serde_json::from_str(data).map_err(Error::from)
}

#[cfg(feature = "verbose-logging")]
macro_rules! verbose {
    ($($arg:tt)*) => { eprintln!("[VERBOSE] {}", format!($($arg)*)); }
}
#[cfg(not(feature = "verbose-logging"))]
macro_rules! verbose {
    ($($arg:tt)*) => { }; // 编译为空
}
```

#### `#define MACRO(x)` → `macro_rules!`

```cpp
// C++ —— 文本替换，众所周知的容易出错
#define DIAG_CHECK(cond, msg) \
    do { if (!(cond)) { log_error(msg); return false; } } while(0)
```

```rust
// Rust —— 卫生、类型检查、操作语法树
macro_rules! diag_check {
    ($cond:expr, $msg:expr) => {
        if !($cond) {
            log_error($msg);
            return Err(DiagError::CheckFailed($msg.to_string()));
        }
    };
}

fn run_test() -> Result<(), DiagError> {
    diag_check!(temperature < 85.0, "GPU too hot");
    diag_check!(voltage > 0.8, "Rail voltage too low");
    Ok(())
}
```

| C++ 预处理器 | Rust 等价物 | 优势 |
|-----------------|----------------|-----------|
| `#define PI 3.14` | `const PI: f64 = 3.14;` | 类型化、作用域、调试器可见 |
| `#define MAX(a,b) ((a)>(b)?(a):(b))` | `macro_rules!` 或泛型 `fn max<T: Ord>` | 无双倍求值 bug |
| `#ifdef DEBUG` | `#[cfg(debug_assertions)]` | 编译器检查，无拼写错误风险 |
| `#ifdef FEATURE_X` | `#[cfg(feature = "x")]` | Cargo 管理功能；依赖感知 |
| `#include "header.h"` | `mod module;` + `use module::Item;` | 无包含保护、无循环包含 |
| `#pragma once` | 不需要 | 每个 `.rs` 文件是模块 —— 仅包含一次 |

---

### 头文件和 `#include` → 模块和 `use`

在 C++ 中，编译模型围绕文本包含：

```cpp
// widget.h —— 每个使用 Widget 的翻译单元包含这个
#pragma once
#include <string>
#include <vector>

class Widget {
public:
    Widget(std::string name);
    void activate();
private:
    std::string name_;
    std::vector<int> data_;
};
```

```cpp
// widget.cpp —— 独立定义
#include "widget.h"
Widget::Widget(std::string name) : name_(std::move(name)) {}
void Widget::activate() { /* ... */ }
```

在 Rust 中，**没有头文件、没有前向声明、无包含保护**：

```rust
// src/widget.rs —— 声明和定义在一个文件中
pub struct Widget {
    name: String,         // 默认私有
    data: Vec<i32>,
}

impl Widget {
    pub fn new(name: String) -> Self {
        Widget { name, data: Vec::new() }
    }
    pub fn activate(&self) { /* ... */ }
}
```

```rust
// src/main.rs —— 通过模块路径导入
mod widget;  // 告诉编译器包含 src/widget.rs
use widget::Widget;

fn main() {
    let w = Widget::new("sensor".to_string());
    w.activate();
}
```

| C++ | Rust | 为什么更好 |
|-----|------|-----------------|
| `#include "foo.h"` | 父级中的 `mod foo;` + `use foo::Item;` | 无文本包含、无 ODR 违规 |
| `#pragma once` / 包含保护 | 不需要 | 每个 `.rs` 文件是模块 —— 编译一次 |
| 前向声明 | 不需要 | 编译器看到整个 crate；顺序无关 |
| `class Foo;` (不完整类型) | 不需要 | 无独立的声明/定义分离 |
| 每个类的 `.h` + `.cpp` | 单个 `.rs` 文件 | 无声明/定义不匹配 bug |
| `using namespace std;` | `use std::collections::HashMap;` | 总是显式 —— 无全局命名空间污染 |
| 嵌套 `namespace a::b` | 嵌套 `mod a { mod b { } }` 或 `a/b.rs` | 文件系统镜像模块树 |

---

### `friend` 和访问控制 → 模块可见性

C++ 使用 `friend` 授予特定类或函数访问私有成员的权限。Rust 没有 `friend` 关键字 —— **可见性是模块作用域的**：

```cpp
// C++
class Engine {
    friend class Car;   // Car 可以访问私有成员
    int rpm_;
    void set_rpm(int r) { rpm_ = r; }
public:
    int rpm() const { return rpm_; }
};
```

```rust
// Rust —— 同一模块内的项可以访问所有字段，无需 `friend`
mod vehicle {
    pub struct Engine {
        rpm: u32,  // 对模块私有（不是对结构体私有！）
    }

    impl Engine {
        pub fn new() -> Self { Engine { rpm: 0 } }
        pub fn rpm(&self) -> u32 { self.rpm }
    }

    pub struct Car {
        engine: Engine,
    }

    impl Car {
        pub fn new() -> Self { Car { engine: Engine::new() } }
        pub fn accelerate(&mut self) {
            self.engine.rpm = 3000; // ✅ 同一模块 —— 直接字段访问
        }
        pub fn rpm(&self) -> u32 {
            self.engine.rpm  // ✅ 同一模块 —— 可以读取私有字段
        }
    }
}

fn main() {
    let mut car = vehicle::Car::new();
    car.accelerate();
    // car.engine.rpm = 9000;  // ❌ 编译错误：`engine` 是私有的
    println!("RPM: {}", car.rpm()); // ✅ Car 上的公共方法
}
```

| C++ 访问 | Rust 等价物 | 作用域 |
|-----------|----------------|-------|
| `private` | （默认，无关键字） | 仅在同一模块内可访问 |
| `protected` | 无直接等价物 | 使用 `pub(super)` 访问父模块 |
| `public` | `pub` | 到处可访问 |
| `friend class Foo` | 将 `Foo` 放在同一模块 | 模块级可见性替换 friend |
| — | `pub(crate)` | crate 内可见但对外部依赖不可见 |
| — | `pub(super)` | 仅对父模块可见 |
| — | `pub(in crate::path)` | 在特定模块子树内可见 |

> **关键洞察**：C++ 可见性是每类的。Rust 可见性是每模块的。这意味着你通过选择哪些类型生活在同一模块中来控制访问 —— 同位置的类型可以完全访问彼此的私有字段。

---

### `volatile` → 原子操作和 `read_volatile`/`write_volatile`

在 C++ 中，`volatile` 告诉编译器不要优化掉读/写 —— 通常用于内存映射硬件寄存器。**Rust 没有 `volatile` 关键字。**

```cpp
// C++: 硬件寄存器用 volatile
volatile uint32_t* const GPIO_REG = reinterpret_cast<volatile uint32_t*>(0x4002'0000);
*GPIO_REG = 0x01;              // 写入不被优化掉
uint32_t val = *GPIO_REG;     // 读取不被优化掉
```

```rust
// Rust: 显式 volatile 操作 —— 仅在 unsafe 代码中
use std::ptr;

const GPIO_REG: *mut u32 = 0x4002_0000 as *mut u32;

// 安全：GPIO_REG 是有效的内存映射 I/O 地址。
unsafe {
    ptr::write_volatile(GPIO_REG, 0x01);   // 写入不被优化掉
    let val = ptr::read_volatile(GPIO_REG); // 读取不被优化掉
}
```

对于**并发共享状态**（另一个常见的 C++ `volatile` 用例），Rust 使用原子操作：

```cpp
// C++: volatile 不足以用于线程安全（常见错误！）
volatile bool stop_flag = false;  // ❌ 数据竞争 —— C++11+ 中的 UB

// 正确的 C++:
std::atomic<bool> stop_flag{false};
```

```rust
// Rust: 原子操作是跨线程共享可变状态的唯一方式
use std::sync::atomic::{AtomicBool, Ordering};

static STOP_FLAG: AtomicBool = AtomicBool::new(false);

// 从另一个线程：
STOP_FLAG.store(true, Ordering::Release);

// 检查：
if STOP_FLAG.load(Ordering::Acquire) {
    println!("Stopping");
}
```

| C++ 用例 | Rust 等价物 | 注释 |
|-----------|----------------|-------|
| `volatile` 用于硬件寄存器 | `ptr::read_volatile` / `ptr::write_volatile` | 需要 `unsafe` —— 适合 MMIO |
| `volatile` 用于线程信号 | `AtomicBool` / `AtomicU32` 等 | C++ `volatile` 对此也是错误的！ |
| `std::atomic<T>` | `std::sync::atomic::AtomicT` | 相同语义、相同的顺序 |
| `std::atomic<T>::load(memory_order_acquire)` | `AtomicT::load(Ordering::Acquire)` | 1:1 映射 |

---

### `static` 变量 → `static`、`const`、`LazyLock`、`OnceLock`

#### 基础 `static` 和 `const`

```cpp
// C++
const int MAX_RETRIES = 5;                    // 编译时常量
static std::string CONFIG_PATH = "/etc/app";  // 静态初始化 —— 顺序未定义！
```

```rust
// Rust
const MAX_RETRIES: u32 = 5;                   // 编译时常量，内联
static CONFIG_PATH: &str = "/etc/app";         // 'static 生命周期，固定地址
```

#### 静态初始化顺序陷阱

C++ 有一个众所周知的问题：不同翻译单元中的全局构造函数按**未指定顺序**执行。Rust 完全避免了这个问题 —— `static` 值必须是编译时常量（无构造函数）。

对于运行时初始化的全局变量，使用 `LazyLock`（Rust 1.80+）或 `OnceLock`：

```rust
use std::sync::LazyLock;

// 等价于 C++ `static std::regex` —— 首次访问时初始化，线程安全
static CONFIG_REGEX: LazyLock<regex::Regex> = LazyLock::new(|| {
    regex::Regex::new(r"^[a-z]+_diag$").expect("invalid regex")
});

fn is_valid_diag(name: &str) -> bool {
    CONFIG_REGEX.is_match(name)  // 首次调用初始化；后续调用快速
}
```

```rust
use std::sync::OnceLock;

// OnceLock: 初始化一次，可以从运行时数据设置
static DB_CONN: OnceLock<String> = OnceLock::new();

fn init_db(connection_string: &str) {
    DB_CONN.set(connection_string.to_string())
        .expect("DB_CONN already initialized");
}

fn get_db() -> &'static str {
    DB_CONN.get().expect("DB not initialized")
}
```

| C++ | Rust | 注释 |
|-----|------|-------|
| `const int X = 5;` | `const X: i32 = 5;` | 两者都是编译时。Rust 需要类型注解 |
| `constexpr int X = 5;` | `const X: i32 = 5;` | Rust `const` 总是 constexpr |
| `static int count = 0;` (文件作用域) | `static COUNT: AtomicI32 = AtomicI32::new(0);` | 可变 static 需要 `unsafe` 或原子操作 |
| `static std::string s = "hi";` | `static S: &str = "hi";` 或 `LazyLock<String>` | 简单情况无运行时构造函数 |
| `static MyObj obj;` (复杂初始化) | `static OBJ: LazyLock<MyObj> = LazyLock::new(\|\| { ... });` | 线程安全、惰性、无初始化顺序问题 |
| `thread_local` | `thread_local! { static X: Cell<u32> = Cell::new(0); }` | 相同语义 |

---

### `constexpr` → `const fn`

C++ `constexpr` 标记函数和变量用于编译时计算。Rust 使用 `const fn` 和 `const` 用于相同目的：

```cpp
// C++
constexpr int factorial(int n) {
    return n <= 1 ? 1 : n * factorial(n - 1);
}
constexpr int val = factorial(5);  // 编译时计算 → 120
```

```rust
// Rust
const fn factorial(n: u32) -> u32 {
    if n <= 1 { 1 } else { n * factorial(n - 1) }
}
const VAL: u32 = factorial(5);  // 编译时计算 → 120

// 也适用于数组大小和 match 模式：
const LOOKUP: [u32; 5] = [factorial(1), factorial(2), factorial(3),
                           factorial(4), factorial(5)];
```

| C++ | Rust | 注释 |
|-----|------|-------|
| `constexpr int f()` | `const fn f() -> i32` | 相同意图 —— 编译时可计算 |
| `constexpr` 变量 | `const` 变量 | Rust `const` 总是编译时 |
| `consteval` (C++20) | 无等价物 | `const fn` 也可以在运行时运行 |
| `if constexpr` (C++17) | 无等价物（使用 `cfg!` 或泛型） | trait 特化填充某些用例 |
| `constinit` (C++20) | 带常量初始化器的 `static` | Rust `static` 默认必须是常量初始化的 |

> **`const fn` 的当前限制**（Rust 1.82 已稳定）：
> - 无 trait 方法（不能在 const 上下文中调用 `Vec` 的 `.len()`）
> - 无堆分配（`Box::new`、`Vec::new` 不是 const）
> - ~~无浮点算术~~ —— **Rust 1.82 已稳定**
> - 不能使用 `for` 循环（使用递归或带手动索引的 `while`）

---

### SFINAE 和 `enable_if` → Trait 约束和 `where` 子句

在 C++ 中，SFINAE（替换失败不是错误）是条件泛型编程背后的机制。它功能强大但以难以阅读而闻名。Rust 完全用**trait 约束**替换它：

```cpp
// C++: 基于 SFINAE 的条件函数（C++20 之前）
template<typename T,
         std::enable_if_t<std::is_integral_v<T>, int> = 0>
T double_it(T val) { return val * 2; }

template<typename T,
         std::enable_if_t<std::is_floating_point_v<T>, int> = 0>
T double_it(T val) { return val * 2.0; }

// C++20 concepts —— 更清晰但仍然冗长：
template<std::integral T>
T double_it(T val) { return val * 2; }
```

```rust
// Rust: trait 约束 —— 可读、可组合、出色的错误消息
use std::ops::Mul;

fn double_it<T: Mul<Output = T> + From<u8>>(val: T) -> T {
    val * T::from(2)
}

// 或带 where 子句的复杂约束：
fn process<T>(val: T) -> String
where
    T: std::fmt::Display + Clone + Send,
{
    format!("Processing: {}", val)
}

// 通过独立的 impl 的条件行为（替换 SFINAE 重载）：
trait Describable {
    fn describe(&self) -> String;
}

impl Describable for u32 {
    fn describe(&self) -> String { format!("integer: {self}") }
}

impl Describable for f64 {
    fn describe(&self) -> String { format!("float: {self:.2}") }
}
```

| C++ 模板元编程 | Rust 等价物 | 可读性 |
|-----------------------------|----------------|-------------|
| `std::enable_if_t<cond>` | `where T: Trait` | 🟢 清晰的英语 |
| `std::is_integral_v<T>` | 数值 trait 或特定类型的约束 | 🟢 无 `_v` / `_t` 后缀 |
| SFINAE 重载集 | 独立的 `impl Trait for ConcreteType` 块 | 🟢 每个 impl 独立 |
| `if constexpr (std::is_same_v<T, int>)` | 通过 trait impl 特化 | 🟢 编译时分发 |
| C++20 `concept` | `trait` | 🟢 几乎相同的意图 |
| `requires` 子句 | `where` 子句 | 🟢 相同位置、相似语法 |
| 编译失败深入模板内部 | 编译失败在调用点带 trait 不匹配 | 🟢 无 200 行错误级联 |

> **关键洞察**：C++ concepts（C++20）是最接近 Rust traits 的。如果你熟悉 C++20 concepts，将 Rust traits 视为 concepts 自 1.0 以来就是一流语言特性，带有一致的实现模型（trait impl）而不是 duck typing。

---

### `std::function` → 函数指针、`impl Fn` 和 `Box<dyn Fn>`

C++ `std::function<R(Args...)>` 是类型擦除的可调用对象。Rust 有三个选项，每个都有不同的权衡：

```cpp
// C++: 万能（堆分配，类型擦除）
#include <functional>
std::function<int(int)> make_adder(int n) {
    return [n](int x) { return x + n; };
}
```

```rust
// Rust 选项 1: fn 指针 —— 简单、无捕获、无分配
fn add_one(x: i32) -> i32 { x + 1 }
let f: fn(i32) -> i32 = add_one;
println!("{}", f(5)); // 6

// Rust 选项 2: impl Fn —— 单态化、零开销、可以捕获
fn apply(val: i32, f: impl Fn(i32) -> i32) -> i32 { f(val) }
let n = 10;
let result = apply(5, |x| x + n);  // 闭包捕获 `n`

// Rust 选项 3: Box<dyn Fn> —— 类型擦除、堆分配（类似 std::function）
fn make_adder(n: i32) -> Box<dyn Fn(i32) -> i32> {
    Box::new(move |x| x + n)
}
let adder = make_adder(10);
println!("{}", adder(5));  // 15

// 存储异质可调用对象（类似 vector<function<int(int)>>）：
let callbacks: Vec<Box<dyn Fn(i32) -> i32>> = vec![
    Box::new(|x| x + 1),
    Box::new(|x| x * 2),
    Box::new(make_adder(100)),
];
for cb in &callbacks {
    println!("{}", cb(5));  // 6, 10, 105
}
```

| 何时使用 | C++ 等价物 | Rust 选择 |
|------------|---------------|-------------|
| 顶层函数、无捕获 | 函数指针 | `fn(Args) -> Ret` |
| 接受可调用对象的泛型函数 | 模板参数 | `impl Fn(Args) -> Ret`（静态分发） |
| 泛型中的 trait 约束 | `template<typename F>` | `F: Fn(Args) -> Ret` |
| 存储的可调用对象，类型擦除 | `std::function<R(Args)>` | `Box<dyn Fn(Args) -> Ret>` |
| 可变状态的回调 | 带可变 lambda 的 `std::function` | `Box<dyn FnMut(Args) -> Ret>` |
| 一次性回调（消耗） | （移动）`std::function` | `Box<dyn FnOnce(Args) -> Ret>` |

> **性能注意**：`impl Fn` 零开销（单态化，类似 C++ 模板）。`Box<dyn Fn>` 与 `std::function` 有相同的开销（vtable + 堆分配）。优先使用 `impl Fn`，除非你需要存储异质可调用对象。

---

### 容器映射：C++ STL → Rust `std::collections`

| C++ STL 容器 | Rust 等价物 | 注释 |
|------------------|----------------|-------|
| `std::vector<T>` | `Vec<T>` | 几乎相同的 API。Rust 默认检查边界 |
| `std::array<T, N>` | `[T; N]` | 栈分配固定大小数组 |
| `std::deque<T>` | `std::collections::VecDeque<T>` | 环形缓冲区。两端高效 push/pop |
| `std::list<T>` | `std::collections::LinkedList<T>` | Rust 中很少使用 —— `Vec` 几乎总是更快 |
| `std::forward_list<T>` | 无等价物 | 使用 `Vec` 或 `VecDeque` |
| `std::unordered_map<K, V>` | `std::collections::HashMap<K, V>` | 默认使用 `SipHash`（DoS 抵抗） |
| `std::map<K, V>` | `std::collections::BTreeMap<K, V>` | B 树；键已排序；需要 `K: Ord` |
| `std::unordered_set<T>` | `std::collections::HashSet<T>` | 需要 `T: Hash + Eq` |
| `std::set<T>` | `std::collections::BTreeSet<T>` | 排序集合；需要 `T: Ord` |
| `std::priority_queue<T>` | `std::collections::BinaryHeap<T>` | 默认最大堆（与 C++ 相同） |
| `std::stack<T>` | 带 `.push()` / `.pop()` 的 `Vec<T>` | 无需独立的栈类型 |
| `std::queue<T>` | 带 `.push_back()` / `.pop_front()` 的 `VecDeque<T>` | 无需独立的队列类型 |
| `std::string` | `String` | 保证 UTF-8，非 null 终止 |
| `std::string_view` | `&str` | 借用的 UTF-8 切片 |
| `std::span<T>` (C++20) | `&[T]` / `&mut [T]` | Rust 切片自 1.0 以来就是一等类型 |
| `std::tuple<A, B, C>` | `(A, B, C)` | 一等语法，可解构 |
| `std::pair<A, B>` | `(A, B)` | 只是二元组 |
| `std::bitset<N>` | 无标准库等价物 | 使用 `bitvec` crate 或 `[u8; N/8]` |

**关键区别**：
- Rust 的 `HashMap`/`HashSet` 需要 `K: Hash + Eq` —— 编译器在类型级别强制执行此要求，与 C++ 不同，在 C++ 中使用不可哈希的键会在 STL 深处给出模板错误
- `Vec` 索引（`v[i]`）默认在越界时 panic。使用 `.get(i)` 获取 `Option<&T>` 或迭代器完全避免边界检查
- 无 `std::multimap` 或 `std::multiset` —— 使用 `HashMap<K, Vec<V>>` 或 `BTreeMap<K, Vec<V>>`

---

### 异常安全 → Panic 安全

C++ 定义了三个级别的异常安全（Abrahams 保证）：

| C++ 级别 | 含义 | Rust 等价物 |
|----------|---------|----------------|
| **No-throw** | 函数从不抛出 | 函数从不 panic（返回 `Result`） |
| **Strong**（提交或回滚） | 如果抛出，状态不变 | 所有权模型使其自然 —— 如果 `?` 提前返回，部分构建的值被 drop |
| **Basic** | 如果抛出，不变量保持 | Rust 的默认 —— `Drop` 运行，无泄漏 |

#### Rust 所有权模型如何帮助

```rust
// 免费提供强保证 —— 如果 file.write() 失败，config 不变
fn update_config(config: &mut Config, path: &str) -> Result<(), Error> {
    let new_data = fetch_from_network()?; // Err → 提前返回，config 未触及
    let validated = validate(new_data)?;   // Err → 提前返回，config 未触及
    *config = validated;                   // 仅在成功时到达（提交）
    Ok(())
}
```

在 C++ 中，实现强保证需要手动回滚或复制 - 交换习语。在 Rust 中，`?` 传播为大多数代码默认提供强保证。

#### `catch_unwind` —— Rust 的 `catch(...)` 等价物

```rust
use std::panic;

// 捕获 panic（类似 C++ 中的 catch(...)）—— 很少需要
let result = panic::catch_unwind(|| {
    // 可能 panic 的代码
    let v = vec![1, 2, 3];
    v[10]  // Panic！（索引越界）
});

match result {
    Ok(val) => println!("Got: {val}"),
    Err(_) => eprintln!("Caught a panic — cleaned up"),
}
```

#### `UnwindSafe` —— 标记类型 panic 安全

```rust
use std::panic::UnwindSafe;

// &mut 背后的类型默认 NOT UnwindSafe —— panic 可能使它们处于部分修改状态
fn safe_execute<F: FnOnce() + UnwindSafe>(f: F) {
    let _ = std::panic::catch_unwind(f);
}

// 使用 AssertUnwindSafe 覆盖当你审核代码后：
use std::panic::AssertUnwindSafe;
let mut data = vec![1, 2, 3];
let _ = std::panic::catch_unwind(AssertUnwindSafe(|| {
    data.push(4);
}));
```

| C++ 异常模式 | Rust 等价物 |
|-----------------------|-----------------|
| `throw MyException()` | `return Err(MyError::...)`（首选）或 `panic!("...")` |
| `try { } catch (const E& e)` | `match result { Ok(v) => ..., Err(e) => ... }` 或 `?` |
| `catch (...)` | `std::panic::catch_unwind(...)` |
| `noexcept` | `-> Result<T, E>`（错误是值，不是异常） |
| 栈展开中的 RAII 清理 | `Drop::drop()` 在 panic 展开期间运行 |
| `std::uncaught_exceptions()` | `std::thread::panicking()` |
| `-fno-exceptions` 编译标志 | `Cargo.toml` [profile] 中的 `panic = "abort"` |

> **底线**：在 Rust 中，大多数代码使用 `Result<T, E>` 而不是异常，使错误路径显式且可组合。`panic!` 预留给 bug（如 `assert!` 失败），不是常规错误。这意味着"异常安全"主要不是问题 —— 所有权系统自动处理清理。

---

## C++ 到 Rust 迁移模式

### 快速参考：C++ → Rust 习语映射

| **C++ 模式** | **Rust 习语** | **注释** |
|----------------|---------------|----------|
| `class Derived : public Base` | `enum Variant { A {...}, B {...} }` | 对封闭集优先使用 enum |
| `virtual void method() = 0` | `trait MyTrait { fn method(&self); }` | 对开放/可扩展接口使用 |
| `dynamic_cast<Derived*>(ptr)` | `match value { Variant::A(data) => ..., }` | 穷尽、无运行时失败 |
| `vector<unique_ptr<Base>>` | `Vec<Box<dyn Trait>>` | 仅当真正多态时 |
| `shared_ptr<T>` | `Rc<T>` 或 `Arc<T>` | 优先使用 `Box<T>` 或自有值 |
| `enable_shared_from_this<T>` | Arena 模式（`Vec<T>` + 索引） | 完全消除引用循环 |
| 每个类中的 `Base* m_pFramework` | `fn execute(&mut self, ctx: &mut Context)` | 传递上下文，不要存储指针 |
| `try { } catch (...) { }` | `match result { Ok(v) => ..., Err(e) => ... }` | 或使用 `?` 进行传播 |
| `std::optional<T>` | `Option<T>` | 需要 `match`，不能忘记 None |
| `const std::string&` 参数 | `&str` 参数 | 接受 `String` 和 `&str` |
| `enum class Foo { A, B, C }` | `enum Foo { A, B, C }` | Rust enum 也可以携带数据 |
| `auto x = std::move(obj)` | `let x = obj;` | Move 是默认，无需 `std::move` |
| CMake + make + lint | `cargo build / test / clippy / fmt` | 一个工具搞定一切 |

### 迁移策略
1. **从数据类型开始**：首先翻译结构体和 enum —— 这迫使你思考所有权
2. **将工厂转换为 enum**：如果工厂创建不同的派生类型，它应该可能是 `enum` + `match`
3. **将上帝对象转换为组合结构体**：将相关字段分组为专注的结构体
4. **将指针转换为借用**：将 `Base*` 存储指针转换为 `'a T` 生命周期限制的借用
5. **谨慎使用 `Box<dyn Trait>`**：仅用于插件系统和测试 mocking
6. **让编译器指导你**：Rust 的错误消息非常出色 —— 仔细阅读它们

---
