# Rust 枚举类型

> **你将学到什么：** Rust 枚举作为可区分联合（正确的标签联合）、`match` 用于穷尽模式匹配，以及枚举如何取代 C++ 类层次结构和 C 标签联合，带有编译器强制执行的安全性。

- 枚举类型是可区分联合，即它们是多个可能不同类型的和类型，带有一个标识特定变体的标签
    - 对于 C 开发者：Rust 中的枚举可以携带数据（正确的标签联合 —— 编译器跟踪哪个变体是活跃的）
    - 对于 C++ 开发者：Rust 枚举像 `std::variant`，但带有穷尽模式匹配、无 `std::get` 异常、无 `std::visit` 样板
    - `enum` 的大小是最大可能类型的大小。单个变体彼此不相关，可以有不同的类型
    - `enum` 类型是语言最强大的特性之一 —— 它们取代 C++ 中的整个类层次结构（案例研究中有更多介绍）
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let a = Numbers::Zero;
    let b = Numbers::SmallNumber(42);
    let c : Numbers = a; // Ok -- a 的类型是 Numbers
    let d : Numbers = b; // Ok -- b 的类型是 Numbers
}
```
----
# Rust match 语句
- Rust 的 `match` 相当于 C 的"switch"加强版
    - `match` 可用于简单数据类型、`struct`、`enum` 的模式匹配
    - `match` 语句必须是穷尽的，即它们必须覆盖给定 `type` 的所有可能情况。`_` 可用作"其他所有"情况的通配符
    - `match` 可以产生一个值，但所有分支（`=>`）必须返回相同类型的值

```rust
fn main() {
    let x = 42;
    // 在这种情况下，_ 覆盖除了明确列出的所有数字
    let is_secret_of_life = match x {
        42 => true, // 返回类型是布尔值
        _ => false, // 返回类型布尔值
        // 这无法编译因为返回类型不是布尔
        // _ => 0  
    };
    println!("{is_secret_of_life}");
}
```

# Rust match 语句
- `match` 支持范围、布尔过滤器和 `if` 守卫语句
```rust
fn main() {
    let x = 42;
    match x {
        // 注意 =41 确保包含范围
        0..=41 => println!("Less than the secret of life"),
        42 => println!("Secret of life"),
        _ => println!("More than the secret of life"),
    }
    let y = 100;
    match y {
        100 if x == 43 => println!("y is 100% not secret of life"),
        100 if x == 42 => println!("y is 100% secret of life"),
        _ => (),    // 什么都不做
    }
}
```

# Rust match 语句
- `match` 和 `enum` 经常组合使用
    - match 语句可以"绑定"包含的值到变量。如果不关心值则使用 `_`
    - `matches!` 宏可用于匹配特定变体
```rust
fn main() {
    enum Numbers {
        Zero,
        SmallNumber(u8),
        BiggerNumber(u32),
        EvenBiggerNumber(u64),
    }
    let b = Numbers::SmallNumber(42);
    match b {
        Numbers::Zero => println!("Zero"),
        Numbers::SmallNumber(value) => println!("Small number {value}"),
        Numbers::BiggerNumber(_) | Numbers::EvenBiggerNumber(_) => println!("Some BiggerNumber or EvenBiggerNumber"),
    }
    
    // 布尔测试特定变体
    if matches!(b, Numbers::Zero | Numbers::SmallNumber(_)) {
        println!("Matched Zero or small number");
    }
}
```

# Rust match 语句
- `match` 还可以使用解构和切片进行匹配
```rust
fn main() {
    struct Foo {
        x: (u32, bool),
        y: u32
    }
    let f = Foo {x: (42, true), y: 100};
    match f {
        // 捕获 x 的值到名为 tuple 的变量
        Foo{y: 100, x : tuple} => println!("Matched x: {tuple:?}"),
        _ => ()
    }
    let a = [40, 41, 42];
    match a {
        // 切片的最后一个元素必须是 42。@ 用于绑定匹配
        [rest @ .., 42] => println!("{rest:?}"),
        // 切片的第一个元素必须是 42。@ 用于绑定匹配
        [42, rest @ ..] => println!("{rest:?}"),
        _ => (),
    }
}
```

# 练习：使用 match 和 enum 实现加法和减法

🟢 **入门**

- 编写一个函数实现无符号 64 位数字的算术运算
- **步骤 1**：定义操作枚举：
```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}
```
- **步骤 2**：定义结果枚举：
```rust
enum CalcResult {
    Ok(u64),                    // 成功结果
    Invalid(String),            // 无效操作的错误消息
}
```
- **步骤 3**：实现 `calculate(op: Operation) -> CalcResult`
    - 对于 Add：返回 Ok(sum)
    - 对于 Subtract：如果第一个 >= 第二个则返回 Ok(difference)，否则返回 Invalid("Underflow")
- **提示**：在函数中使用模式匹配：
```rust
match op {
    Operation::Add(a, b) => { /* your code */ },
    Operation::Subtract(a, b) => { /* your code */ },
}
```

<details><summary>答案（点击展开）</summary>

```rust
enum Operation {
    Add(u64, u64),
    Subtract(u64, u64),
}

