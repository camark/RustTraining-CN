## Traits vs 鸭子类型

> **你将学到什么：** Traits 作为显式契约（vs Python 鸭子类型），`Protocol`（PEP 544）≈ Trait，
> 使用 `where` 子句的泛型类型约束，trait 对象（`dyn Trait`）vs 静态分派，以及常见的标准库 traits。
>
> **难度：** 🟡 中级

这是 Rust 类型系统真正为 Python 开发者发光发热的地方。Python 的
"鸭子类型" 说："如果它像鸭子一样走路并且像鸭子一样叫，那它就是鸭子。"
Rust 的 traits 说："我会在编译时告诉你我具体需要哪些鸭子行为。"

### Python 鸭子类型
```python
# Python —— 鸭子类型：有任何正确方法的东西都能工作
def total_area(shapes):
    """适用于任何有 .area() 方法的东西。"""
    return sum(shape.area() for shape in shapes)

class Circle:
    def __init__(self, radius): self.radius = radius
    def area(self): return 3.14159 * self.radius ** 2

class Rectangle:
    def __init__(self, w, h): self.w, self.h = w, h
    def area(self): return self.w * self.h

# 在运行时工作 —— 不需要继承！
shapes = [Circle(5), Rectangle(3, 4)]
print(total_area(shapes))  # 90.54

# 但如果某个东西没有 .area() 会怎样？
class Dog:
    def bark(self): return "Woof!"

total_area([Dog()])  # 💥 AttributeError: 'Dog' has no attribute 'area'
# 错误发生在*运行时*，不是定义时
```

### Rust Traits —— 显式鸭子类型
```rust
// Rust —— traits 使"鸭子"契约显式
trait HasArea {
    fn area(&self) -> f64;      // 任何实现这个 trait 的类型都有 .area()
}

struct Circle { radius: f64 }
struct Rectangle { width: f64, height: f64 }

impl HasArea for Circle {
    fn area(&self) -> f64 {
        std::f64::consts::PI * self.radius * self.radius
    }
}

impl HasArea for Rectangle {
    fn area(&self) -> f64 {
        self.width * self.height
    }
}

// trait 约束是显式的 —— 编译器在编译时检查
fn total_area(shapes: &[&dyn HasArea]) -> f64 {
    shapes.iter().map(|s| s.area()).sum()
}

// 使用它：
let shapes: Vec<&dyn HasArea> = vec![&Circle { radius: 5.0 }, &Rectangle { width: 3.0, height: 4.0 }];
println!("{}", total_area(&shapes));  // 90.54

// struct Dog;
// total_area(&[&Dog {}]);  // ❌ 编译错误：Dog 没有实现 HasArea
```

> **关键见解**：Python 的鸭子类型将错误推迟到运行时。Rust 的 traits 在编译时捕获它们。同样的灵活性，更早的错误检测。

***

## Protocol (PEP 544) vs Traits

Python 3.8 引入了 `Protocol`（PEP 544）用于结构子类型化 —— 它是最接近 Rust traits 的 Python 概念。

### Python Protocol
```python
# Python —— Protocol（结构类型，类似 Rust traits）
from typing import Protocol, runtime_checkable

@runtime_checkable
class Printable(Protocol):
    def to_string(self) -> str: ...

class User:
    def __init__(self, name: str):
        self.name = name
    def to_string(self) -> str:
        return f"User({self.name})"

class Product:
    def __init__(self, name: str, price: float):
        self.name = name
        self.price = price
    def to_string(self) -> str:
        return f"Product({self.name}, ${self.price:.2f})"

def print_all(items: list[Printable]) -> None:
    for item in items:
        print(item.to_string())

# 可行是因为 User 和 Product 都有 to_string()
print_all([User("Alice"), Product("Widget", 9.99)])

# 但是：mypy 检查这个，Python 运行时不强制执行
# print_all([42])  # mypy 警告，但 Python 运行它并崩溃
```

### Rust Trait（等价，但强制执行！）
```rust
// Rust —— traits 在编译时强制执行
trait Printable {
    fn to_string(&self) -> String;
}

struct User { name: String }
struct Product { name: String, price: f64 }

impl Printable for User {
    fn to_string(&self) -> String {
        format!("User({})", self.name)
    }
}

impl Printable for Product {
    fn to_string(&self) -> String {
        format!("Product({}, ${:.2})", self.name, self.price)
    }
}

fn print_all(items: &[&dyn Printable]) {
    for item in items {
        println!("{}", item.to_string());
    }
}

// print_all(&[&42i32]);  // ❌ 编译错误：i32 没有实现 Printable
```

### 对比表

| 特性 | Python Protocol | Rust Trait |
|------|----------------|------------|
| 结构类型 | ✅（隐式） | ❌（显式 `impl`） |
| 检查于 | 运行时（或 mypy） | 编译时（总是） |
| 默认实现 | ❌ | ✅ |
| 可以添加到外部类型 | ❌ | ✅（有限制） |
| 多个 protocols | ✅ | ✅（多个 traits） |
| 关联类型 | ❌ | ✅ |
| 泛型约束 | ✅（使用 `TypeVar`） | ✅（trait 约束） |

