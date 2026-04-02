## Rust 宏：从预处理器到元编程

> **你将学到什么：** Rust 宏如何工作、何时使用它们代替函数或泛型，以及它们如何替换 C/C++ 预处理器。在本章结束时，你可以编写自己的 `macro_rules!` 宏并理解 `#[derive(Debug)]` 在底层做什么。

宏是你在 Rust 中最早接触的东西之一（第一行的 `println!("hello")`），但也是大多数课程最后解释的东西之一。本章修复这个问题。

### 为什么宏存在

函数和泛型处理 Rust 中的大多数代码重用。宏填补类型系统无法触及的空白：

| 需求 | 函数/泛型？ | 宏？ | 为什么 |
|------|-------------------|--------|-----|
| 计算值 | ✅ `fn max<T: Ord>(a: T, b: T) -> T` | — | 类型系统处理它 |
| 接受可变数量参数 | ❌ Rust 没有变参函数 | ✅ `println!("{} {}", a, b)` | 宏接受任意数量的 token |
| 为多个类型生成重复的 `impl` 块 | ❌ 仅用泛型无法完成 | ✅ `macro_rules!` | 宏在编译时生成代码 |
| 在编译时运行代码 | ❌ `const fn` 有限制 | ✅ 过程宏 | 完整的 Rust 代码在编译时运行 |
| 条件包含代码 | ❌ | ✅ `#[cfg(...)]` | 属性宏控制编译 |

如果你来自 C/C++，将宏视为*预处理器的唯一正确替换* —— 除了它们操作语法树而不是原始文本，所以它们是卫生的（无意外的名称冲突）且类型感知的。

> **对于 C 开发者：** Rust 宏完全替换 `#define`。没有文本预处理器。请参阅 [ch18](ch18-cpp-rust-semantic-deep-dives.md) 获取完整的预处理器 → Rust 映射。

---

## 使用 `macro_rules!` 的声明式宏

声明式宏（也称为"示例宏"）是 Rust 最常见的宏形式。它们使用语法模式匹配，类似于值上的 `match`。

### 基础语法

```rust
macro_rules! say_hello {
    () => {
        println!("Hello!");
    };
}

fn main() {
    say_hello!();  // 展开为：println!("Hello!");
}
```

名称后的 `!` 是告诉你（和编译器）这是宏调用的内容。

### 带参数的模式匹配

宏使用片段说明符在*token 树*上匹配：

```rust
macro_rules! greet {
    // 模式 1：无参数
    () => {
        println!("Hello, world!");
    };
    // 模式 2：一个表达式参数
    ($name:expr) => {
        println!("Hello, {}!", $name);
    };
}

fn main() {
    greet!();           // "Hello, world!"
    greet!("Rust");     // "Hello, Rust!"
}
```

#### 片段说明符参考

| 说明符 | 匹配 | 示例 |
|-----------|---------|---------|
| `$x:expr` | 任何表达式 | `42`、`a + b`、`foo()` |
| `$x:ty` | 类型 | `i32`、`Vec<String>`、`&str` |
| `$x:ident` | 标识符 | `foo`、`my_var` |
| `$x:pat` | 模式 | `Some(x)`、`_`、`(a, b)` |
| `$x:stmt` | 语句 | `let x = 5;` |
| `$x:block` | 块 | `{ println!("hi"); 42 }` |
| `$x:literal` | 字面量 | `42`、`"hello"`、`true` |
| `$x:tt` | 单个 token 树 | 任何内容 —— 通配符 |
| `$x:item` | 项（fn、struct、impl 等） | `fn foo() {}` |

### 重复 —— 杀手级功能

C/C++ 宏不能循环。Rust 宏可以重复模式：

```rust
macro_rules! make_vec {
    // 匹配零个或多个逗号分隔的表达式
    ( $( $element:expr ),* ) => {
        {
            let mut v = Vec::new();
            $( v.push($element); )*  // 对每个匹配的元素重复
            v
        }
    };
}

fn main() {
    let v = make_vec![1, 2, 3, 4, 5];
    println!("{v:?}");  // [1, 2, 3, 4, 5]
}
```

`$( ... ),*` 语法意味着"匹配零个或多个此模式，用逗号分隔。"展开式中的 `$( ... )*` 对每个匹配重复主体一次。

> **这正是 `vec![]` 在标准库中的实现方式。** 实际源代码是：
> ```rust
> macro_rules! vec {
>     () => { Vec::new() };
>     ($elem:expr; $n:expr) => { vec::from_elem($elem, $n) };
>     ($($x:expr),+ $(,)?) => { <[_]>::into_vec(Box::new([$($x),+])) };
> }
> ```
> 末尾的 `$(,)?` 允许尾随逗号。

