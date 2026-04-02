# Rust From 和 Into traits

> **你将学到什么：** Rust 的类型转换 traits —— `From<T>` 和 `Into<T>` 用于不可失败的转换，`TryFrom` 和 `TryInto` 用于可失败的转换。实现 `From` 并免费获得 `Into`。取代 C++ 转换运算符和构造函数。

- ```From``` 和 ```Into``` 是互补的 traits，用于促进类型转换
- 类型通常实现 ```From``` trait。```String::from()``` 从 "&str" 转换为 ```String```，编译器可以自动派生 ```&str.into```
```rust
struct Point {x: u32, y: u32}
// 从元组构造 Point
impl From<(u32, u32)> for Point {
    fn from(xy : (u32, u32)) -> Self {
        Point {x : xy.0, y: xy.1}       // 使用元组元素构造 Point
    }
}
fn main() {
    let s = String::from("Rust");
    let x = u32::from(true);
    let p = Point::from((40, 42));
    // let p : Point = (40.42)::into(); // 上述的交替形式
    println!("s: {s} x:{x} p.x:{} p.y {}", p.x, p.y);   
}
```

# 练习：From 和 Into
- 为 ```Point``` 实现一个 ```From``` trait 转换为名为 ```TransposePoint``` 的类型。```TransposePoint``` 交换 ```Point``` 的 ```x``` 和 ```y``` 元素

<details><summary>答案（点击展开）</summary>

```rust
struct Point { x: u32, y: u32 }
struct TransposePoint { x: u32, y: u32 }

impl From<Point> for TransposePoint {
    fn from(p: Point) -> Self {
        TransposePoint { x: p.y, y: p.x }
    }
}

fn main() {
    let p = Point { x: 10, y: 20 };
    let tp = TransposePoint::from(p);
    println!("转置：x={}, y={}", tp.x, tp.y);  // x=20, y=10

    // 使用 .into() —— From 实现后自动有效
    let p2 = Point { x: 3, y: 7 };
    let tp2: TransposePoint = p2.into();
    println!("转置：x={}, y={}", tp2.x, tp2.y);  // x=7, y=3
}
// 输出：
// 转置：x=20, y=10
// 转置：x=7, y=3
```

</details>

# Rust Default trait
- ```Default``` 可用于为类型实现默认值
    - 类型可以使用 ```Derive``` 宏与 ```Default``` 或提供自定义实现
```rust
#[derive(Default, Debug)]
struct Point {x: u32, y: u32}
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = Point::default();   // 创建 Point{0, 0}
    println!("{x:?}");
    let y = CustomPoint::default();
    println!("{y:?}");
}
```

### Rust Default trait
- ```Default``` trait 有多个用例包括
    - 执行部分复制并对其余部分使用默认初始化
    - 在 ```unwrap_or_default()``` 等方法中作为 ```Option``` 类型的默认替代
```rust
#[derive(Debug)]
struct CustomPoint {x: u32, y: u32}
impl Default for CustomPoint {
    fn default() -> Self {
        CustomPoint {x: 42, y: 42}
    }
}
fn main() {
    let x = CustomPoint::default();
    // 覆盖 y，但将其余元素保留为默认值
    let y = CustomPoint {y: 43, ..CustomPoint::default()};
    println!("{x:?} {y:?}");
    let z : Option<CustomPoint> = None;
    // 尝试将 unwrap_or_default() 改为 unwrap()
    println!("{:?}", z.unwrap_or_default());
}
```

### 其他 Rust 类型转换
- Rust 不支持隐式类型转换，```as``` 可用于 ```显式``` 转换
- ```as``` 应该少用，因为它受数据丢失的影响（如 narrowing 等）。一般来说，最好尽可能使用 ```into()``` 或 ```from()```
```rust
fn main() {
    let f = 42u8;
    // let g : u32 = f;    // 无法编译
    let g = f as u32;      // OK，但不推荐。受 narrowing 规则约束
    let g : u32 = f.into(); // 最推荐的形式；不可失败且由编译器检查
    //let k : u8 = f.into();  // 无法编译；narrowing 可能导致数据丢失
    
    // 尝试 narrowing 操作需要使用 try_into
    if let Ok(k) = TryInto::<u8>::try_into(g) {
        println!("{k}");
    }
}
```


