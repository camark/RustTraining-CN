# Rust 生命周期和借用

> **你将学到什么：** Rust 的生命周期系统如何确保引用永远不会悬空 —— 从隐式生命周期到显式标注，再到使大多数代码无需标注的三条省略规则。在进入下一节智能指针之前，理解这里的生命周期至关重要。

- Rust 强制执行单个可变引用和任意数量的不可变引用
    - 任何引用的生命周期必须至少与原始所有生命周期一样长。这些是隐式生命周期，由编译器推断（参见 https://doc.rust-lang.org/nomicon/lifetime-elision.html）
```rust
fn borrow_mut(x: &mut u32) {
    *x = 43;
}
fn main() {
    let mut x = 42;
    let y = &mut x;
    borrow_mut(y);
    let _z = &x; // 允许，因为编译器知道 y 随后未被使用
    //println!("{y}"); // 如果取消注释将无法编译
    borrow_mut(&mut x); // 允许，因为 _z 未使用 
    let z = &x; // Ok —— x 的可变借用在 borrow_mut() 返回后结束
    println!("{z}");
}
```

# Rust 生命周期标注
- 处理多个生命周期时需要显式生命周期标注
    - 生命周期用 `'` 表示，可以是任何标识符（`'a`、`'b`、`'static` 等）
    - 当编译器无法判断引用应该存活多久时需要帮助
- **常见场景**：函数返回引用，但它来自哪个输入？
```rust
#[derive(Debug)]
struct Point {x: u32, y: u32}

// 没有生命周期标注，这无法编译：
// fn left_or_right(pick_left: bool, left: &Point, right: &Point) -> &Point

// 带生命周期标注 —— 所有引用共享相同的生命周期 'a
fn left_or_right<'a>(pick_left: bool, left: &'a Point, right: &'a Point) -> &'a Point {
    if pick_left { left } else { right }
}

// 更复杂：输入有不同的生命周期
fn get_x_coordinate<'a, 'b>(p1: &'a Point, _p2: &'b Point) -> &'a u32 {
    &p1.x  // 返回值生命周期绑定到 p1，不是 p2
}

fn main() {
    let p1 = Point {x: 20, y: 30};
    let result;
    {
        let p2 = Point {x: 42, y: 50};
        result = left_or_right(true, &p1, &p2);
        // 这有效是因为我们在 p2 离开作用域之前使用 result
        println!("Selected: {result:?}");
    }
    // 这不可行 —— result 引用 p2，它现在已消失：
    // println!("After scope: {result:?}");
}
```

# Rust 生命周期标注
- 数据结构中的引用也需要生命周期标注
```rust
use std::collections::HashMap;
#[derive(Debug)]
struct Point {x: u32, y: u32}
struct Lookup<'a> {
    map: HashMap<u32, &'a Point>,
}
fn main() {
    let p = Point{x: 42, y: 42};
    let p1 = Point{x: 50, y: 60};
    let mut m = Lookup {map : HashMap::new()};
    m.map.insert(0, &p);
    m.map.insert(1, &p1);
    {
        let p3 = Point{x: 60, y:70};
        //m.map.insert(3, &p3); // 无法编译
        // p3 在这里被 drop，但 m 会比它活得长
    }
    for (k, v) in m.map {
        println!("{v:?}");
    }
    // m 在这里被 drop
    // p1 和 p 在这里按顺序被 drop
} 
```

# 练习：带生命周期的第一个单词

🟢 **入门** —— 实践中练习生命周期省略

编写一个函数 `fn first_word(s: &str) -> &str`，从字符串返回第一个空白分隔的单词。思考为什么这无需显式生命周期标注就能编译（提示：省略规则 #1 和 #2）。

<details><summary>答案（点击展开）</summary>