#### 重复运算符

| 运算符 | 含义 | 示例 |
|----------|---------|---------|
| `$( ... )*` | 零个或多个 | `vec![]`、`vec![1]`、`vec![1, 2, 3]` |
| `$( ... )+` | 一个或多个 | 至少需要一个元素 |
| `$( ... )?` | 零个或一个 | 可选元素 |

### 实用示例：`hashmap!` 构造器

标准库有 `vec![]` 但没有 `hashmap!{}`。让我们构建一个：

```rust
macro_rules! hashmap {
    ( $( $key:expr => $value:expr ),* $(,)? ) => {
        {
            let mut map = std::collections::HashMap::new();
            $( map.insert($key, $value); )*
            map
        }
    };
}

fn main() {
    let scores = hashmap! {
        "Alice" => 95,
        "Bob" => 87,
        "Carol" => 92,  // 尾随逗号 OK，多亏了 $(,)?
    };
    println!("{scores:?}");
}
```

### 实用示例：诊断检查宏

嵌入式/诊断代码中常见的模式 —— 检查条件并返回错误：

```rust
use thiserror::Error;

#[derive(Error, Debug)]
enum DiagError {
    #[error("Check failed: {0}")]
    CheckFailed(String),
}

macro_rules! diag_check {
    ($cond:expr, $msg:expr) => {
        if !($cond) {
            return Err(DiagError::CheckFailed($msg.to_string()));
        }
    };
}

fn run_diagnostics(temp: f64, voltage: f64) -> Result<(), DiagError> {
    diag_check!(temp < 85.0, "GPU too hot");
    diag_check!(voltage > 0.8, "Rail voltage too low");
    diag_check!(voltage < 1.5, "Rail voltage too high");
    println!("All checks passed");
    Ok(())
}
```

> **C/C++ 对比：**
> ```c
> // C 预处理器 —— 文本替换，无类型安全，无卫生性
> #define DIAG_CHECK(cond, msg) \
>     do { if (!(cond)) { log_error(msg); return -1; } } while(0)
> ```
> Rust 版本返回适当的 `Result` 类型，无双倍求值风险，编译器检查 `$cond` 实际上是 `bool` 表达式。

### 卫生性：为什么 Rust 宏是安全的

C/C++ 宏 bug 通常来自名称冲突：

```c
// C: 危险 —— `x` 可能遮蔽调用者的 `x`
#define SQUARE(x) ((x) * (x))
int x = 5;
int result = SQUARE(x++);  // UB: x 被递增两次！
```

Rust 宏是**卫生的** —— 宏内创建的变量不会泄露：

```rust
macro_rules! make_x {
    () => {
        let x = 42;  // 此 `x` 作用域限于宏展开
    };
}

fn main() {
    let x = 10;
    make_x!();
    println!("{x}");  // 打印 10，不是 42 —— 卫生性防止冲突
}
```

宏的 `x` 和调用者的 `x` 被编译器视为不同的变量，即使它们名称相同。**这在 C 预处理器中是不可能的。**

---

## 常见标准库宏

你从第 1 章就开始使用这些 —— 这是它们实际做什么：

| 宏 | 它做什么 | 展开为（简化） |
|-------|-------------|------------------------|
| `println!("{}", x)` | 格式化并打印到 stdout + 换行 | `std::io::_print(format_args!(...))` |
| `eprintln!("{}", x)` | 打印到 stderr + 换行 | 同但到 stderr |
| `format!("{}", x)` | 格式化为 `String` | 分配并返回 `String` |
| `vec![1, 2, 3]` | 带元素创建 `Vec` | `Vec::from([1, 2, 3])`（大约） |
| `todo!()` | 标记未完成代码 | `panic!("not yet implemented")` |
| `unimplemented!()` | 标记故意未实现的代码 | `panic!("not implemented")` |
| `unreachable!()` | 标记编译器无法证明不可达的代码 | `panic!("unreachable")` |
| `assert!(cond)` | 如果条件为 false 则 panic | `if !cond { panic!(...) }` |
| `assert_eq!(a, b)` | 如果值不相等则 panic | 失败时显示两个值 |
| `dbg!(expr)` | 打印表达式 + 值到 stderr，返回值 | `eprintln!("[file:line] expr = {:#?}", &expr); expr` |
| `include_str!("file.txt")` | 编译时将文件内容嵌入为 `&str` | 编译期间读取文件 |
| `include_bytes!("data.bin")` | 编译时将文件内容嵌入为 `&[u8]` | 编译期间读取文件 |
| `cfg!(condition)` | 编译时条件作为 `bool` | 基于目标的 `true` 或 `false` |
| `env!("VAR")` | 编译时读取环境变量 | 如果未设置则编译失败 |
| `concat!("a", "b")` | 编译时连接字面量 | `"ab"` |

