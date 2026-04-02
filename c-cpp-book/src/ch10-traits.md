# Rust traits

> **你将学到什么：** Traits —— Rust 对接口、抽象基类和运算符重载的回答。你将学习如何定义 traits、为你的类型实现它们，以及使用动态派（`dyn Trait`）vs 静态派（泛型）。对于 C++ 开发者：traits 取代虚函数、CRTP 和 concepts。对于 C 开发者：traits 是 Rust 进行多态性的结构化方式。

- Rust traits 与其他语言中的接口类似
    - Traits 定义必须由实现 trait 的类型定义的方法。
```rust
fn main() {
    trait Pet {
        fn speak(&self);
    }
    struct Cat;
    struct Dog;
    impl Pet for Cat {
        fn speak(&self) {
            println!("Meow");
        }
    }
    impl Pet for Dog {
        fn speak(&self) {
            println!("Woof!")
        }
    }
    let c = Cat{};
    let d = Dog{};
    c.speak();  // Cat 和 Dog 之间没有"is a"关系
    d.speak(); // Cat 和 Dog 之间没有"is a"关系
}
```

## Traits vs C++ Concepts 和接口

### 传统 C++ 继承 vs Rust Traits

```cpp
// C++ - 基于继承的多态
class Animal {
public:
    virtual void speak() = 0;  // 纯虚函数
    virtual ~Animal() = default;
};

class Cat : public Animal {  // "Cat IS-A Animal"
public:
    void speak() override {
        std::cout << "Meow" << std::endl;
    }
};

void make_sound(Animal* animal) {  // 运行时多态
    animal->speak();  // 虚函数调用
}
```

```rust
// Rust - 使用 traits 组合优于继承
trait Animal {
    fn speak(&self);
}

struct Cat;  // Cat 不是 Animal，但实现 Animal 行为

impl Animal for Cat {  // "Cat CAN-DO Animal 行为"
    fn speak(&self) {
        println!("Meow");
    }
}

fn make_sound<T: Animal>(animal: &T) {  // 静态多态
    animal.speak();  // 直接函数调用（零成本）
}
```

```mermaid
graph TD
    subgraph "C++ 面向对象层次结构"
        CPP_ANIMAL["Animal<br/>（抽象基类）"]
        CPP_CAT["Cat : public Animal<br/>（IS-A 关系）"]
        CPP_DOG["Dog : public Animal<br/>（IS-A 关系）"]
        
        CPP_ANIMAL --> CPP_CAT
        CPP_ANIMAL --> CPP_DOG
        
        CPP_VTABLE["虚函数表<br/>（运行时派）"]
        CPP_HEAP["通常需要<br/>堆分配"]
        CPP_ISSUES["[错误] 深层继承树<br/>[错误] 菱形问题<br/>[错误] 运行时开销<br/>[错误] 紧耦合"]
    end
    
    subgraph "Rust 基于 Trait 的组合"
        RUST_TRAIT["trait Animal<br/>（行为定义）"]
        RUST_CAT["struct Cat<br/>（仅数据）"]
        RUST_DOG["struct Dog<br/>（仅数据）"]
        
        RUST_CAT -.->|"impl Animal for Cat<br/>（CAN-DO 行为）"| RUST_TRAIT
        RUST_DOG -.->|"impl Animal for Dog<br/>（CAN-DO 行为）"| RUST_TRAIT
        
        RUST_STATIC["静态派<br/>（编译时）"]
        RUST_STACK["栈分配<br/>可能"]
        RUST_BENEFITS["[好] 无继承层次结构<br/>[好] 多个 trait 实现<br/>[好] 零运行时成本<br/>[好] 松耦合"]
    end
    
    style CPP_ISSUES fill:#ff6b6b,color:#000
    style RUST_BENEFITS fill:#91e5a3,color:#000
    style CPP_VTABLE fill:#ffa07a,color:#000
    style RUST_STATIC fill:#91e5a3,color:#000
```

### Trait 边界和泛型约束

```rust
use std::fmt::Display;
use std::ops::Add;

// C++ 模板等价物（约束较少）
// template<typename T>
// T add_and_print(T a, T b) {
//     // 无保证 T 支持 + 或打印
//     return a + b;  // 可能在编译时失败
// }

// Rust - 显式 trait 边界
fn add_and_print<T>(a: T, b: T) -> T 
where 
    T: Display + Add<Output = T> + Copy,
{
    println!("相加 {} + {}", a, b);  // Display trait
    a + b  // Add trait
}
```