***

## 泛型约束

### Python 泛型
```python
# Python —— TypeVar 用于泛型函数
from typing import TypeVar, Sequence

T = TypeVar('T')

def first(items: Sequence[T]) -> T | None:
    return items[0] if items else None

# 有界 TypeVar
from typing import SupportsFloat
T = TypeVar('T', bound=SupportsFloat)

def average(items: Sequence[T]) -> float:
    return sum(float(x) for x in items) / len(items)
```

### Rust 带 Trait 约束的泛型
```rust
// Rust —— 带 trait 约束的泛型
fn first<T>(items: &[T]) -> Option<&T> {
    items.first()
}

// 带 trait 约束 —— "T 必须实现这些 traits"
fn average<T>(items: &[T]) -> f64
where
    T: Into<f64> + Copy,   // T 必须能转换为 f64 并且可复制
{
    let sum: f64 = items.iter().map(|&x| x.into()).sum();
    sum / items.len() as f64
}

// 多个约束 —— "T 必须实现 Display AND Debug AND Clone"
fn log_and_clone<T: std::fmt::Display + std::fmt::Debug + Clone>(item: &T) -> T {
    println!("Display: {}", item);
    println!("Debug: {:?}", item);
    item.clone()
}

// impl Trait 的简写（用于简单情况）
fn print_it(item: &impl std::fmt::Display) {
    println!("{}", item);
}
```

### 泛型快速参考

| Python | Rust | 说明 |
|--------|------|------|
| `TypeVar('T')` | `<T>` | 无界泛型 |
| `TypeVar('T', bound=X)` | `<T: X>` | 有界泛型 |
| `Union[int, str]` | `enum` 或 trait 对象 | Rust 没有联合类型 |
| `Sequence[T]` | `&[T]`（切片） | 借用序列 |
| `Callable[[A], R]` | `Fn(A) -> R` | 函数 trait |
| `Optional[T]` | `Option<T>` | 内置于语言中 |

***

## 常见标准库 Traits

这些是 Rust 版本的 Python"魔术方法" —— 它们定义类型在常见情况下的行为。

### Display 和 Debug（打印）
```rust
use std::fmt;

// Debug —— 类似 __repr__（可自动派生）
#[derive(Debug)]
struct Point { x: f64, y: f64 }
// 现在可以：println!("{:?}", point);

// Display —— 类似 __str__（必须手动实现）
impl fmt::Display for Point {
    fn fmt(&self, f: &mut fmt::Formatter<'_>) -> fmt::Result {
        write!(f, "({}, {})", self.x, self.y)
    }
}
// 现在可以：println!("{}", point);
```

### 比较 Traits
```rust
// PartialEq —— 类似 __eq__
// Eq —— 完全相等（f64 是 PartialEq 但不是 Eq，因为 NaN != NaN）
// PartialOrd —— 类似 __lt__、__le__ 等
// Ord —— 完全有序

#[derive(Debug, PartialEq, Eq, PartialOrd, Ord, Hash, Clone)]
struct Student {
    name: String,
    grade: i32,
}

// 现在 students 可以：比较、排序、用作 HashMap 键、克隆
let mut students = vec![
    Student { name: "Charlie".into(), grade: 85 },
    Student { name: "Alice".into(), grade: 92 },
];
students.sort();  // 使用 Ord —— 按姓名然后成绩排序（结构体字段顺序）
```

### Iterator Trait
```rust
// 实现 Iterator —— 类似 Python 的 __iter__/__next__
struct Countdown { value: i32 }

impl Iterator for Countdown {
    type Item = i32;       // 迭代器产生的类型

    fn next(&mut self) -> Option<Self::Item> {
        if self.value > 0 {
            self.value -= 1;
            Some(self.value + 1)
        } else {
            None             // 迭代完成
        }
    }
}

// 用法：
for n in (Countdown { value: 5 }) {
    println!("{n}");  // 5, 4, 3, 2, 1
}
```

### 常见 Traits 一览

| Rust Trait | Python 等价物 | 用途 |
|-----------|--------------|------|
| `Display` | `__str__` | 可读字符串 |
| `Debug` | `__repr__` | Debug 字符串（可派生） |
| `Clone` | `copy.deepcopy` | 深拷贝 |
| `Copy` | （int/float 自动拷贝） | 简单类型的隐式拷贝 |
| `PartialEq` / `Eq` | `__eq__` | 相等性比较 |
| `PartialOrd` / `Ord` | `__lt__` 等 | 排序 |
| `Hash` | `__hash__` | 可哈希（用于 dict 键） |
| `Default` | 默认 `__init__` | 默认值 |
| `From` / `Into` | `__init__` 重载 | 类型转换 |
| `Iterator` | `__iter__` / `__next__` | 迭代 |
| `Drop` | `__del__` / `__exit__` | 清理 |
| `Add`、`Sub`、`Mul` | `__add__`、`__sub__`、`__mul__` | 运算符重载 |
| `Index` | `__getitem__` | 使用 `[]` 索引 |
| `Deref` | （无等价物） | 智能指针解引用 |
| `Send` / `Sync` | （无等价物） | 线程安全标记 |