### `dbg!` —— 你每天使用的调试宏

```rust
fn factorial(n: u32) -> u32 {
    if dbg!(n <= 1) {     // 打印：[src/main.rs:2] n <= 1 = false
        dbg!(1)           // 打印：[src/main.rs:3] 1 = 1
    } else {
        dbg!(n * factorial(n - 1))  // 打印中间值
    }
}

fn main() {
    dbg!(factorial(4));   // 打印所有递归调用带 file:line
}
```

`dbg!` 返回它包装的值，所以你可以在任何地方插入它而不改变程序行为。它打印到 stderr（不是 stdout），所以不干扰程序输出。**在提交代码前删除所有 `dbg!` 调用。**

### 格式字符串语法

由于 `println!`、`format!`、`eprintln!` 和 `write!` 都使用相同的格式机制，这是快速参考：

```rust
let name = "sensor";
let value = 3.14159;
let count = 42;

println!("{name}");                    // 按名称变量（Rust 1.58+）
println!("{}", name);                  // 位置
println!("{value:.2}");                // 2 位小数："3.14"
println!("{count:>10}");               // 右对齐，宽度 10："        42"
println!("{count:0>10}");              // 零填充："0000000042"
println!("{count:#06x}");              // 带前缀的十六进制："0x002a"
println!("{count:#010b}");             // 带前缀的二进制："0b00101010"
println!("{value:?}");                 // Debug 格式
println!("{value:#?}");                // 美观打印的 Debug 格式
```

> **对于 C 开发者：** 将此视为类型安全的 `printf` —— 编译器检查 `{:.2}` 应用于浮点数，不是字符串。无 `%s`/`%d` 格式不匹配 bug。
>
> **对于 C++ 开发者：** 这替换 `std::cout << std::fixed << std::setprecision(2) << value` 为单个可读格式字符串。

---

## Derive 宏

你在这本书的几乎每个结构体上都见过 `#[derive(...)]`：

```rust
#[derive(Debug, Clone, PartialEq)]
struct Point {
    x: f64,
    y: f64,
}
```

`#[derive(Debug)]` 是 **derive 宏** —— 一种特殊的过程宏，自动生成 trait 实现。这是它产生的内容（简化）：

```rust
// #[derive(Debug)] 为 Point 生成的：
impl std::fmt::Debug for Point {
    fn fmt(&self, f: &mut std::fmt::Formatter<'_>) -> std::fmt::Result {
        f.debug_struct("Point")
            .field("x", &self.x)
            .field("y", &self.y)
            .finish()
    }
}
```

没有 `#[derive(Debug)]`，你必须为每个结构体手动编写该 `impl` 块。

### 常见的 derive trait

| Derive | 生成什么 | 何时使用 |
|--------|-------------------|-------------|
| `Debug` | `{:?}` 格式化 | 几乎总是 —— 启用调试打印 |
| `Clone` | `.clone()` 方法 | 当需要复制值时 |
| `Copy` | 赋值时隐式复制 | 小型、仅栈类型（整数、`[f64; 3]`） |
| `PartialEq` / `Eq` | `==` 和 `!=` 运算符 | 当需要相等比较时 |
| `PartialOrd` / `Ord` | `<`、`>`、`<=`、`>=` 运算符 | 当需要排序时 |
| `Hash` | `HashMap`/`HashSet` 键的哈希 | 用作映射键的类型 |
| `Default` | `Type::default()` 构造函数 | 带合理零/空值的类型 |
| `serde::Serialize` / `Deserialize` | JSON/TOML 等序列化 | 跨 API 边界的数据类型 |

### derive 决策树

```text
我应该 derive 吗？
  │
  ├── 我的类型包含的所有类型都实现了该 trait 吗？
  │     ├── 是 → #[derive] 有效
  │     └── 否  → 手动实现（或跳过）
  │
  └── 我的类型的用户会合理地期望这种行为吗？
        ├── 是 → derive 它（Debug、Clone、PartialEq 几乎总是合理的）
        └── 否  → 不要 derive（例如，不要为带文件句柄的类型 derive Copy）
```

