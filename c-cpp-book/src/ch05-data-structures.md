### Rust 数组类型

> **你将学到什么：** Rust 的核心数据结构 —— 数组、元组、切片、字符串、结构体、`Vec` 和 `HashMap`。这是密集的一章；重点理解 `String` vs `&str` 以及结构体如何工作。你将在第 7 章深入 revisiting 引用和借用。

- 数组包含固定数量的同类型元素
    - 像所有其他 Rust 类型一样，数组默认是不可变的（除非使用 mut）
    - 数组使用 [] 索引并进行边界检查。可以使用 len() 方法获取数组长度
```rust
    fn get_index(y : usize) -> usize {
        y+1        
    }
    
    fn main() {
        // 初始化一个 3 元素数组并全部设为 42
        let a : [u8; 3] = [42; 3];
        // 替代语法
        // let a = [42u8, 42u8, 42u8];
        for x in a {
            println!("{x}");
        }
        let y = get_index(a.len());
        // 注释掉下面会导致 panic
        //println!("{}", a[y]);
    }
```

----
### Rust 数组类型（续）
- 数组可以嵌套
    - Rust 有多种内置的打印格式化器。在下面，`:?` 是 `debug` 打印格式化器。`:#?` 格式化器可用于 `pretty print`。这些格式化器可以按类型定制（稍后详细介绍）
```rust
    fn main() {
        let a = [
            [40, 0], // 定义嵌套数组
            [41, 0],
            [42, 1],
        ];
        for x in a {
            println!("{x:?}");
        }
    }
```
----
### Rust 元组
- 元组具有固定大小，可以将任意类型分组为单个复合类型
    - 组成类型可以按其相对位置索引（.0、.1、.2、...）。空元组，即 ()，称为单位值，等同于 void 返回值
    - Rust 支持元组解构，使绑定变量到单个元素变得容易
```rust
fn get_tuple() -> (u32, bool) {
    (42, true)        
}

fn main() {
   let t : (u8, bool) = (42, true);
   let u : (u32, bool) = (43, false);
   println!("{}, {}", t.0, t.1);
   println!("{}, {}", u.0, u.1);
   let (num, flag) = get_tuple(); // 元组解构
   println!("{num}, {flag}");
}
```

### Rust 引用
- Rust 中的引用大致等同于 C 中的指针，但有一些关键区别
    - 在任何时候都可以有任意数量的对变量的只读（不可变）引用。引用不能超出变量作用域（这是一个称为**生命周期**的关键概念；稍后详细讨论）
    - 只允许对可变变量有单个可写（可变）引用，且不能与任何其他引用重叠
```rust
fn main() {
    let mut a = 42;
    {
        let b = &a;
        let c = b;
        println!("{} {}", *b, *c); // 编译器自动解引用 *c
        
        let d = &mut a;
        
        /* 
         * 取消注释下面会导致
         * 程序无法编译，因为 `b` 被使用
         * 而可变引用 `d` 在当前作用域中存活
         * 
         * 你不能在同一作用域中同时使用可变和不可变引用
         */
        // println!("{}", *b);
    }
    let d = &mut a; // Ok：b 和 c 不在作用域中
    *d = 43;
}
```

----
# Rust 切片
- Rust 引用可用于创建数组的子集
    - 与数组不同（数组在编译时确定静态固定长度），切片可以是任意大小。在内部，切片实现为"胖指针"，包含切片的长度和指向原始数组中起始元素的指针
```rust
fn main() {
    let a = [40, 41, 42, 43];
    let b = &a[1..a.len()]; // 从原始数组第二个元素开始的切片
    let c = &a[1..]; // 与上面相同
    let d = &a[..]; // 与 &a[0..] 或 &a[0..a.len()] 相同
    println!("{b:?} {c:?} {d:?}");
}
```
----
# Rust 常量和静态变量
- `const` 关键字可用于定义常数值。常数值在**编译时**求值并内联到程序中
- `static` 关键字用于定义类似 C/C++ 中全局变量的等价物。静态变量有可寻址的内存位置，创建一次并持续整个程序生命周期
```rust
const SECRET_OF_LIFE: u32 = 42;
static GLOBAL_VARIABLE : u32 = 2;
fn main() {
    println!("The secret of life is {}", SECRET_OF_LIFE);
    println!("Value of global variable is {GLOBAL_VARIABLE}")
}
```

----
# Rust 字符串：String vs &str

