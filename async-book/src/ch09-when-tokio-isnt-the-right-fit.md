# 9. When Tokio Isn't the Right Fit 🟡

> **你将学到什么：**
> - `'static` 问题：当 `tokio::spawn` 迫使你到处使用 `Arc`
> - 用于 `!Send` futures 的 `LocalSet`
> - `FuturesUnordered` 用于对借用友好的并发（不需要 spawn）
> - `JoinSet` 用于管理的任务组
> - 编写运行时无关的库

```mermaid
graph TD
    START["Need concurrent futures?"] --> STATIC{"Can futures be 'static?"}
    STATIC -->|Yes| SEND{"Are futures Send?"}
    STATIC -->|No| FU["FuturesUnordered<br/>Runs on current task"]
    SEND -->|Yes| SPAWN["tokio::spawn<br/>Multi-threaded"]
    SEND -->|No| LOCAL["LocalSet<br/>Single-threaded"]
    SPAWN --> MANAGE{"Need to track/abort tasks?"}
    MANAGE -->|Yes| JOINSET["JoinSet / TaskTracker"]
    MANAGE -->|No| HANDLE["JoinHandle"]

    style START fill:#f5f5f5,stroke:#333,color:#000
    style FU fill:#d4efdf,stroke:#27ae60,color:#000
    style SPAWN fill:#e8f4f8,stroke:#2980b9,color:#000
    style LOCAL fill:#fef9e7,stroke:#f39c12,color:#000
    style JOINSET fill:#e8daef,stroke:#8e44ad,color:#000
    style HANDLE fill:#e8f4f8,stroke:#2980b9,color:#000
```

## 'static Future 问题

Tokio 的 `spawn` 需要 `'static` futures。这意味着你不能在生成的任务中借用本地数据：

```rust
async fn process_items(items: &[String]) {
    // ❌ 不能这样做 —— items 被借用，不是 'static
    // for item in items {
    //     tokio::spawn(async {
    //         process(item).await;
    //     });
    // }

    // 😐 变通方案 1：克隆所有东西
    for item in items {
        let item = item.clone();
        tokio::spawn(async move {
            process(&item).await;
        });
    }

    // 😐 变通方案 2：使用 Arc
    let items = Arc::new(items.to_vec());
    for i in 0..items.len() {
        let items = Arc::clone(&items);
        tokio::spawn(async move {
            process(&items[i]).await;
        });
    }
}
```

这很烦人！在 Go 中，你可以直接 `go func() { use(item) }` 使用闭包。在 Rust 中，所有权系统迫使你思考谁拥有什么以及它存活多久。

### `tokio::spawn` 的替代方案

不是每个问题都需要 `spawn`。这里有三个工具，每个解决*不同*的约束：

```rust
// 1. FuturesUnordered —— 完全避免 'static（不需要 spawn！）
use futures::stream::{FuturesUnordered, StreamExt};

async fn process_items(items: &[String]) {
    let futures: FuturesUnordered<_> = items
        .iter()
        .map(|item| async move {
            // ✅ 可以借用 item —— 不需要 spawn，不需要 'static！
            process(item).await
        })
        .collect();

    // 驱动所有 futures 到完成
    futures.for_each(|result| async move {
        println!("Result: {result:?}");
    }).await;
}

// 2. tokio::task::LocalSet —— 在当前线程运行 !Send futures
//    ⚠️  仍然需要 'static —— 解决 Send，不是 'static
use tokio::task::LocalSet;

let local_set = LocalSet::new();
local_set.run_until(async {
    tokio::task::spawn_local(async {
        // 可以在这里使用 Rc、Cell 和其他 !Send 类型
        let rc = std::rc::Rc::new(42);
        println!("{rc}");
    }).await.unwrap();
}).await;

// 3. tokio JoinSet（tokio 1.21+）—— 管理一组生成的任务
//    ⚠️  仍然需要 'static + Send —— 解决任务*管理*，
//    不是 'static 问题。用于跟踪、取消和加入动态任务组。
use tokio::task::JoinSet;

async fn with_joinset() {
    let mut set = JoinSet::new();

    for i in 0..10 {
        // i 是 Copy 并被移入闭包 —— 已经是 'static。
        // 你仍然需要 Arc 或 clone 来借用数据。
        set.spawn(async move {
            tokio::time::sleep(Duration::from_millis(100)).await;
            i * 2
        });
    }

    while let Some(result) = set.join_next().await {
        println!("Task completed: {:?}", result.unwrap());
    }
}
```