> **C++ 对比：** `#[derive(Clone)]` 类似于自动生成正确的复制构造函数。`#[derive(PartialEq)]` 类似于自动生成 `operator==` 比较每个字段 —— 这是 C++20 的 `= default` spaceship 运算符最终提供的。

---

## 属性宏

属性宏转换它们附加的项。你已经使用过几个：

```rust
#[test]                    // 标记函数为测试
fn test_addition() {
    assert_eq!(2 + 2, 4);
}

#[cfg(target_os = "linux")] // 条件包含此函数
fn linux_only() { /* ... */ }

#[derive(Debug)]            // 生成 Debug 实现
struct MyType { /* ... */ }

#[allow(dead_code)]         // 抑制编译器警告
fn unused_helper() { /* ... */ }

#[must_use]                 // 如果返回值被丢弃则警告
fn compute_checksum(data: &[u8]) -> u32 { /* ... */ }
```

常见的内置属性：

| 属性 | 用途 |
|-----------|---------|
| `#[test]` | 标记为测试函数 |
| `#[cfg(...)]` | 条件编译 |
| `#[derive(...)]` | 自动生成 trait 实现 |
| `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` | 控制 lint 级别 |
| `#[must_use]` | 未使用返回值时警告 |
| `#[inline]` / `#[inline(always)]` | 内联函数的提示 |
| `#[repr(C)]` | 使用 C 兼容内存布局（用于 FFI） |
| `#[no_mangle]` | 不修饰符号名（用于 FFI） |
| `#[deprecated]` | 标记为已弃用，带可选消息 |

> **对于 C/C++ 开发者：** 属性替换预处理器指令（`#pragma`、`__attribute__((...)`）和编译器特定扩展的混合。它们是语言语法的一部分，不是临时附加的扩展。

---

## 过程宏（概念概述）

过程宏（"proc 宏"）是作为**独立 Rust 程序**编写的宏，在编译时运行并生成代码。它们比 `macro_rules!` 更强大但也更复杂。

有三种类型：

| 种类 | 语法 | 示例 | 它做什么 |
|------|--------|---------|-------------|
| **函数式** | `my_macro!(...)` | `sql!(SELECT * FROM users)` | 解析自定义语法，生成 Rust 代码 |
| **Derive** | `#[derive(MyTrait)]` | `#[derive(Serialize)]` | 从结构体定义生成 trait 实现 |
| **属性** | `#[my_attr]` | `#[tokio::main]`、`#[instrument]` | 转换注解的项 |

### 你已经使用过 proc 宏

- `#[derive(Error)]` 来自 `thiserror` —— 为错误 enum 生成 `Display` 和 `From` 实现
- `#[derive(Serialize, Deserialize)]` 来自 `serde` —— 生成序列化代码
- `#[tokio::main]` —— 将 `async fn main()` 转换为运行时设置 + block_on
- `#[test]` —— 由测试工具架注册（内置 proc 宏）

### 何时编写自己的 proc 宏

在本课程期间你可能不需要编写 proc 宏。它们有用的情况：
- 需要在编译时检查结构体字段/enum 变体（derive 宏）
- 正在构建领域特定语言（函数式宏）
- 需要转换函数签名（属性宏）

对于大多数代码，`macro_rules!` 或普通函数就足够了。

> **C++ 对比：** 过程宏填充代码生成器、模板元编程和外部工具（如 `protoc`）在 C++ 中的角色。区别在于 proc 宏是 cargo 构建管道的一部分 —— 无外部构建步骤、无 CMake 自定义命令。

---

## 何时使用什么：宏 vs 函数 vs 泛型

```text
需要生成代码？
  │
  ├── 否 → 使用函数或泛型函数
  │         （更简单、更好的错误消息、IDE 支持）
  │
  └── 是 ─┬── 可变数量参数？
            │     └── 是 → macro_rules!（例如 println!、vec!）
            │
            ├── 为多个类型生成重复的 impl 块？
            │     └── 是 → 带重复的 macro_rules!
            │
            ├── 需要检查结构体字段？
            │     └── 是 → Derive 宏（proc 宏）
            │
            ├── 需要自定义语法（DSL）？
            │     └── 是 → 函数式 proc 宏
            │
            └── 需要转换函数/结构体？
                  └── 是 → 属性 proc 宏
```

**一般指南：** 如果函数或泛型可以完成，不要使用宏。宏有更差的错误消息、宏体内无 IDE 自动补全，且更难调试。

---

## 练习

### 🟢 练习 1：`min!` 宏