```rust
fn first_word(s: &str) -> &str {
    // 编译器应用省略规则：
    // 规则 1：输入 &str 获得生命周期 'a → fn first_word(s: &'a str) -> &str
    // 规则 2：单个输入生命周期 → 输出获得相同的 → fn first_word(s: &'a str) -> &'a str
    match s.find(' ') {
        Some(pos) => &s[..pos],
        None => s,
    }
}

fn main() {
    let text = "hello world foo";
    let word = first_word(text);
    println!("First word: {word}");  // "hello"
    
    let single = "onlyone";
    println!("First word: {}", first_word(single));  // "onlyone"
}
```

</details>

# 练习：带生命周期的切片存储

🟡 **中级** —— 你第一次遇到生命周期标注
- 创建一个结构体，存储对 `&str` 切片的引用
    - 创建一个长 `&str` 并在结构体内部存储来自它的引用切片
    - 编写一个函数，接受结构体并返回包含的切片
```rust
// TODO：创建一个结构体来存储对切片的引用
struct SliceStore {

}
fn main() {
    let s = "This is long string";
    let s1 = &s[0..];
    let s2 = &s[1..2];
    // let slice = struct SliceStore {...};
    // let slice2 = struct SliceStore {...};
}
```

<details><summary>答案（点击展开）</summary>

```rust
struct SliceStore<'a> {
    slice: &'a str,
}

impl<'a> SliceStore<'a> {
    fn new(slice: &'a str) -> Self {
        SliceStore { slice }
    }

    fn get_slice(&self) -> &'a str {
        self.slice
    }
}

fn main() {
    let s = "This is a long string";
    let store1 = SliceStore::new(&s[0..4]);   // "This"
    let store2 = SliceStore::new(&s[5..7]);   // "is"
    println!("store1: {}", store1.get_slice());
    println!("store2: {}", store2.get_slice());
}
// 输出：
// store1: This
// store2: is
```

</details>

---

## 生命周期省略规则深入探讨

C 程序员经常问："如果生命周期如此重要，为什么大多数 Rust 函数没有 `'a` 标注？" 答案是**生命周期省略** —— 编译器应用三条确定性规则自动推断生命周期。

### 三条省略规则

Rust 编译器**按顺序**将这些规则应用于函数签名。如果在应用规则后所有输出生命周期都确定了，则无需标注。

```mermaid
flowchart TD
    A["带引用的<br/>函数签名"] --> R1
    R1["规则 1：每个输入<br/>引用获得自己的<br/>生命周期<br/><br/>fn f(&amp;str, &amp;str)<br/>→ fn f&lt;'a,'b&gt;(&amp;'a str,<br/>&amp;'b str)"]
    R1 --> R2
    R2["规则 2：如果恰好一个<br/>输入生命周期，分配给<br/>所有输出<br/><br/>fn f(&amp;str) → &amp;str<br/>→ fn f&lt;'a&gt;(&amp;'a str)<br/>→ &amp;'a str"]
    R2 --> R3
    R3["规则 3：如果一个输入是<br/>&amp;self 或 &amp;mut self,<br/>分配其生命周期给<br/>所有输出<br/><br/>fn f(&amp;self, &amp;str) → &amp;str<br/>→ fn f&lt;'a&gt;(&amp;'a self, &amp;str)<br/>→ &amp;'a str"]
    R3 --> CHECK{{"所有输出<br/>生命周期<br/>确定了？"}}
    CHECK -->|是 | OK["✅ 无需<br/>标注"]
    CHECK -->|否 | ERR["❌ 编译错误:<br/>必须手动<br/>标注"]
    
    style OK fill:#91e5a3,color:#000
    style ERR fill:#ff6b6b,color:#000
```

### 逐条规则示例

**规则 1** —— 每个输入引用获得自己的生命周期参数：
```rust
// 你写的：
fn first_word(s: &str) -> &str { ... }

// 编译器在规则 1 后看到的：
fn first_word<'a>(s: &'a str) -> &str { ... }
// 只有一个输入生命周期 → 应用规则 2
```

**规则 2** —— 单个输入生命周期传播到所有输出：
```rust
// 规则 2 后：
fn first_word<'a>(s: &'a str) -> &'a str { ... }
// ✅ 所有输出生命周期确定 —— 无需标注！
```