- Rust 有**两种**字符串类型，服务于不同目的
    - `String` —— 所有、堆分配、可增长（类似 C 的 `malloc` 缓冲区，或 C++ 的 `std::string`）
    - `&str` —— 借用、轻量级引用（类似 C 的 `const char*` 带长度，或 C++ 的 `std::string_view` —— 但 `&str` **经过生命周期检查**，永远不会悬空）
    - 与 C 的空终止字符串不同，Rust 字符串跟踪其长度并保证有效的 UTF-8

> **对于 C++ 开发者：** `String` ≈ `std::string`，`&str` ≈ `std::string_view`。与 `std::string_view` 不同，`&str` 由借用检查器保证在其整个生命周期内有效。

## String vs &str：所有 vs 借用

> **生产模式：** 参见 [JSON 处理：nlohmann::json → serde](ch17-2-avoiding-unchecked-indexing.md#json-handling-nlohmannjson--serde) 了解字符串处理如何在生产代码中与 serde 一起工作。

| **方面** | **C `char*`** | **C++ `std::string`** | **Rust `String`** | **Rust `&str`** |
|------------|--------------|----------------------|-------------------|----------------|
| **内存** | 手动（`malloc`/`free`） | 堆分配、拥有缓冲区 | 堆分配、自动释放 | 借用引用（生命周期检查） |
| **可变性** | 总是通过指针可变 | 可变 | 使用 `mut` 可变 | 总是不可变 |
| **大小信息** | 无（依赖 `'\0'`） | 跟踪长度和容量 | 跟踪长度和容量 | 跟踪长度（胖指针） |
| **编码** | 未指定（通常是 ASCII） | 未指定（通常是 ASCII） | 保证有效的 UTF-8 | 保证有效的 UTF-8 |
| **空终止符** | 必需 | 必需（`c_str()`） | 不使用 | 不使用 |

```rust
fn main() {
    // &str - 字符串切片（借用、不可变、通常是字符串字面量）
    let greeting: &str = "Hello";  // 指向只读内存

    // String - 所有、堆分配、可增长
    let mut owned = String::from(greeting);  // 复制数据到堆
    owned.push_str(", World!");        // 增长字符串
    owned.push('!');                   // 追加单个字符

    // String 和 &str 之间的转换
    let slice: &str = &owned;          // String -> &str（免费，只是借用）
    let owned2: String = slice.to_string();  // &str -> String（分配）
    let owned3: String = String::from(slice); // 与上面相同

    // 字符串连接（注意：+ 消耗左操作数）
    let hello = String::from("Hello");
    let world = String::from(", World!");
    let combined = hello + &world;  // hello 被移动（消耗），world 被借用
    // println!("{hello}");  // 无法编译：hello 被移动

    // 使用 format! 避免移动问题
    let a = String::from("Hello");
    let b = String::from("World");
    let combined = format!("{a}, {b}!");  // a 和 b 都不被消耗

    println!("{combined}");
}
```

## 为什么不能用 `[]` 索引字符串
```rust
fn main() {
    let s = String::from("hello");
    // let c = s[0];  // 无法编译！Rust 字符串是 UTF-8，不是字节数组

    // 安全替代方案：
    let first_char = s.chars().next();           // Option<char>: Some('h')
    let as_bytes = s.as_bytes();                 // &[u8]: 原始 UTF-8 字节
    let substring = &s[0..1];                    // &str: "h"（字节范围，必须是有效的 UTF-8 边界）

    println!("First char: {:?}", first_char);
    println!("Bytes: {:?}", &as_bytes[..5]);
}
```

## 练习：字符串操作

🟢 **入门**
- 编写一个函数 `fn count_words(text: &str) -> usize`，计算字符串中空白分隔的单词数
- 编写一个函数 `fn longest_word(text: &str) -> &str`，返回最长的单词（提示：你需要思考生命周期 —— 为什么返回类型需要是 `&str` 而不是 `String`？）

<details><summary>答案（点击展开）</summary>

```rust
fn count_words(text: &str) -> usize {
    text.split_whitespace().count()
}

fn longest_word(text: &str) -> &str {
    text.split_whitespace()
        .max_by_key(|word| word.len())
        .unwrap_or("")
}

fn main() {
    let text = "the quick brown fox jumps over the lazy dog";
    println!("Word count: {}", count_words(text));       // 9
    println!("Longest word: {}", longest_word(text));     // "jumps"
}
```

</details>

# Rust 结构体
- `struct` 关键字声明用户定义的结构体类型
    - `struct` 成员可以是有名称的，或匿名的（元组结构体）
- 与 C++ 等语言不同，Rust 中没有"数据继承"的概念
```rust
fn main() {
    struct MyStruct {
        num: u32,
        is_secret_of_life: bool,
    }
    let x = MyStruct {
        num: 42,
        is_secret_of_life: true,
    };
    let y = MyStruct {
        num: x.num,
        is_secret_of_life: x.is_secret_of_life,
    };
    let z = MyStruct { num: x.num, ..x }; // .. 表示复制剩余部分
    println!("{} {} {}", x.num, y.is_secret_of_life, z.num);
}
```

# Rust 元组结构体
- Rust 元组结构体类似于元组，单个字段没有名称
    - 与元组一样，单个元素使用 .0、.1、.2 访问。元组结构体的一个常见用例是包装原始类型以创建自定义类型。**这有助于避免混合相同类型的不同值**
```rust
struct WeightInGrams(u32);
struct WeightInMilligrams(u32);
fn to_weight_in_grams(kilograms: u32) -> WeightInGrams {
    WeightInGrams(kilograms * 1000)
}

fn to_weight_in_milligrams(w : WeightInGrams) -> WeightInMilligrams  {
    WeightInMilligrams(w.0 * 1000)
}

fn main() {
    let x = to_weight_in_grams(42);
    let y = to_weight_in_milligrams(x);
    // let z : WeightInGrams = x;  // 无法编译：x 被移动到 to_weight_in_milligrams()
    // let a : WeightInGrams = y;   // 无法编译：类型不匹配（WeightInMilligrams vs WeightInGrams）
}
```


**注意**：`#[derive(...)]` 属性自动为结构体和枚举生成常见的 trait 实现。你将在整个课程中看到它的使用：
```rust
#[derive(Debug, Clone, PartialEq)]
struct Point { x: i32, y: i32 }

fn main() {
    let p = Point { x: 1, y: 2 };
    println!("{:?}", p);           // Debug：有效，因为 #[derive(Debug)]
    let p2 = p.clone();           // Clone：有效，因为 #[derive(Clone)]
    assert_eq!(p, p2);            // PartialEq：有效，因为 #[derive(PartialEq)]
}
```
我们稍后将深入介绍 trait 系统，但 `#[derive(Debug)]` 非常有用，你应该将它添加到几乎每个你创建的 `struct` 和 `enum` 中。

# Rust Vec 类型
- `Vec<T>` 类型实现动态堆分配缓冲区（类似于 C 中手动管理的 `malloc`/`realloc` 数组，或 C++ 的 `std::vector`）
    - 与固定大小的数组不同，`Vec` 可以在运行时增长和缩小
    - `Vec` 拥有其数据并自动管理内存分配/释放
- 常见操作：`push()`、`pop()`、`insert()`、`remove()`、`len()`、`capacity()`
```rust
fn main() {
    let mut v = Vec::new();    // 空向量，类型从使用中推断
    v.push(42);                // 添加元素到末尾 - Vec<i32>
    v.push(43);                
    
    // 安全迭代（首选）
    for x in &v {              // 借用元素，不消耗向量
        println!("{x}");
    }
    
    // 初始化快捷方式
    let mut v2 = vec![1, 2, 3, 4, 5];           // 用于初始化的宏
    let v3 = vec![0; 10];                       // 10 个零
    
    // 安全访问方法（首选于索引）
    match v2.get(0) {
        Some(first) => println!("First: {first}"),
        None => println!("Empty vector"),
    }
    
    // 有用的方法
    println!("Length: {}, Capacity: {}", v2.len(), v2.capacity());
    if let Some(last) = v2.pop() {             // 移除并返回最后一个元素
        println!("Popped: {last}");
    }
    
    // 危险：直接索引（可能 panic！）
    // println!("{}", v2[100]);  // 将在运行时 panic
}
```
> **生产模式：** 参见 [避免未检查的索引](ch17-2-avoiding-unchecked-indexing.md#avoiding-unchecked-indexing) 了解来自生产 Rust 代码的安全 `.get()` 模式。

# Rust HashMap 类型
- `HashMap` 实现通用 `key` -> `value` 查找（也称为 `dictionary` 或 `map`）
```rust
fn main() {
    use std::collections::HashMap;  // 需要显式导入，与 Vec 不同
    let mut map = HashMap::new();       // 分配空的 HashMap
    map.insert(40, false);  // 类型推断为 int -> bool
    map.insert(41, false);
    map.insert(42, true);
    for (key, value) in map {
        println!("{key} {value}");
    }
    let map = HashMap::from([(40, false), (41, false), (42, true)]);
    if let Some(x) = map.get(&43) {
        println!("43 was mapped to {x:?}");
    } else {
        println!("No mapping was found for 43");
    }
    let x = map.get(&43).or(Some(&false));  // 如果键未找到则使用默认值
    println!("{x:?}"); 
}
```

# 练习：Vec 和 HashMap

🟢 **入门**
- 创建一个 `HashMap<u32, bool>`，带有一些条目（确保一些值是 `true`，其他是 `false`）。遍历 hashmap 中的所有元素，将键放入一个 `Vec`，将值放入另一个

<details><summary>答案（点击展开）</summary>

```rust
use std::collections::HashMap;

fn main() {
    let map = HashMap::from([(1, true), (2, false), (3, true), (4, false)]);
    let mut keys = Vec::new();
    let mut values = Vec::new();
    for (k, v) in &map {
        keys.push(*k);
        values.push(*v);
    }
    println!("Keys:   {keys:?}");
    println!("Values: {values:?}");

    // 替代方案：使用迭代器和 unzip()
    let (keys2, values2): (Vec<u32>, Vec<bool>) = map.into_iter().unzip();
    println!("Keys (unzip):   {keys2:?}");
    println!("Values (unzip): {values2:?}");
}
```

</details>

---

## 深入探讨：C++ 引用 vs Rust 引用

> **对于 C++ 开发者：** C++ 程序员经常假设 Rust `&T` 像 C++ `T&` 一样工作。虽然表面上相似，但有关键区别会导致混淆。C 开发者可以跳过本节 —— Rust 引用将在 [所有权和借用](ch07-ownership-and-borrowing.md) 中介绍。

#### 1. 无右值引用或通用引用

在 C++ 中，`&&` 有两种含义，取决于上下文：

```cpp
// C++：&& 表示不同的东西：
int&& rref = 42;           // 右值引用 —— 绑定到临时对象
void process(Widget&& w);   // 右值引用 —— 调用者必须 std::move

// 通用（转发）引用 —— 推导的模板上下文：
template<typename T>
void forward(T&& arg) {     // 不是右值引用！推导为 T& 或 T&&
    inner(std::forward<T>(arg));  // 完美转发
}
```

**在 Rust 中：这些都不存在。** `&&` 只是逻辑 AND 运算符。

```rust
// Rust：&& 只是布尔 AND
let a = true && false; // false

// Rust 没有右值引用、没有通用引用、没有完美转发。
// 取而代之的是：
//   - 对于非 Copy 类型，移动是默认（无需 std::move）
//   - 泛型 + trait 边界取代通用引用
//   - 无临时绑定区别 —— 值就是值

fn process(w: Widget) { }      // 获取所有权（类似 C++ 值参数 + 隐式移动）
fn process_ref(w: &Widget) { } // 不变地借用（类似 C++ const T&）
fn process_mut(w: &mut Widget) { } // 可变地借用（类似 C++ T&，但独占）
```

| C++ 概念 | Rust 等价物 | 注意 |
|-------------|-----------------|-------|
| `T&`（左值引用） | `&T` 或 `&mut T` | Rust 分为共享 vs 独占 |
| `T&&`（右值引用） | 就是 `T` | 按值获取 = 获取所有权 |
| 模板中的 `T&&`（通用引用） | `impl Trait` 或 `<T: Trait>` | 泛型取代转发 |
| `std::move(x)` | `x`（直接使用） | 移动是默认 |
| `std::forward<T>(x)` | 无需等价物 | 无通用引用可转发 |

#### 2. 移动是按位的 —— 无移动构造函数

在 C++ 中，移动是*用户定义的操作*（移动构造函数/移动赋值）。在 Rust 中，移动总是值的**按位 memcpy**，并且源失效：

```rust
// Rust 移动 = 复制字节，标记源为无效
let s1 = String::from("hello");
let s2 = s1; // s1 的字节复制到 s2 的栈槽
              // s1 现在无效 —— 编译器强制执行
// println!("{s1}"); // ❌ 编译错误：value used after move
```

```cpp
// C++ 移动 = 调用移动构造函数（用户定义！）
std::string s1 = "hello";
std::string s2 = std::move(s1); // 调用 string 的移动构造函数
// s1 现在是"有效但未指定状态"的僵尸
std::cout << s1; // 编译！打印... 某个东西（通常是空字符串）
```

**结果**：
- Rust 没有 Rule of Five（无需定义复制构造函数、移动构造函数、复制赋值、移动赋值、析构函数）
- 无移动后的"僵尸"状态 —— 编译器简单地防止访问
- 移动无需 `noexcept` 考虑 —— 按位复制不会抛出

#### 3. 自动解引用：编译器看穿间接层

Rust 通过 `Deref` trait 自动解引用多层指针/包装器。这在 C++ 中没有等价物：

```rust
use std::sync::{Arc, Mutex};

// 嵌套包装：Arc<Mutex<Vec<String>>>
let data = Arc::new(Mutex::new(vec!["hello".to_string()]));

// 在 C++ 中，你需要在每一层显式解锁和手动解引用。
// 在 Rust 中，编译器自动解引用通过 Arc → Mutex → MutexGuard → Vec：
let guard = data.lock().unwrap(); // Arc 自动解引用到 Mutex
let first: &str = &guard[0];      // MutexGuard→Vec (Deref)、Vec[0] (Index)、
                                   // &String→&str (Deref 强制)
println!("First: {first}");

// 方法调用也自动解引用：
let boxed_string = Box::new(String::from("hello"));
println!("Length: {}", boxed_string.len());  // Box→String，然后 String::len()
// 无需 (*boxed_string).len() 或 boxed_string->len()
```

**Deref 强制**也适用于函数参数 —— 编译器插入解引用以匹配类型：

```rust
fn greet(name: &str) {
    println!("Hello, {name}");
}

fn main() {
    let owned = String::from("Alice");
    let boxed = Box::new(String::from("Bob"));
    let arced = std::sync::Arc::new(String::from("Carol"));

    greet(&owned);  // &String → &str  (1 次解引用强制)
    greet(&boxed);  // &Box<String> → &String → &str  (2 次解引用强制)
    greet(&arced);  // &Arc<String> → &String → &str  (2 次解引用强制)
    greet("Dave");  // 已经是 &str —— 无需强制
}
// 在 C++ 中，你需要 .c_str() 或每个情况的显式转换。
```

**Deref 链**：当你调用 `x.method()` 时，Rust 的方法解析
尝试接收者类型 `T`，然后 `&T`，然后 `&mut T`。如果没有匹配，它
通过 `Deref` trait 解引用并对目标类型重复。
这继续通过多层 —— 这就是为什么 `Box<Vec<T>>`
"就是有效"像 `Vec<T>` 一样。Deref *强制*（用于函数参数）
是单独但相关的机制，通过链接 `Deref` impl 自动将 `&Box<String>`
转换为 `&str`。

#### 4. 无空引用，无可选引用

```cpp
// C++：引用不能为空，但指针可以为，区别模糊
Widget& ref = *ptr;  // 如果 ptr 为空 → UB
Widget* opt = nullptr;  // 通过指针的"可选"引用
```

```rust
// Rust：引用总是有效的 —— 由借用检查器保证
// 在安全代码中无法创建空或悬空引用
let r: &i32 = &42; // 总是有效

// "可选引用"是显式的：
let opt: Option<&Widget> = None; // 清晰的意图，无空指针
if let Some(w) = opt {
    w.do_something(); // 仅当存在时可到达
}
```

#### 5. 引用不能重新绑定

```cpp
// C++：引用是别名 —— 不能重新绑定
int a = 1, b = 2;
int& r = a;
r = b;  // 这将 b 的值赋给 a —— 它不重新绑定 r！
// a 现在是 2，r 仍然引用 a
```

```rust
// Rust：let 绑定可以遮蔽，但引用遵循不同的规则
let a = 1;
let b = 2;
let r = &a;
// r = &b;   // ❌ 不能赋值给不可变变量
let r = &b;  // ✅ 但你可以用新绑定遮蔽 r
             // 旧绑定消失，不是重新绑定

// 使用 mut：
let mut r = &a;
r = &b;      // ✅ r 现在指向 b —— 这是重新绑定（不是通过赋值）
```

> **心智模型**：在 C++ 中，引用是一个对象的永久别名。
> 在 Rust 中，引用是一个值（带生命周期保证的指针），
> 遵循正常的变量绑定规则 —— 默认不可变，仅当声明为 `mut` 时可重新绑定。