编写一个 `min!` 宏：
- `min!(a, b)` 返回两个值中较小的
- `min!(a, b, c)` 返回三个值中最小的
- 适用于任何实现 `PartialOrd` 的类型

**提示：** 你的 `macro_rules!` 中需要两个匹配臂。

<details><summary>答案（点击展开）</summary>

```rust
macro_rules! min {
    ($a:expr, $b:expr) => {
        if $a < $b { $a } else { $b }
    };
    ($a:expr, $b:expr, $c:expr) => {
        min!(min!($a, $b), $c)
    };
}

fn main() {
    println!("{}", min!(3, 7));        // 3
    println!("{}", min!(9, 2, 5));     // 2
    println!("{}", min!(1.5, 0.3));    // 0.3
}
```

**注意：** 对于生产代码，优先使用 `std::cmp::min` 或 `a.min(b)`。此练习演示多臂宏的机制。

</details>

### 🟡 练习 2：从头开始编写 `hashmap!`

在不看上面示例的情况下，编写一个 `hashmap!` 宏：
- 从 `key => value` 对创建 `HashMap`
- 支持尾随逗号
- 适用于任何可哈希的键类型

测试：
```rust
let m = hashmap! {
    "name" => "Alice",
    "role" => "Engineer",
};
assert_eq!(m["name"], "Alice");
assert_eq!(m.len(), 2);
```

<details><summary>答案（点击展开）</summary>

```rust
use std::collections::HashMap;

macro_rules! hashmap {
    ( $( $key:expr => $val:expr ),* $(,)? ) => {{
        let mut map = HashMap::new();
        $( map.insert($key, $val); )*
        map
    }};
}

fn main() {
    let m = hashmap! {
        "name" => "Alice",
        "role" => "Engineer",
    };
    assert_eq!(m["name"], "Alice");
    assert_eq!(m.len(), 2);
    println!("Tests passed!");
}
```

</details>

### 🟡 练习 3：`assert_approx_eq!` 用于浮点比较

编写一个宏 `assert_approx_eq!(a, b, epsilon)`，如果 `|a - b| > epsilon` 则 panic。这对于测试浮点计算很有用，因为精确相等会失败。

测试：
```rust
assert_approx_eq!(0.1 + 0.2, 0.3, 1e-10);        // 应该通过
assert_approx_eq!(3.14159, std::f64::consts::PI, 1e-4); // 应该通过
// assert_approx_eq!(1.0, 2.0, 0.5);              // 应该 panic
```

<details><summary>答案（点击展开）</summary>

```rust
macro_rules! assert_approx_eq {
    ($a:expr, $b:expr, $eps:expr) => {
        let (a, b, eps) = ($a as f64, $b as f64, $eps as f64);
        let diff = (a - b).abs();
        if diff > eps {
            panic!(
                "assertion failed: |{} - {}| = {} > {} (epsilon)",
                a, b, diff, eps
            );
        }
    };
}

fn main() {
    assert_approx_eq!(0.1 + 0.2, 0.3, 1e-10);
    assert_approx_eq!(3.14159, std::f64::consts::PI, 1e-4);
    println!("All float comparisons passed!");
}
```

</details>

### 🔴 练习 4：`impl_display_for_enum!`

编写一个宏，为简单的 C 风格 enum 生成 `Display` 实现。给定：

```rust
impl_display_for_enum! {
    enum Color {
        Red => "red",
        Green => "green",
        Blue => "blue",
    }
}
```

它应该生成 `enum Color { Red, Green, Blue }` 定义和 `impl Display for Color`，将每个变体映射到其字符串。

**提示：** 你需要 `$( ... ),*` 重复和多个片段说明符。

<details><summary>答案（点击展开）</summary>

```rust
use std::fmt;

macro_rules! impl_display_for_enum {
    (enum $name:ident { $( $variant:ident => $display:expr ),* $(,)? }) => {
        #[derive(Debug, Clone, Copy, PartialEq)]
        enum $name {
            $( $variant ),*
        }

        impl fmt::Display for $name {
            fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
                match self {
                    $( $name::$variant => write!(f, "{}", $display), )*
                }
            }
        }
    };
}

impl_display_for_enum! {
    enum Color {
        Red => "red",
        Green => "green",
        Blue => "blue",
    }
}

fn main() {
    let c = Color::Green;
    println!("Color: {c}");          // "Color: green"
    println!("Debug: {c:?}");        // "Debug: Green"
    assert_eq!(format!("{}", Color::Red), "red");
    println!("All tests passed!");
}
```

</details>