enum CalcResult {
    Ok(u64),
    Invalid(String),
}

fn calculate(op: Operation) -> CalcResult {
    match op {
        Operation::Add(a, b) => CalcResult::Ok(a + b),
        Operation::Subtract(a, b) => {
            if a >= b {
                CalcResult::Ok(a - b)
            } else {
                CalcResult::Invalid("Underflow".to_string())
            }
        }
    }
}

fn main() {
    match calculate(Operation::Add(10, 20)) {
        CalcResult::Ok(result) => println!("10 + 20 = {result}"),
        CalcResult::Invalid(msg) => println!("Error: {msg}"),
    }
    match calculate(Operation::Subtract(5, 10)) {
        CalcResult::Ok(result) => println!("5 - 10 = {result}"),
        CalcResult::Invalid(msg) => println!("Error: {msg}"),
    }
}
// 输出：
// 10 + 20 = 30
// Error: Underflow
```

</details>

# Rust 关联方法
- `impl` 可以为 `struct`、`enum` 等类型定义关联方法
    - 方法可以选择以 `self` 作为参数。`self` 在概念上类似于在 C 中将指向结构体的指针作为第一个参数传递，或在 C++ 中的 `this`
    - `self` 的引用可以是不变的（默认：`&self`）、可变的（`&mut self`）或 `self`（转移所有权）
    - `Self` 关键字可用作暗示类型的快捷方式
```rust
struct Point {x: u32, y: u32}
impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point {x, y}
    }
    fn increment_x(&mut self) {
        self.x += 1;
    }
}
fn main() {
    let mut p = Point::new(10, 20);
    p.increment_x();
}
```

# 练习：Point 加法和变换

🟡 **中级** —— 需要从方法签名中理解移动 vs 借用
- 为 `Point` 实现以下关联方法
    - `add()` 将接受另一个 `Point` 并就地递增 x 和 y 值（提示：使用 `&mut self`）
    - `transform()` 将消耗现有的 `Point`（提示：使用 `self`）并返回一个新的 `Point`，通过对 x 和 y 平方

<details><summary>答案（点击展开）</summary>

```rust
struct Point { x: u32, y: u32 }

impl Point {
    fn new(x: u32, y: u32) -> Self {
        Point { x, y }
    }
    fn add(&mut self, other: &Point) {
        self.x += other.x;
        self.y += other.y;
    }
    fn transform(self) -> Point {
        Point { x: self.x * self.x, y: self.y * self.y }
    }
}

fn main() {
    let mut p1 = Point::new(2, 3);
    let p2 = Point::new(10, 20);
    p1.add(&p2);
    println!("After add: x={}, y={}", p1.x, p1.y);           // x=12, y=23
    let p3 = p1.transform();
    println!("After transform: x={}, y={}", p3.x, p3.y);     // x=144, y=529
    // p1 不再可访问 —— transform() 消耗了它
}
```

</details>

----

