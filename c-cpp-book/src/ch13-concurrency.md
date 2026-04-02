# Rust 并发

> **你将学到什么：** Rust 的并发模型 —— 线程、`Send`/`Sync` 标记 trait、`Mutex<T>`、`Arc<T>`、channel，以及编译器如何在编译时防止数据竞争。不为你不使用的线程安全产生运行时开销。

- Rust 对并发有内置支持，类似于 C++ 中的 `std::thread`
    - 关键区别：Rust 通过 `Send` 和 `Sync` 标记 trait**在编译时防止数据竞争**
    - 在 C++ 中，不带 mutex 在线程间共享 `std::vector` 是 UB 但可以编译。在 Rust 中，无法编译。
    - Rust 中的 `Mutex<T>` 包装**数据**，而不仅仅是访问 —— 没有锁定你根本无法读取数据
- `thread::spawn()` 可用于创建单独的线程，并行执行闭包 `||`
```rust
use std::thread;
use std::time::Duration;
fn main() {
    let handle = thread::spawn(|| {
        for i in 0..10 {
            println!("线程中的计数：{i}!");
            thread::sleep(Duration::from_millis(5));
        }
    });

    for i in 0..5 {
        println!("主线程：{i}");
        thread::sleep(Duration::from_millis(5));
    }

    handle.join().unwrap(); // handle.join() 确保生成的线程退出
}
```

# Rust 并发
- 在需要从环境借用的情况下可以使用 ```thread::scope()```。这是有效的，因为 ```thread::scope``` 等待内部线程返回
- 不使用 ```thread::scope``` 执行此练习查看问题
```rust
use std::thread;
fn main() {
  let a = [0, 1, 2];
  thread::scope(|scope| {
      scope.spawn(|| {
          for x in &a {
            println!("{x}");
          }
      });
  });
}
```
----
# Rust 并发
- 我们还可以使用 ```move``` 将所有权转移到线程。对于像 `[i32; 3]` 这样的 `Copy` 类型，`move` 关键字将数据复制到闭包中，原始数据仍然可用
```rust
use std::thread;
fn main() {
  let mut a = [0, 1, 2];
  let handle = thread::spawn(move || {
      for x in a {
        println!("{x}");
      }
  });
  a[0] = 42;    // 不影响发送到线程的副本
  handle.join().unwrap();
}
```

# Rust 并发
- ```Arc<T>``` 可用于在多个线程之间共享*只读*引用
    - ```Arc``` 代表原子引用计数。在引用计数达到 0 之前引用不会释放
    - ```Arc::clone()``` 仅增加引用计数而不克隆数据
```rust
use std::sync::Arc;
use std::thread;
fn main() {
    let a = Arc::new([0, 1, 2]);
    let mut handles = Vec::new();
    for i in 0..2 {
        let arc = Arc::clone(&a);
        handles.push(thread::spawn(move || {
            println!("线程：{i} {arc:?}");
        }));
    }
    handles.into_iter().for_each(|h| h.join().unwrap());
}
```

# Rust 并发
- ```Arc<T>``` 可以与 ```Mutex<T>``` 组合提供可变引用。
    - ```Mutex``` 保护数据并确保只有持有锁的线程有访问权限。
    - `MutexGuard` 在离开作用域时自动释放（RAII）。注意：`std::mem::forget` 仍然可以泄漏 guard —— 所以"不可能忘记解锁"比"不可能泄漏"更准确。
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let counter = Arc::new(Mutex::new(0));
    let mut handles = Vec::new();

    for _ in 0..5 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            let mut num = counter.lock().unwrap();
            *num += 1;
            // MutexGuard 在这里被 drop —— 锁自动释放
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("最终计数：{}", *counter.lock().unwrap());
    // 输出：最终计数：5
}
```

# Rust 并发：RwLock
- `RwLock<T>` 允许**多个并发读者**或**一个独占写者** —— 来自 C++ 的读/写锁模式（`std::shared_mutex`）
    - 当读远多于写时使用 `RwLock`（例如，配置、缓存）
    - 当读/写频率相似或临界区短时使用 `Mutex`
```rust
use std::sync::{Arc, RwLock};
use std::thread;