```mermaid
graph TD
    subgraph "泛型约束演进"
        UNCONSTRAINED["fn process<T>(data: T)<br/>[错误] T 可以是任何类型"]
        SINGLE_BOUND["fn process<T: Display>(data: T)<br/>[好] T 必须实现 Display"]
        MULTI_BOUND["fn process<T>(data: T)<br/>where T: Display + Clone + Debug<br/>[好] 多个要求"]
        
        UNCONSTRAINED --> SINGLE_BOUND
        SINGLE_BOUND --> MULTI_BOUND
    end
    
    subgraph "Trait 边界语法"
        INLINE["fn func<T: Trait>(param: T)"]
        WHERE_CLAUSE["fn func<T>(param: T)<br/>where T: Trait"]
        IMPL_PARAM["fn func(param: impl Trait)"]
        
        COMPARISON["内联：简单情况<br/>Where：复杂边界<br/>impl：简洁语法"]
    end
    
    subgraph "编译时魔法"
        GENERIC_FUNC["泛型函数<br/>带 trait 边界"]
        TYPE_CHECK["编译器验证<br/>trait 实现"]
        MONOMORPH["单态化<br/>（创建特化版本）"]
        OPTIMIZED["完全优化的<br/>机器码"]
        
        GENERIC_FUNC --> TYPE_CHECK
        TYPE_CHECK --> MONOMORPH
        MONOMORPH --> OPTIMIZED
        
        EXAMPLE["add_and_print::<i32><br/>add_and_print::<f64><br/>（生成单独的函数）"]
        MONOMORPH --> EXAMPLE
    end
    
    style UNCONSTRAINED fill:#ff6b6b,color:#000
    style SINGLE_BOUND fill:#ffa07a,color:#000
    style MULTI_BOUND fill:#91e5a3,color:#000
    style OPTIMIZED fill:#91e5a3,color:#000
```

### C++ 运算符重载 → Rust `std::ops` Traits

在 C++ 中，你通过编写具有特殊名称（`operator+`、`operator<<`、`operator[]` 等）的自由函数或成员函数来重载运算符。在 Rust 中，每个运算符映射到 `std::ops`（或 `std::fmt` 用于输出）中的一个 trait。你**实现 trait** 而不是编写魔法名称的函数。

#### 并排对比：`+` 运算符

```cpp
// C++：运算符重载作为成员或自由函数
struct Vec2 {
    double x, y;
    Vec2 operator+(const Vec2& rhs) const {
        return {x + rhs.x, y + rhs.y};
    }
};

Vec2 a{1.0, 2.0}, b{3.0, 4.0};
Vec2 c = a + b;  // 调用 a.operator+(b)
```

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Vec2 { x: f64, y: f64 }

