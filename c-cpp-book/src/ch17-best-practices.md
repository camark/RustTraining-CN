# Rust 最佳实践总结

> **你将学到什么：** 编写惯用 Rust 的实用指南 —— 代码组织、命名约定、错误处理模式和文档。一个你会经常返回的快速参考章节。

## 代码组织
- **偏好小函数**：易于测试和推理
- **使用描述性名称**：`calculate_total_price()` vs `calc()`
- **分组相关功能**：使用模块和独立文件
- **编写文档**：对公共 API 使用 `///`

## 错误处理
- **避免 `unwrap()` 除非不可失败**：仅在你 100% 确定不会 panic 时使用
```rust
// 坏：可能 panic
let value = some_option.unwrap();

// 好：处理 None 情况
let value = some_option.unwrap_or(default_value);
let value = some_option.unwrap_or_else(|| expensive_computation());
let value = some_option.unwrap_or_default(); // 使用 Default trait

// 对于 Result<T, E>
let value = some_result.unwrap_or(fallback_value);
let value = some_result.unwrap_or_else(|err| {
    eprintln!("发生错误：{err}");
    default_value
});
```
- **使用 `expect()` 带描述性消息**：当 unwrap 合理时，解释为什么
```rust
let config = std::env::var("CONFIG_PATH")
    .expect("CONFIG_PATH 环境变量必须设置");
```
- **为可失败操作返回 `Result<T, E>`**：让调用者决定如何处理错误
- **使用 `thiserror` 用于自定义错误类型**：比手动实现更符合人体工程学
```rust
use thiserror::Error;

#[derive(Error, Debug)]
pub enum MyError {
    #[error("IO 错误：{0}")]
    Io(#[from] std::io::Error),
    
    #[error("解析错误：{message}")]
    Parse { message: String },
    
    #[error("值 {value} 超出范围")]
    OutOfRange { value: i32 },
}
```
- **使用 `?` 运算符链式错误**：将错误向上传播到调用栈
- **优先使用 `thiserror` 而不是 `anyhow`**：我们的团队约定是使用 `#[derive(thiserror::Error)]` 定义显式错误 enum，以便调用者可以匹配特定变体。
  `anyhow::Error` 对于快速原型化很方便，但会擦除错误类型，使调用者更难处理特定失败。对库和生产代码使用 `thiserror`；仅为临时脚本或只需要打印错误的顶层二进制文件保留 `anyhow`。
- **何时接受 `unwrap()`**：
  - **单元测试**：`assert_eq!(result.unwrap(), expected)`
  - **原型化**：你将替换的临时快速代码
  - **不可失败操作**：当你能证明它不会失败时
```rust
let numbers = vec![1, 2, 3];
let first = numbers.get(0).unwrap(); // 安全：我们刚用元素创建了 vec

// 更好：使用 expect() 带解释
let first = numbers.get(0).expect("根据构造 numbers vec 非空");
```
- **快速失败**：尽早检查前置条件并立即返回错误

## 内存管理
- **优先借用而不是克隆**：可能时使用 `&T` 而不是克隆
- **谨慎使用 `Rc<T>``**：仅在你需要共享所有权时
- **限制生命周期**：使用作用域 `{}` 控制何时 drop 值
- **避免在公共 API 中使用 `RefCell<T>`**：保持内部可变性内部化

## 性能
- **优化前分析**：使用 `cargo bench` 和分析工具
- **优先迭代器而不是循环**：更具可读性且通常更快
- **使用 `&str` 而不是 `String`**：当你不需要所有权时
- **考虑对大型栈对象使用 `Box<T>`**：如果需要将它们移动到堆

## 要实现的基本 Traits

### 每个类型应考虑的核心 Traits

创建自定义类型时，考虑实现这些基本 traits 以使你的类型在 Rust 中感觉原生：

#### **Debug 和 Display**
```rust
use std::fmt;

#[derive(Debug)]  // 自动实现用于调试
struct Person {
    name: String,
    age: u32,
}

// 手动 Display 实现用于面向用户的输出
impl fmt::Display for Person {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "{}（年龄 {}）", self.name, self.age)
    }
}

// 用法：
let person = Person { name: "Alice".to_string(), age: 30 };
println!("{:?}", person);  // Debug: Person { name: "Alice", age: 30 }
println!("{}", person);    // Display: Alice（年龄 30）
```

#### **Clone 和 Copy**
```rust
// Copy：小型、简单类型的隐式复制
#[derive(Debug, Clone, Copy)]
struct Point {
    x: i32,
    y: i32,
}

// Clone：复杂类型的显式复制
#[derive(Debug, Clone)]
struct Person {
    name: String,  // String 不实现 Copy
    age: u32,
}

let p1 = Point { x: 1, y: 2 };
let p2 = p1;  // Copy（隐式）

let person1 = Person { name: "Bob".to_string(), age: 25 };
let person2 = person1.clone();  // Clone（显式）
```

#### **PartialEq 和 Eq**
```rust
#[derive(Debug, PartialEq, Eq)]
struct UserId(u64);

#[derive(Debug, PartialEq)]
struct Temperature {
    celsius: f64,  // f64 不实现 Eq（由于 NaN）
}

let id1 = UserId(123);
let id2 = UserId(123);
assert_eq!(id1, id2);  // 有效因为 PartialEq

