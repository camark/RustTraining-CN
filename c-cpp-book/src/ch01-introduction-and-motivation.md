# 演讲者介绍和总体方法

> **你将学到什么：** 课程结构、交互式格式，以及熟悉的 C/C++ 概念如何映射到 Rust 等价物。本章设定期望并为你提供一个本书其余部分的路线图。

- 演讲者介绍
    - 微软 SCHIE（硅和云硬件基础设施工程）团队首席固件架构师
    - 行业资深人士，专长于安全、系统编程（固件、操作系统、管理程序）、CPU 和平台架构以及 C++ 系统
    - 2017 年开始用 Rust 编程（@AWS EC2），从此爱上了这门语言
- 本课程尽可能保持互动
    - 假设：你了解 C、C++ 或两者
    - 示例经过精心设计，将熟悉的概念映射到 Rust 等价物
    - **请随时在任何时候提出澄清性问题**
- 演讲者期待与团队持续互动

# 选择 Rust 的理由
> **想直接看代码？** 跳到 [看代码](ch02-getting-started.md#enough-talk-already-show-me-some-code)

无论你来自 C 还是 C++，核心痛点是一样的：内存安全 bug 可以干净地编译，但在运行时崩溃、损坏或泄漏。

- 超过 **70% 的 CVE** 是由内存安全问题引起的 —— 缓冲区溢出、悬空指针、释放后使用
- C++ 的 `shared_ptr`、`unique_ptr`、RAII 和移动语义是朝正确方向迈出的步骤，但它们只是**权宜之计，而非根治** —— 它们让移动后使用、引用循环、迭代器失效和异常安全缺口保持大开
- Rust 提供你依赖 C/C++ 的性能，但具有**编译时保证**的安全性

> **📖 深入探讨：** 参见 [为什么 C/C++ 开发者需要 Rust](ch01-1-why-c-cpp-developers-need-rust.md) 了解具体的漏洞示例、Rust 消除的完整列表，以及为什么 C++ 智能指针不够

----

# Rust 如何解决这些问题？

## 缓冲区溢出和边界违规
- 所有 Rust 数组、切片和字符串都有明确的边界关联。编译器插入检查以确保任何边界违规导致**运行时崩溃**（Rust 术语中的 panic）—— 绝不是未定义行为

## 悬空指针和引用
- Rust 引入生命周期和借用检查来在**编译时**消除悬空引用
- 没有悬空指针，没有释放后使用 —— 编译器根本不会让你这样做

## 移动后使用
- Rust 的所有权系统使移动具有**破坏性** —— 一旦你移动了一个值，编译器**拒绝**让你使用原始值。没有僵尸对象，没有"有效但未指定状态"

## 资源管理
- Rust 的 `Drop` trait 是正确的 RAII —— 编译器在资源离开作用域时自动释放它们，并**防止移动后使用**，这是 C++ RAII 无法做到的
- 不需要 Rule of Five（无需定义复制构造函数、移动构造函数、复制赋值、移动赋值、析构函数）

## 错误处理
- Rust 没有异常。所有错误都是值（`Result<T, E>`），使错误处理显式并在类型签名中可见

## 迭代器失效
- Rust 的借用检查器**禁止在迭代集合时修改它**。你根本无法写出困扰 C++ 代码库的 bug：
```rust
// Rust 等价物：在迭代期间删除：retain()
pending_faults.retain(|f| f.id != fault_to_remove.id);

// 或：收集到新的 Vec 中（函数式风格）
let remaining: Vec<_> = pending_faults
    .into_iter()
    .filter(|f| f.id != fault_to_remove.id)
    .collect();
```

## 数据竞争
- 类型系统通过 `Send` 和 `Sync` traits 在**编译时**防止数据竞争

## 内存安全可视化

### Rust 所有权 —— 本质安全

```rust
fn safe_rust_ownership() {
    // 移动是破坏性的：原始值消失
    let data = vec![1, 2, 3];
    let data2 = data;           // 发生移动
    // data.len();              // 编译错误：值在移动后使用
    
    // 借用：安全共享访问
    let owned = String::from("Hello, World!");
    let slice: &str = &owned;  // 借用 —— 无分配
    println!("{}", slice);     // 始终安全
    
    // 不可能有悬空引用
    /*
    let dangling_ref;
    {
        let temp = String::from("temporary");
        dangling_ref = &temp;  // 编译错误：temp 活得不够长
    }
    */
}
```

```mermaid
graph TD
    A[Rust 所有权安全] --> B[破坏性移动]
    A --> C[自动内存管理]
    A --> D[编译时生命周期检查]
    A --> E[无异常 —— Result 类型]
    
    B --> B1["移动后使用是编译错误"]
    B --> B2["无僵尸对象"]
    
    C --> C1["Drop trait = 正确的 RAII"]
    C --> C2["不需要 Rule of Five"]
    
    D --> D1["借用检查器防止悬空"]
    D --> D2["引用始终有效"]
    
    E --> E1["Result<T,E> —— 错误在类型中"]
    E --> E2["? 运算符用于传播"]
    
    style A fill:#51cf66,color:#000
    style B fill:#91e5a3,color:#000
    style C fill:#91e5a3,color:#000
    style D fill:#91e5a3,color:#000
    style E fill:#91e5a3,color:#000
```

## 内存布局：Rust 引用

```mermaid
graph TD
    RM1[栈] --> RP1["&i32 引用"]
    RM2[栈/堆] --> RV1["i32 值 = 42"]
    RP1 -.->|"安全引用 —— 生命周期检查"| RV1
    RM3[借用检查器] --> RC1["编译时防止悬空引用"]
    
    style RC1 fill:#51cf66,color:#000
    style RP1 fill:#91e5a3,color:#000
```

### `Box<T>` 堆分配可视化

```rust
fn box_allocation_example() {
    // 栈分配
    let stack_value = 42;
    
    // 使用 Box 进行堆分配
    let heap_value = Box::new(42);
    
    // 移动所有权
    let moved_box = heap_value;
    // heap_value 不再可访问
}
```

```mermaid
graph TD
    subgraph "栈帧"
        SV["stack_value: 42"]
        BP["heap_value: Box<i32>"]
        BP2["moved_box: Box<i32>"]
    end
    
    subgraph "堆"
        HV["42"]
    end
    
    BP -->|"拥有"| HV
    BP -.->|"移动所有权"| BP2
    BP2 -->|"现在拥有"| HV
    
    subgraph "移动后"
        BP_X["heap_value: [警告] 已移动"]
        BP2_A["moved_box: Box<i32>"]
    end
    
    BP2_A -->|"拥有"| HV
    
    style BP_X fill:#ff6b6b,color:#000
    style HV fill:#91e5a3,color:#000
    style BP2_A fill:#51cf66,color:#000
```

## 切片操作可视化

```rust
fn slice_operations() {
    let data = vec![1, 2, 3, 4, 5, 6, 7, 8];
    
    let full_slice = &data[..];        // [1,2,3,4,5,6,7,8]
    let partial_slice = &data[2..6];   // [3,4,5,6]
    let from_start = &data[..4];       // [1,2,3,4]
    let to_end = &data[3..];           // [4,5,6,7,8]
}
```

```mermaid
graph TD
    V["Vec: [1, 2, 3, 4, 5, 6, 7, 8]"]
    V --> FS["&data[..] → 所有元素"]
    V --> PS["&data[2..6] → [3, 4, 5, 6]"]
    V --> SS["&data[..4] → [1, 2, 3, 4]"]
    V --> ES["&data[3..] → [4, 5, 6, 7, 8]"]
    
    style V fill:#e3f2fd,color:#000
    style FS fill:#91e5a3,color:#000
    style PS fill:#91e5a3,color:#000
    style SS fill:#91e5a3,color:#000
    style ES fill:#91e5a3,color:#000
```

# Rust 的其他独特卖点和特性
- 线程间无数据竞争（编译时 `Send`/`Sync` 检查）
- 无移动后使用（不同于 C++ `std::move` 留下僵尸对象）
- 无未初始化变量
    - 所有变量在使用前必须初始化
- 无简单的内存泄漏
    - `Drop` trait = 正确的 RAII，无需 Rule of Five
    - 编译器在内存离开作用域时自动释放
- 无忘记解锁的互斥锁
    - 锁 guard 是访问数据的*唯一*方式（`Mutex<T>` 包装数据，而不是访问）
- 无异常处理的复杂性
    - 错误是值（`Result<T, E>`），在函数签名中可见，用 `?` 传播
- 出色的类型推断、枚举、模式匹配、零成本抽象支持
- 内置依赖管理、构建、测试、格式化、linting 支持
    - `cargo` 取代 make/CMake + lint + test 框架

# 快速参考：Rust vs C/C++

| **概念** | **C** | **C++** | **Rust** | **主要区别** |
|-------------|-------|---------|----------|-------------------|
| 内存管理 | `malloc()/free()` | `unique_ptr`、`shared_ptr` | `Box<T>`、`Rc<T>`、`Arc<T>` | 自动，无循环 |
| 数组 | `int arr[10]` | `std::vector<T>`、`std::array<T>` | `Vec<T>`、`[T; N]` | 默认边界检查 |
| 字符串 | `char*` 带`\0` | `std::string`、`string_view` | `String`、`&str` | UTF-8 保证，生命周期检查 |
| 引用 | `int* ptr` | `T&`、`T&&`（移动） | `&T`、`&mut T` | 借用检查、生命周期 |
| 多态 | 函数指针 | 虚函数、继承 | Traits、trait 对象 | 组合优于继承 |
| 泛型编程 | 宏（`void*`） | 模板 | 泛型 + trait 边界 | 更好的错误信息 |
| 错误处理 | 返回码、`errno` | 异常、`std::optional` | `Result<T, E>`、`Option<T>` | 无隐藏控制流 |
| NULL/空安全 | `ptr == NULL` | `nullptr`、`std::optional<T>` | `Option<T>` | 强制空检查 |
| 线程安全 | 手动（pthreads） | 手动同步 | 编译时保证 | 数据竞争不可能 |
| 构建系统 | Make、CMake | CMake、Make 等 | Cargo | 集成工具链 |
| 未定义行为 | 运行时崩溃 | 微妙的 UB（有符号溢出、别名） | 编译时错误 | 安全保证 |

