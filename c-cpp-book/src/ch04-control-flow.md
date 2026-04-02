# Rust if 关键字

> **你将学到什么：** Rust 的控制流结构 —— `if`/`else` 作为表达式、`loop`/`while`/`for`、`match`，以及它们与 C/C++ 对应物的区别。关键洞察：大多数 Rust 控制流返回值。

- 在 Rust 中，`if` 实际上是一个表达式，即它可以用于赋值，但也表现得像语句。[▶ 尝试](https://play.rust-lang.org/)

```rust
fn main() {
    let x = 42;
    if x < 42 {
        println!("Smaller than the secret of life");
    } else if x == 42 {
        println!("Is equal to the secret of life");
    } else {
        println!("Larger than the secret of life");
    }
    let is_secret_of_life = if x == 42 {true} else {false};
    println!("{}", is_secret_of_life);
}
```

# Rust 使用 while 和 for 循环
- `while` 关键字用于在表达式为真时循环
```rust
fn main() {
    let mut x = 40;
    while x != 42 {
        x += 1;
    }
}
```
- `for` 关键字可用于迭代范围
```rust
fn main() {
    // 不会打印 43；使用 40..=43 包含最后一个元素
    for x in 40..43 {
        println!("{}", x);
    } 
}
```

# Rust 使用 loop 循环
- `loop` 关键字创建无限循环，直到遇到 `break`
```rust
fn main() {
    let mut x = 40;
    // 改为 'here: loop' 指定循环的可选标签
    loop {
        if x == 42 {
            break; // 使用 break x; 返回 x 的值
        }
        x += 1;
    }
}
```
- `break` 语句可以包含一个可选表达式，用于赋值 `loop` 表达式的值
- `continue` 关键字可用于返回 `loop` 的顶部
- 循环标签可以与 `break` 或 `continue` 一起使用，在处理嵌套循环时很有用

# Rust 表达式块
- Rust 表达式块只是用 `{}` 包围的表达式序列。求值结果就是块中的最后一个表达式
```rust
fn main() {
    let x = {
        let y = 40;
        y + 2 // 注意：必须省略;
    };
    // 注意 Python 风格的打印
    println!("{x}");
}
```
- Rust 风格是在函数中使用这个来省略 `return` 关键字
```rust
fn is_secret_of_life(x: u32) -> bool {
    // 与 if x == 42 {true} else {false} 相同
    x == 42 // 注意：必须省略;
}
fn main() {
    println!("{}", is_secret_of_life(42));
}
```