```mermaid
flowchart TB
    subgraph Static ["Static Dispatch (impl Trait)"]
        G["fn notify(item: &impl Summary)"] --> M1["Compiled: notify_Article()"]
        G --> M2["Compiled: notify_Tweet()"]
        M1 --> O1["Inlined, zero-cost"]
        M2 --> O2["Inlined, zero-cost"]
    end
    subgraph Dynamic ["Dynamic Dispatch (dyn Trait)"]
        D["fn notify(item: &dyn Summary)"] --> VT["vtable lookup"]
        VT --> I1["Article::summarize()"]
        VT --> I2["Tweet::summarize()"]
    end
    style Static fill:#d4edda
    style Dynamic fill:#fff3cd
```

> **Python 等价物**：Python *总是* 使用动态分派（运行时 `getattr`）。Rust 默认使用静态分派（单体化 —— 编译器为每个具体类型生成专用代码）。仅当你需要运行时多态时才使用 `dyn Trait`。
>
> 📌 **另见**：[第 11 章 —— From/Into Traits](ch11-from-and-into-traits.md) 深入涵盖转换 traits（`From`、`Into`、`TryFrom`）。

### 关联类型

Rust traits 可以定义*关联类型* —— 每个实现者填充的类型占位符。Python 没有等价物：

```rust
// Iterator 定义了一个关联类型 'Item'
trait Iterator {
    type Item;
    fn next(&mut self) -> Option<Self::Item>;
}

struct Countdown { remaining: u32 }

impl Iterator for Countdown {
    type Item = u32;  // 这个迭代器产生 u32 值
    fn next(&mut self) -> Option<u32> {
        if self.remaining > 0 {
            self.remaining -= 1;
            Some(self.remaining)
        } else {
            None
        }
    }
}
```

在 Python 中，`__iter__` / `__next__` 返回 `Any` —— 没有办法声明"这个迭代器产生 `int`"并强制执行（类型提示 `Iterator[int]` 只是建议性的）。

### 运算符重载：`__add__` → `impl Add`

Python 使用魔术方法（`__add__`、`__mul__`）。Rust 使用 trait 实现 —— 相同的想法，但是在编译时类型检查：

```python
# Python
class Vec2:
    def __init__(self, x, y):
        self.x, self.y = x, y
    def __add__(self, other):
        return Vec2(self.x + other.x, self.y + other.y)  # 'other' 上没有类型检查
```

```rust
use std::ops::Add;

#[derive(Debug, Clone, Copy)]
struct Vec2 { x: f64, y: f64 }

impl Add for Vec2 {
    type Output = Vec2;  // 关联类型：+ 返回什么？
    fn add(self, rhs: Vec2) -> Vec2 {
        Vec2 { x: self.x + rhs.x, y: self.y + rhs.y }
    }
}

let a = Vec2 { x: 1.0, y: 2.0 };
let b = Vec2 { x: 3.0, y: 4.0 };
let c = a + b;  // 类型安全：只允许 Vec2 + Vec2
```

关键差异：Python 的 `__add__` 在运行时接受*任何* `other`（你手动检查类型或者得到 `TypeError`）。Rust 的 `Add` trait 在编译时强制执行操作数类型 —— `Vec2 + i32` 是编译错误，除非你显式地 `impl Add<i32> for Vec2`。

---

## 练习

<details>
<summary><strong>🏋️ 练习：泛型 Summary Trait</strong>（点击展开）</summary>

**挑战**：定义一个 trait `Summary`，带方法 `fn summarize(&self) -> String`。为两个结构体实现它：`Article { title: String, body: String }` 和 `Tweet { username: String, content: String }`。然后编写一个函数 `fn notify(item: &impl Summary)` 打印摘要。

<details>
<summary>🔑 解决方案</summary>

```rust
trait Summary {
    fn summarize(&self) -> String;
}

struct Article { title: String, body: String }
struct Tweet { username: String, content: String }

impl Summary for Article {
    fn summarize(&self) -> String {
        format!("{} — {}...", self.title, &self.body[..20.min(self.body.len())])
    }
}

impl Summary for Tweet {
    fn summarize(&self) -> String {
        format!("@{}: {}", self.username, self.content)
    }
}

fn notify(item: &impl Summary) {
    println!("📢 {}", item.summarize());
}

fn main() {
    let article = Article {
        title: "Rust is great".into(),
        body: "Here is why Rust beats Python for systems...".into(),
    };
    let tweet = Tweet {
        username: "rustacean".into(),
        content: "Just shipped my first crate!".into(),
    };
    notify(&article);
    notify(&tweet);
}
```

**关键要点**：`&impl Summary` 是 Python 的 `Protocol` 带 `summarize` 方法的 Rust 等价物。但是 Rust 在编译时检查它 —— 传递没有实现 `Summary` 的类型是编译错误，不是运行时 `AttributeError`。

</details>
</details>
***