**规则 3** —— `&self` 生命周期传播到输出：
```rust
// 你写的：
impl SliceStore<'_> {
    fn get_slice(&self) -> &str { self.slice }
}

// 编译器在规则 1 + 3 后看到的：
impl SliceStore<'_> {
    fn get_slice<'a>(&'a self) -> &'a str { self.slice }
}
// ✅ 无需标注 —— &self 生命周期用于输出
```

**当省略失败时** —— 你必须标注：
```rust
// 两个输入引用，无 &self → 规则 2 和 3 不适用
// fn longest(a: &str, b: &str) -> &str  ← 无法编译

// 修复：告诉编译器输出从哪个输入借用
fn longest<'a>(a: &'a str, b: &'a str) -> &'a str {
    if a.len() >= b.len() { a } else { b }
}
```

### C 程序员心智模型

在 C 中，每个指针都是独立的 —— 程序员在心理上跟踪每个指针引用哪个分配，编译器完全信任你。在 Rust 中，生命周期使这种跟踪变得**显式且编译器验证**：

| C | Rust | 发生了什么 |
|---|------|-------------|
| `char* get_name(struct User* u)` | `fn get_name(&self) -> &str` | 规则 3 省略：输出从 `self` 借用 |
| `char* concat(char* a, char* b)` | `fn concat<'a>(a: &'a str, b: &'a str) -> &'a str` | 必须标注 —— 两个输入 |
| `void process(char* in, char* out)` | `fn process(input: &str, output: &mut String)` | 无输出引用 —— 无需生命周期 |
| `char* buf; /* 谁拥有这个？ */` | 如果生命周期错误则编译错误 | 编译器捕获悬空指针 |

### `'static` 生命周期

`'static` 意味着引用在**整个程序持续时间**内有效。它是 C 全局变量或字符串字面量的 Rust 等价物：

```rust
// 字符串字面量总是 'static —— 它们生活在二进制文件的只读部分
let s: &'static str = "hello";  // 与 C 中的 static const char* s = "hello"; 相同

// 常量也是 'static
static GREETING: &str = "hello";

// 常见于线程产生的 trait 边界：
fn spawn<F: FnOnce() + Send + 'static>(f: F) { /* ... */ }
// 这里的 'static 意味着："闭包不能借用任何局部变量"
// （要么移动它们进来，或只使用 'static 数据）
```

### 练习：预测省略

🟡 **中级**

对于下面的每个函数签名，预测编译器是否可以省略生命周期。如果不能，添加必要的标注：

```rust
// 1. 编译器能省略吗？
fn trim_prefix(s: &str) -> &str { &s[1..] }

// 2. 编译器能省略吗？
fn pick(flag: bool, a: &str, b: &str) -> &str {
    if flag { a } else { b }
}

// 3. 编译器能省略吗？
struct Parser { data: String }
impl Parser {
    fn next_token(&self) -> &str { &self.data[..5] }
}

// 4. 编译器能省略吗？
fn split_at(s: &str, pos: usize) -> (&str, &str) {
    (&s[..pos], &s[pos..])
}
```

<details><summary>答案（点击展开）</summary>

```rust,ignore
// 1. 是 —— 规则 1 给 s 分配 'a，规则 2 传播到输出
fn trim_prefix(s: &str) -> &str { &s[1..] }

// 2. 否 —— 两个输入引用，无 &self。必须标注：
fn pick<'a>(flag: bool, a: &'a str, b: &'a str) -> &'a str {
    if flag { a } else { b }
}

// 3. 是 —— 规则 1 给 &self 分配 'a，规则 3 传播到输出
impl Parser {
    fn next_token(&self) -> &str { &self.data[..5] }
}

// 4. 是 —— 规则 1 给 s 分配 'a（只有一个输入引用），
//    规则 2 传播到两个输出。两个切片都从 s 借用。
fn split_at(s: &str, pos: usize) -> (&str, &str) {
    (&s[..pos], &s[pos..])
}
```

</details>