impl Add for Vec2 {
    type Output = Vec2;                     // 关联类型 —— + 的结果
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

let a = Vec2 { x: 1.0, y: 2.0 };
let b = Vec2 { x: 3.0, y: 4.0 };
let c = a + b;  // 调用 <Vec2 as Add>::add(a, b)
println!("{c:?}"); // Vec2 { x: 4.0, y: 6.0 }
```

#### 与 C++ 的关键区别

| 方面 | C++ | Rust |
|--------|-----|------|
| **机制** | 魔法函数名（`operator+`） | 实现 trait（`impl Add for T`） |
| **发现** | Grep 搜索 `operator+` 或阅读头文件 | 查找 trait 实现 —— IDE 支持优秀 |
| **返回类型** | 自由选择 | 由 `Output` 关联类型固定 |
| **接收者** | 通常采用 `const T&`（借用） | 默认采用 `self` 按值（移动！） |
| **对称性** | 可以写 `impl operator+(int, Vec2)` | 必须添加 `impl Add<Vec2> for i32`（适用外部 trait 规则） |
| **`<<` 用于打印** | `operator<<(ostream&, T)` —— 为*任何*流重载 | `impl fmt::Display for T` —— 一个规范的 `to_string` 表示 |

#### `self` 按值的陷阱

在 Rust 中，`Add::add(self, rhs)` 采用 `self`**按值**。对于 `Copy` 类型（如上面的 `Vec2`，派生 `Copy`）这没问题 —— 编译器复制。但对于非 `Copy` 类型，`+`**消耗**操作数：

```rust
let s1 = String::from("hello ");
let s2 = String::from("world");
let s3 = s1 + &s2;  // s1 被移动到 s3！
// println!("{s1}");  // ❌ 编译错误：移动后使用值
println!("{s2}");     // ✅ s2 仅被借用（&s2）
```

这就是为什么 `String + &str` 可行但 `&str + &str` 不可行 —— `Add` 仅为 `String + &str` 实现，消耗左侧 `String` 以重用其缓冲区。这在 C++ 中没有 аналог：`std::string::operator+` 总是创建新字符串。

#### 完整映射：C++ 运算符 → Rust traits

| C++ 运算符 | Rust Trait | 注释 |
|-------------|-----------|-------|
| `operator+` | `std::ops::Add` | `Output` 关联类型 |
| `operator-` | `std::ops::Sub` | |
| `operator*` | `std::ops::Mul` | 不是指针解引用 —— 那是 `Deref` |
| `operator/` | `std::ops::Div` | |
| `operator%` | `std::ops::Rem` | |
| `operator-`（一元） | `std::ops::Neg` | |
| `operator!` / `operator~` | `std::ops::Not` | Rust 使用 `!` 表示逻辑和按位 NOT（无 `~` 运算符） |
| `operator&`、`\|`、`^` | `BitAnd`、`BitOr`、`BitXor` | |
| `operator<<`、`>>`（移位） | `Shl`、`Shr` | 不是流 I/O！ |
| `operator+=` | `std::ops::AddAssign` | 采用 `&mut self`（不是 `self`） |
| `operator[]` | `std::ops::Index` / `IndexMut` | 返回 `&Output` / `&mut Output` |
| `operator()` | `Fn` / `FnMut` / `FnOnce` | 闭包实现这些；你不能直接 `impl Fn` |
| `operator==` | `PartialEq`（+ `Eq`） | 在 `std::cmp` 中，不在 `std::ops` |
| `operator<` | `PartialOrd`（+ `Ord`） | 在 `std::cmp` 中 |
| `operator<<`（流） | `fmt::Display` | `println!("{}", x)` |
| `operator<<`（调试） | `fmt::Debug` | `println!("{:?}", x)` |
| `operator bool` | 无直接等价物 | 使用 `impl From<T> for bool` 或命名方法如 `.is_empty()` |
| `operator T()`（隐式转换） | 无隐式转换 | 使用 `From`/`Into` traits（显式） |

#### 防护栏：Rust 防止什么

1. **无隐式转换**：C++ `operator int()` 可能导致静默、惊人的转换。Rust 没有隐式转换运算符 —— 使用 `From`/`Into` 并显式调用 `.into()`。
2. **无重载 `&&` / `||`**：C++ 允许它（破坏短路语义！）。Rust 不允许。
3. **无重载 `=`**：赋值总是移动或复制，从不用户定义。复合赋值（`+=`）可通过 `AddAssign` 等重载。
4. **无重载 `,`**：C++ 允许 `operator,()` —— 最臭名昭著的 C++ 陷阱之一。Rust 不允许。
5. **无重载 `&`（取地址）**：另一个 C++ 陷阱（`std::addressof` 存在以解决它）。Rust 的 `&` 总是表示"借用"。
6. **相干规则**：你只能为自己的类型实现 `Add<Foreign>`，或为外部类型实现 `Add<YourType>` —— 从不为 `Foreign` 实现 `Add<Foreign>`。这防止跨 crate 的冲突运算符定义。

> **底线**：在 C++ 中，运算符重载强大但基本不受监管 —— 你可以重载几乎所有内容，包括逗号和取地址，隐式转换可能静默触发。Rust 通过 traits 为你提供与 C++ 相同的算术和比较运算符表现力，但**阻止历史上危险的重载**并强制所有转换显式。

----
# Rust traits
- Rust 允许在内置类型（如本示例中的 u32）上实现用户定义的 trait。但是，trait 或类型必须属于 crate
```rust
trait IsSecret {
  fn is_secret(&self);
}
// IsSecret trait 属于 crate，所以我们 OK
impl IsSecret for u32 {
  fn is_secret(&self) {
      if *self == 42 {
          println!("是生命之谜");
      }
  }
}

fn main() {
  42u32.is_secret();
  43u32.is_secret();
}
```


# Rust traits
- Traits 支持接口继承和默认实现
```rust
trait Animal {
  // 默认实现
  fn is_mammal(&self) -> bool {
    true
  }
}
trait Feline : Animal {
  // 默认实现
  fn is_feline(&self) -> bool {
    true
  }
}

struct Cat;
// 使用默认实现。注意必须单独实现超 trait 的所有 traits
impl Feline for Cat {}
impl Animal for Cat {}
fn main() {
  let c = Cat{};
  println!("{} {}", c.is_mammal(), c.is_feline());
}
```
----
# 练习：Logger trait 实现

🟡 **中级**

- 实现一个 ```Log trait```，带单个名为 log() 的方法，接受 u64
    - 实现两个不同的日志记录器 ```SimpleLogger``` 和 ```ComplexLogger``` 实现 ```Log trait```。一个应该输出 "Simple logger" 带 ```u64```，另一个应该输出 "Complex logger" 带 ```u64```

<details><summary>答案（点击展开）</summary>

```rust
trait Log {
    fn log(&self, value: u64);
}

struct SimpleLogger;
struct ComplexLogger;

impl Log for SimpleLogger {
    fn log(&self, value: u64) {
        println!("Simple logger: {value}");
    }
}

impl Log for ComplexLogger {
    fn log(&self, value: u64) {
        println!("Complex logger: {value} (hex: 0x{value:x}, binary: {value:b})");
    }
}

fn main() {
    let simple = SimpleLogger;
    let complex = ComplexLogger;
    simple.log(42);
    complex.log(42);
}
// 输出：
// Simple logger: 42
// Complex logger: 42 (hex: 0x2a, binary: 101010)
```

</details>

----
# Rust trait 关联类型
```rust
#[derive(Debug)]
struct Small(u32);
#[derive(Debug)]
struct Big(u32);
trait Double {
    type T;
    fn double(&self) -> Self::T;
}

impl Double for Small {
    type T = Big;
    fn double(&self) -> Self::T {
        Big(self.0 * 2)
    }
}
fn main() {
    let a = Small(42);
    println!("{:?}", a.double());
}
```

# Rust trait 实现
- ```impl``` 可与 traits 一起使用以接受任何实现 trait 的类型
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &impl Pet) {
    p.speak();
}
fn main() {
    let c = Cat {};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```

# Rust trait 实现
- ```impl``` 也可用于返回值
```rust
trait Pet {}
struct Dog;
struct Cat;
impl Pet for Cat {}
impl Pet for Dog {}
fn cat_as_pet() -> impl Pet {
    let c = Cat {};
    c
}
fn dog_as_pet() -> impl Pet {
    let d = Dog {};
    d
}
fn main() {
    let p = cat_as_pet();
    let d = dog_as_pet();
}
```
----
# Rust 动态 traits
- 动态 traits 可用于调用 trait 功能而无需知道底层类型。这称为 ```类型擦除```
```rust
trait Pet {
    fn speak(&self);
}
struct Dog {}
struct Cat {x: u32}
impl Pet for Dog {
    fn speak(&self) {println!("Woof!")}
}
impl Pet for Cat {
    fn speak(&self) {println!("Meow")}
}
fn pet_speak(p: &dyn Pet) {
    p.speak();
}
fn main() {
    let c = Cat {x: 42};
    let d = Dog {};
    pet_speak(&c);
    pet_speak(&d);
}
```
----

## 在 `impl Trait`、`dyn Trait` 和 Enums 之间选择

这三种方法都实现多态性，但具有不同的权衡：

| 方法 | 派 | 性能 | 异构集合？ | 何时使用 |
|----------|----------|-------------|---------------------------|-------------|
| `impl Trait` / 泛型 | 静态（单态化） | 零成本 —— 编译时内联 | 否 —— 每个槽位有一个具体类型 | 默认选择。函数参数、返回类型 |
| `dyn Trait` | 动态（vtable） | 小开销每次调用（~1 指针间接） | 是 —— `Vec<Box<dyn Trait>>` | 当你需要在集合中混合类型，或插件式可扩展性 |
| `enum` | Match | 零成本 —— 编译时已知变体 | 是 —— 但仅已知变体 | 当变体集合**封闭**且编译时已知 |

```rust
trait Shape {
    fn area(&self) -> f64;
}
struct Circle { radius: f64 }
struct Rect { w: f64, h: f64 }
impl Shape for Circle { fn area(&self) -> f64 { std::f64::consts::PI * self.radius * self.radius } }
impl Shape for Rect   { fn area(&self) -> f64 { self.w * self.h } }

// 静态派 —— 编译器为每个类型生成单独的代码
fn print_area(s: &impl Shape) { println!("{}", s.area()); }

// 动态派 —— 一个函数，适用于指针后面的任何 Shape
fn print_area_dyn(s: &dyn Shape) { println!("{}", s.area()); }

// Enum —— 封闭集合，不需要 trait
enum ShapeEnum { Circle(f64), Rect(f64, f64) }
impl ShapeEnum {
    fn area(&self) -> f64 {
        match self {
            ShapeEnum::Circle(r) => std::f64::consts::PI * r * r,
            ShapeEnum::Rect(w, h) => w * h,
        }
    }
}
```

> **对于 C++ 开发者：** `impl Trait` 像 C++ 模板（单态化，零成本）。`dyn Trait` 像 C++ 虚函数（vtable 派）。带 `match` 的 Rust enums 像带 `std::visit` 的 `std::variant` —— 但穷尽匹配由编译器强制执行。

> **经验法则**：从 `impl Trait`（静态派）开始。仅当你需要异构集合或无法在编译时知道具体类型时才使用 `dyn Trait`。当你拥有所有变体时使用 `enum`。