fn main() {
    let config = Arc::new(RwLock::new(String::from("v1.0")));
    let mut handles = Vec::new();

    // 生成 5 个读者 —— 都可以并发运行
    for i in 0..5 {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let val = config.read().unwrap();  // 多个读者 OK
            println!("读者 {i}: {val}");
        }));
    }

    // 一个写者 —— 阻塞直到所有读者完成
    {
        let config = Arc::clone(&config);
        handles.push(thread::spawn(move || {
            let mut val = config.write().unwrap();  // 独占访问
            *val = String::from("v2.0");
            println!("写者：更新为 {val}");
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }
}
```

# Rust 并发：Mutex 中毒
- 如果线程在持有 `Mutex` 或 `RwLock` 时**panic**，锁变为**中毒**
    - 后续 `.lock()` 调用返回 `Err(PoisonError)` —— 数据可能处于不一致状态
    - 如果你确信数据仍然有效，可以使用 `.into_inner()` 恢复
    - 这在 C++ 中没有等价物 —— `std::mutex` 没有中毒概念；panic 的线程只是让锁保持锁定
```rust
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let data = Arc::new(Mutex::new(vec![1, 2, 3]));

    let data2 = Arc::clone(&data);
    let handle = thread::spawn(move || {
        let mut guard = data2.lock().unwrap();
        guard.push(4);
        panic!("哎呀!");  // 锁现在中毒了
    });

    let _ = handle.join();  // 线程 panic

    // 后续锁定尝试返回 Err(PoisonError)
    match data.lock() {
        Ok(guard) => println!("数据：{guard:?}"),
        Err(poisoned) => {
            println!("锁中毒了！恢复中...");
            let guard = poisoned.into_inner();  // 仍然访问数据
            println!("恢复数据：{guard:?}");  // [1, 2, 3, 4] —— push 在 panic 前成功
        }
    }
}
```

# Rust 并发：Atomics
- 对于简单的计数器和标志，`std::sync::atomic` 类型避免 `Mutex` 的开销
    - `AtomicBool`、`AtomicI32`、`AtomicU64`、`AtomicUsize` 等
    - 等价于 C++ `std::atomic<T>` —— 相同的内存顺序模型（`Relaxed`、`Acquire`、`Release`、`SeqCst`）
```rust
use std::sync::atomic::{AtomicU64, Ordering};
use std::sync::Arc;
use std::thread;

fn main() {
    let counter = Arc::new(AtomicU64::new(0));
    let mut handles = Vec::new();

    for _ in 0..10 {
        let counter = Arc::clone(&counter);
        handles.push(thread::spawn(move || {
            for _ in 0..1000 {
                counter.fetch_add(1, Ordering::Relaxed);
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    println!("计数器：{}", counter.load(Ordering::SeqCst));
    // 输出：计数器：10000
}
```

| 原语 | 何时使用 | C++ 等价物 |
|-----------|-------------|----------------|
| `Mutex<T>` | 通用可变共享状态 | `std::mutex` + 手动数据关联 |
| `RwLock<T>` | 读多写少的工作负载 | `std::shared_mutex` |
| `Atomic*` | 简单计数器、标志、无锁模式 | `std::atomic<T>` |
| `Condvar` | 等待条件变为真 | `std::condition_variable` |

# Rust 并发：Condvar
- `Condvar`（条件变量）让线程**睡眠直到另一个线程信号**条件已改变
    - 总是与 `Mutex` 配对 —— 模式是：锁定、检查条件、如果未准备好则等待、准备好后行动
    - 等价于 C++ `std::condition_variable` / `std::condition_variable::wait`
    - 处理**虚假唤醒** —— 总是在循环中重新检查条件（或使用 `wait_while`/`wait_until`）
```rust
use std::sync::{Arc, Condvar, Mutex};
use std::thread;

fn main() {
    let pair = Arc::new((Mutex::new(false), Condvar::new()));

    // 生成等待信号的工作线程
    let pair2 = Arc::clone(&pair);
    let worker = thread::spawn(move || {
        let (lock, cvar) = &*pair2;
        let mut ready = lock.lock().unwrap();
        // wait：睡眠直到被信号（总是在循环中重新检查虚假唤醒）
        while !*ready {
            ready = cvar.wait(ready).unwrap();
        }
        println!("工作线程：条件满足，继续！");
    });

    // 主线程做一些工作，然后信号通知工作线程
    thread::sleep(std::time::Duration::from_millis(100));
    {
        let (lock, cvar) = &*pair;
        let mut ready = lock.lock().unwrap();
        *ready = true;
        cvar.notify_one();  // 唤醒一个等待的线程（notify_all() 唤醒所有）
    }

    worker.join().unwrap();
}
```

> **何时使用 Condvar vs channel：** 当线程共享可变状态并需要等待该状态的条件时使用 `Condvar`（例如，"缓冲区不为空"）。当线程需要传递*消息*时使用 channel（`mpsc`）。channel 通常更容易推理。

# Rust 并发
- Rust channel 可用于在 ```Sender``` 和 ```Receiver``` 之间交换消息
    - 这使用称为 ```mpsc``` 或 ```多生产者，单消费者``` 的范式
    - ```send()``` 和 ```recv()``` 都可以阻塞线程
```rust
use std::sync::mpsc;

fn main() {
    let (tx, rx) = mpsc::channel();
    
    tx.send(10).unwrap();
    tx.send(20).unwrap();
    
    println!("收到：{:?}", rx.recv());
    println!("收到：{:?}", rx.recv());

    let tx2 = tx.clone();
    tx2.send(30).unwrap();
    println!("收到：{:?}", rx.recv());
}
```

# Rust 并发
- channel 可以与线程组合
```rust
use std::sync::mpsc;
use std::thread;
use std::time::Duration;

fn main() {
    let (tx, rx) = mpsc::channel();
    for _ in 0..2 {
        let tx2 = tx.clone();
        thread::spawn(move || {
            let thread_id = thread::current().id();
            for i in 0..10 {
                tx2.send(format!("消息 {i}")).unwrap();
                println!("{thread_id:?}: 发送消息 {i}");
            }
            println!("{thread_id:?}: 完成");
        });
    }

    // 丢弃原始发送者，以便 rx.iter() 在所有克隆的发送者丢弃后终止
    drop(tx);

    thread::sleep(Duration::from_millis(100));

    for msg in rx.iter() {
        println!("主线程：收到 {msg}");
    }
}
```



## 为什么 Rust 防止数据竞争：Send 和 Sync

- Rust 使用两个标记 trait 在编译时强制执行线程安全：
    - `Send`：如果类型可以安全地**转移**到另一个线程，则它是 `Send`
    - `Sync`：如果类型可以安全地在线程之间（通过 `&T`）**共享**，则它是 `Sync`
- 大多数类型自动是 `Send + Sync`。值得注意的例外：
    - `Rc<T>` **既不**是 Send 也不是 Sync（对线程使用 `Arc<T>`）
    - `Cell<T>` 和 `RefCell<T>` **不**是 Sync（使用 `Mutex<T>` 或 `RwLock<T>`）
    - 原始指针（`*const T`、`*mut T`）**既不**是 Send 也不是 Sync
- 这就是为什么编译器阻止你在线程间使用 `Rc<T>` —— 它根本不实现 `Send`
- `Arc<Mutex<T>>` 是 `Rc<RefCell<T>>` 的线程安全等价物

> **直觉** *(Jon Gjengset)*：把值看作玩具。
> **`Send`** = 你可以把你的玩具**给**另一个孩子（线程） —— 转移所有权是安全的。
> **`Sync`** = 你可以**让其他人同时玩你的玩具** —— 共享引用是安全的。
> `Rc<T>` 有脆弱（非原子）引用计数器；传递或共享它会损坏计数，所以它既不是 `Send` 也不是 `Sync`。


# 练习：多线程词数统计

🔴 **挑战** —— 组合线程、Arc、Mutex 和 HashMap

- 给定文本行的 `Vec<String>`，为每行生成一个线程统计该行中的单词数
- 使用 `Arc<Mutex<HashMap<String, usize>>>` 收集结果
- 打印所有行的总词数
- **bonus**：尝试使用 channel（`mpsc`）而不是共享状态实现

<details><summary>答案（点击展开）</summary>

```rust
use std::collections::HashMap;
use std::sync::{Arc, Mutex};
use std::thread;

fn main() {
    let lines = vec![
        "the quick brown fox".to_string(),
        "jumps over the lazy dog".to_string(),
        "the fox is quick".to_string(),
    ];

    let word_counts: Arc<Mutex<HashMap<String, usize>>> =
        Arc::new(Mutex::new(HashMap::new()));

    let mut handles = vec![];
    for line in &lines {
        let line = line.clone();
        let counts = Arc::clone(&word_counts);
        handles.push(thread::spawn(move || {
            for word in line.split_whitespace() {
                let mut map = counts.lock().unwrap();
                *map.entry(word.to_lowercase()).or_insert(0) += 1;
            }
        }));
    }

    for handle in handles {
        handle.join().unwrap();
    }

    let counts = word_counts.lock().unwrap();
    let total: usize = counts.values().sum();
    println!("词频：{counts:#?}");
    println!("总词数：{total}");
}
// 输出（顺序可能不同）：
// 词频：{
//     "the": 3,
//     "quick": 2,
//     "brown": 1,
//     "fox": 2,
//     "jumps": 1,
//     "over": 1,
//     "lazy": 1,
//     "dog": 1,
//     "is": 1,
// }
// 总词数：13
```

</details>