> **哪个工具解决哪个问题？**
>
> | 你遇到的约束 | 工具 | 避免 `'static`？ | 避免 `Send`？ |
> |---|---|---|---|
> | 无法使 futures `'static` | `FuturesUnordered` | ✅ 是 | ✅ 是 |
> | Futures 是 `'static` 但 `!Send` | `LocalSet` | ❌ 否 | ✅ 是 |
> | 需要跟踪/取消生成的任务 | `JoinSet` | ❌ 否 | ❌ 否 |

### 库的轻量级运行时

如果你在编写库 —— 不要强迫用户使用 tokio：

```rust
// ❌ 坏：库强迫用户使用 tokio
pub async fn my_lib_function() {
    tokio::time::sleep(Duration::from_secs(1)).await;
    // 现在你的用户必须使用 tokio
}

// ✅ 好：库是运行时无关的
pub async fn my_lib_function() {
    // 只使用 std::future 和 futures crate 中的类型
    do_computation().await;
}

// ✅ 好：接受一个通用的 future 进行 I/O 操作
pub async fn fetch_with_retry<F, Fut, T, E>(
    operation: F,
    max_retries: usize,
) -> Result<T, E>
where
    F: Fn() -> Fut,
    Fut: Future<Output = Result<T, E>>,
{
    for attempt in 0..max_retries {
        match operation().await {
            Ok(val) => return Ok(val),
            Err(e) if attempt == max_retries - 1 => return Err(e),
            Err(_) => continue,
        }
    }
    unreachable!()
}
```

> **经验法则**：库应该依赖 `futures` crate，而不是 `tokio`。
> 应用程序应该依赖 `tokio`（或他们选择的运行时）。
> 这使生态系统可组合。

<details>
<summary><strong>🏋️ 练习：FuturesUnordered vs Spawn</strong>（点击展开）</summary>

**挑战**：用两种方式编写相同的函数 —— 一种使用 `tokio::spawn`（需要 `'static`），另一种使用 `FuturesUnordered`（借用数据）。函数接收 `&[String]` 并在模拟异步查找后返回每个字符串的长度。

比较：哪种方法需要 `.clone()`？哪种可以借用输入切片？

<details>
<summary>🔑 答案</summary>

```rust
use futures::stream::{FuturesUnordered, StreamExt};
use tokio::time::{sleep, Duration};

// 版本 1：tokio::spawn —— 需要 'static，必须克隆
async fn lengths_with_spawn(items: &[String]) -> Vec<usize> {
    let mut handles = Vec::new();
    for item in items {
        let owned = item.clone(); // 必须克隆 —— spawn 需要 'static
        handles.push(tokio::spawn(async move {
            sleep(Duration::from_millis(10)).await;
            owned.len()
        }));
    }

    let mut results = Vec::new();
    for handle in handles {
        results.push(handle.await.unwrap());
    }
    results
}

// 版本 2：FuturesUnordered —— 借用数据，不需要克隆
async fn lengths_without_spawn(items: &[String]) -> Vec<usize> {
    let futures: FuturesUnordered<_> = items
        .iter()
        .map(|item| async move {
            sleep(Duration::from_millis(10)).await;
            item.len() // ✅ 借用 item —— 不需要克隆！
        })
        .collect();

    futures.collect().await
}

#[tokio::test]
async fn test_both_versions() {
    let items = vec!["hello".into(), "world".into(), "rust".into()];

    let v1 = lengths_with_spawn(&items).await;
    // 注意：v1 保持插入顺序（顺序 join）

    let mut v2 = lengths_without_spawn(&items).await;
    v2.sort(); // FuturesUnordered 按完成顺序返回

    assert_eq!(v1, vec![5, 5, 4]);
    assert_eq!(v2, vec![4, 5, 5]);
}
```

**关键要点**：`FuturesUnordered` 通过在当前任务上运行所有 futures 来避免 `'static` 要求（无线程迁移）。权衡：所有 futures 共享一个任务 —— 如果一个阻塞，其他停滞。对应该在不同线程上运行的 CPU 重型工作使用 `spawn`。

</details>
</details>

> **关键要点 —— When Tokio Isn't the Right Fit**
> - `FuturesUnordered` 在当前任务上并发运行 futures —— 没有 `'static` 要求
> - `LocalSet` 在单线程执行器上启用 `!Send` futures
> - `JoinSet`（tokio 1.21+）提供管理的任务组，自动清理
> - 对于库：只依赖 `std::future::Future` + `futures` crate，不直接依赖 tokio

> **另见：**[第 8 章 — Tokio Deep Dive](ch08-tokio-deep-dive.md) 了解何时 spawn 是正确的工具，[第 11 章 — Streams](ch11-streams-and-asynciterator.md) 了解 `buffer_unordered()` 作为另一个并发限制器

***