let temp1 = Temperature { celsius: 20.0 };
let temp2 = Temperature { celsius: 20.0 };
assert_eq!(temp1, temp2);  // 使用 PartialEq 有效
```

#### **PartialOrd 和 Ord**
```rust
#[derive(Debug, PartialEq, Eq, PartialOrd, Ord)]
struct Priority(u8);

let high = Priority(1);
let low = Priority(10);
assert!(high < low);  // 较小数字 = 较高优先级

// 在集合中使用
let mut priorities = vec![Priority(5), Priority(1), Priority(8)];
priorities.sort();  // 有效因为 Priority 实现 Ord
```

#### **Default**
```rust
#[derive(Default)]
struct Config {
    debug: bool,           // false（默认）
    max_connections: u32,  // 0（默认）
    timeout: Option<u64>,  // None（默认）
}

// 自定义 Default 实现
impl Default for Config {
    fn default() -> Self {
        Config {
            debug: false,
            max_connections: 100,  // 自定义默认
            timeout: Some(30),     // 自定义默认
        }
    }
}

let config = Config::default();
let config = Config { debug: true, ..Default::default() };  // 部分覆盖
```

#### **From 和 Into**
```rust
struct UserId(u64);
struct UserName(String);

// 实现 From，免费获得 Into
impl From<u64> for UserId {
    fn from(id: u64) -> Self {
        UserId(id)
    }
}

impl From<String> for UserName {
    fn from(name: String) -> Self {
        UserName(name)
    }
}

impl From<&str> for UserName {
    fn from(name: &str) -> Self {
        UserName(name.to_string())
    }
}

// 用法：
let user_id: UserId = 123u64.into();         // 使用 Into
let user_id = UserId::from(123u64);          // 使用 From
let username = UserName::from("alice");      // &str -> UserName
let username: UserName = "bob".into();       // 使用 Into
```

#### **TryFrom 和 TryInto**
```rust
use std::convert::TryFrom;

struct PositiveNumber(u32);

#[derive(Debug)]
struct NegativeNumberError;

impl TryFrom<i32> for PositiveNumber {
    type Error = NegativeNumberError;
    
    fn try_from(value: i32) -> Result<Self, Self::Error> {
        if value >= 0 {
            Ok(PositiveNumber(value as u32))
        } else {
            Err(NegativeNumberError)
        }
    }
}

// 用法：
let positive = PositiveNumber::try_from(42)?;     // Ok(PositiveNumber(42))
let error = PositiveNumber::try_from(-5);         // Err(NegativeNumberError)
```

#### **Serde（用于序列化）**
```rust
use serde::{Deserialize, Serialize};

#[derive(Debug, Serialize, Deserialize)]
struct User {
    id: u64,
    name: String,
    email: String,
}

// 自动 JSON 序列化/反序列化
let user = User {
    id: 1,
    name: "Alice".to_string(),
    email: "alice@example.com".to_string(),
};

let json = serde_json::to_string(&user)?;
let deserialized: User = serde_json::from_str(&json)?;
```

### Trait 实现检查清单

对于任何新类型，考虑此检查清单：

```rust
#[derive(
    Debug,          // [好] 总是实现用于调试
    Clone,          // [好] 如果类型应该是可复制的
    PartialEq,      // [好] 如果类型应该是可比较的
    Eq,             // [好] 如果比较是自反/传递的
    PartialOrd,     // [好] 如果类型有顺序
    Ord,            // [好] 如果顺序是完全的
    Hash,           // [好] 如果类型将用作 HashMap 键
    Default,        // [好] 如果有合理的默认值
)]
struct MyType {
    // 字段...
}

// 考虑的手动实现：
impl Display for MyType { /* 面向用户的表示 */ }
impl From<OtherType> for MyType { /* 方便的转换 */ }
impl TryFrom<FallibleType> for MyType { /* 可失败的转换 */ }
```

### 何时不实现 Traits

- **不要为带堆数据的类型实现 Copy**：`String`、`Vec`、`HashMap` 等
- **如果值可以是 NaN 则不要实现 Eq**：包含 `f32`/`f64` 的类型
- **如果没有合理的默认值则不要实现 Default**：文件句柄、网络连接
- **如果克隆开销大则不要实现 Clone**：大型数据结构（考虑改用 `Rc<T>`）

### 总结：Trait 优势

| Trait | 优势 | 何时使用 |
|-------|---------|-------------|
| `Debug` | `println!("{:?}", value)` | 总是（罕见情况除外） |
| `Display` | `println!("{}", value)` | 面向用户的类型 |
| `Clone` | `value.clone()` | 当显式复制有意义时 |
| `Copy` | 隐式复制 | 小型、简单类型 |
| `PartialEq` | `==` 和 `!=` 运算符 | 大多数类型 |
| `Eq` | 自反平等 | 当平等在数学上合理时 |
| `PartialOrd` | `<`、`>`、`<=`、`>=` | 有自然顺序的类型 |
| `Ord` | `sort()`、`BinaryHeap` | 当顺序是完全时 |
| `Hash` | `HashMap` 键 | 用作映射键的类型 |
| `Default` | `Default::default()` | 有明显默认值的类型 |
| `From/Into` | 方便的转换 | 常见类型转换 |
| `TryFrom/TryInto` | 可失败的转换 | 可能失败的转换 |

----

----


