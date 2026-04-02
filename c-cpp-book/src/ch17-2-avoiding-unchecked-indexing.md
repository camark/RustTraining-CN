## 避免未检查的索引

> **你将学到什么：** 为什么 `vec[i]` 在 Rust 中是危险的（越界时 panic），以及安全替代方案如 `.get()`、迭代器和 `HashMap` 的 `entry()` API。用显式处理替换 C++ 的未定义行为。

- 在 C++ 中，`vec[i]` 和 `map[key]` 有未定义行为/对缺失键自动插入。Rust 的 `[]` 在越界时 panic。
- **原则**：使用 `.get()` 而不是 `[]`，除非你能*证明*索引是有效的。

### C++ → Rust 对比
```cpp
// C++ —— 静默 UB 或插入
std::vector<int> v = {1, 2, 3};
int x = v[10];        // UB! operator[] 没有边界检查

std::map<std::string, int> m;
int y = m["missing"]; // 静默插入键，值为 0！
```

```rust
// Rust —— 安全替代方案
let v = vec![1, 2, 3];

// 坏：如果索引越界则 panic
// let x = v[10];

// 好：返回 Option<&i32>
let x = v.get(10);              // None —— 不 panic
let x = v.get(1).copied().unwrap_or(0);  // 2，或缺失时为 0
```

### 真实示例：生产 Rust 代码中的安全字节解析
```rust
// 示例：diagnostics.rs
// 解析二进制 SEL 记录 —— 缓冲区可能比预期短
let sensor_num = bytes.get(7).copied().unwrap_or(0);
let ppin = cpu_ppin.get(i).map(|s| s.as_str()).unwrap_or("");
```

### 真实示例：使用 `.and_then()` 链式安全查找
```rust
// 示例：profile.rs —— 双重查找：HashMap → Vec
pub fn get_processor(&self, location: &str) -> Option<&Processor> {
    self.processor_by_location
        .get(location)                              // HashMap → Option<&usize>
        .and_then(|&idx| self.processors.get(idx))   // Vec → Option<&Processor>
}
// 两个查找都返回 Option —— 无 panic，无 UB
```

### 真实示例：安全 JSON 导航
```rust
// 示例：framework.rs —— 每个 JSON 键返回 Option
let manufacturer = product_fru
    .get("Manufacturer")            // Option<&Value>
    .and_then(|v| v.as_str())       // Option<&str>
    .unwrap_or(UNKNOWN_VALUE)       // &str（安全回退）
    .to_string();
```
与 C++ 模式对比：`json["SystemInfo"]["ProductFru"]["Manufacturer"]` —— 任何缺失的键都会抛出 `nlohmann::json::out_of_range`。

### 何时 `[]` 可接受
- **边界检查后**：`if i < v.len() { v[i] }`
- **在测试中**：panic 是期望的行为
- **使用常量**：`let first = v[0];` 在 `assert!(!v.is_empty());` 之后

----

## 使用 unwrap_or 安全提取值

- `unwrap()` 在 `None` / `Err` 时 panic。在生产代码中，优先使用安全替代方案。

### unwrap 家族
| **方法** | **在 None/Err 时的行为** | **何时使用** |
|-----------|------------------------|-------------|
| `.unwrap()` | **Panic** | 仅测试，或可证明的不可失败操作 |
| `.expect("msg")` | 带消息 panic | 当 panic 合理时，解释为什么 |
| `.unwrap_or(default)` | 返回 `default` | 有廉价的常量回退值 |
| `.unwrap_or_else(\|\| expr)` | 调用闭包 | 回退计算开销大 |
| `.unwrap_or_default()` | 返回 `Default::default()` | 类型实现 `Default` |

### 真实示例：带安全默认值的解析
```rust
// 示例：peripherals.rs
// 正则捕获组可能不匹配 —— 提供安全回退
let bus_hex = caps.get(1).map(|m| m.as_str()).unwrap_or("00");
let fw_status = caps.get(5).map(|m| m.as_str()).unwrap_or("0x0");
let bus = u8::from_str_radix(bus_hex, 16).unwrap_or(0);
```

### 真实示例：带回退结构体的 `unwrap_or_else`
```rust
// 示例：framework.rs
// 完整函数将逻辑包装在返回 Option 的闭包中；
// 如果任何事失败，返回默认结构体：
(|| -> Option<BaseboardFru> {
    let content = std::fs::read_to_string(path).ok()?;
    let json: serde_json::Value = serde_json::from_str(&content).ok()?;
    // ... 用 .get()? 链提取字段
    Some(baseboard_fru)
})()
.unwrap_or_else(|| BaseboardFru {
    manufacturer: String::new(),
    model: String::new(),
    product_part_number: String::new(),
    serial_number: String::new(),
    asset_tag: String::new(),
})
```

### 真实示例：配置反序列化时的 `unwrap_or_default`
```rust
// 示例：framework.rs
// 如果 JSON 配置解析失败，回退到 Default —— 不崩溃
Ok(json) => serde_json::from_str(&json).unwrap_or_default(),
```
C++ 等价物是在 `nlohmann::json::parse()` 周围使用 `try/catch`，在 catch 块中手动构造默认值。

----

## 函数式转换：map、map_err、find_map

- `Option` 和 `Result` 上的这些方法允许你转换包含的值而无需 unwrap，用线性链替换嵌套 `if/else`。

### 快速参考
| **方法** | **在** | **作用** | **C++ 等价物** |
|-----------|-------|---------|-------------------|
| `.map(\|v\| ...)` | `Option` / `Result` | 转换 `Some`/`Ok` 值 | `if (opt) { *opt = transform(*opt); }` |
| `.map_err(\|e\| ...)` | `Result` | 转换 `Err` 值 | 在 catch 块中添加上下文 |
| `.and_then(\|v\| ...)` | `Option` / `Result` | 链式返回 `Option`/`Result` 的操作 | 嵌套 if 检查 |
| `.find_map(\|v\| ...)` | 迭代器 | 一次遍历完成 `find` + `map` | 带 `if + break` 的循环 |
| `.filter(\|v\| ...)` | `Option` / 迭代器 | 仅保留匹配谓词的值 | `if (!predicate) return nullopt;` |
| `.ok()?` | `Result` | 转换 `Result → Option` 并传播 `None` | `if (result.has_error()) return nullopt;` |

### 真实示例：JSON 字段提取的 `.and_then()` 链
```rust
// 示例：framework.rs —— 带回退查找序列号
let sys_info = json.get("SystemInfo")?;

// 先尝试 BaseboardFru.BoardSerialNumber
if let Some(serial) = sys_info
    .get("BaseboardFru")
    .and_then(|b| b.get("BoardSerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)     // 仅接受非空、有效的序列号
{
    return Some(serial.to_string());
}

// 回退到 BoardFru.SerialNumber
sys_info
    .get("BoardFru")
    .and_then(|b| b.get("SerialNumber"))
    .and_then(|v| v.as_str())
    .filter(valid_serial)
    .map(|s| s.to_string())   // 仅当 Some 时转换 &str → String
```
在 C++ 中这将是一个金字塔 `if (json.contains("BaseboardFru")) { if (json["BaseboardFru"].contains("BoardSerialNumber")) { ... } }`。

### 真实示例：`find_map` —— 一次遍历完成搜索 + 转换
```rust
// 示例：context.rs —— 查找匹配传感器 + 所有者的 SDR 记录
pub fn find_for_event(&self, sensor_number: u8, owner_id: u8) -> Option<&SdrRecord> {
    self.by_sensor.get(&sensor_number).and_then(|indices| {
        indices.iter().find_map(|&i| {
            let record = &self.records[i];
            if record.sensor_owner_id() == Some(owner_id) {
                Some(record)
            } else {
                None
            }
        })
    })
}
```
`find_map` 是融合版的 `find` + `map`：它在第一个匹配处停止并转换。C++ 等价物是带 `if` + `break` 的 `for` 循环。

### 真实示例：`map_err` 用于错误上下文
```rust
// 示例：main.rs —— 在传播前为错误添加上下文
let json_str = serde_json::to_string_pretty(&config)
    .map_err(|e| format!("Failed to serialize config: {}", e))?;
```
将 `serde_json::Error` 转换为描述性的 `String` 错误，包含关于*什么*失败的上下文。

----

## JSON 处理：nlohmann::json → serde

- C++ 团队通常使用 `nlohmann::json` 处理 JSON。Rust 使用 **serde** + **serde_json** —— 更强大，因为 JSON schema 被编码在*类型系统*中。

### C++ (nlohmann) vs Rust (serde) 对比

```cpp
// C++ 使用 nlohmann::json —— 运行时字段访问
#include <nlohmann/json.hpp>
using json = nlohmann::json;

struct Fan {
    std::string logical_id;
    std::vector<std::string> sensor_ids;
};

Fan parse_fan(const json& j) {
    Fan f;
    f.logical_id = j.at("LogicalID").get<std::string>();    // 缺失时抛出
    if (j.contains("SDRSensorIdHexes")) {                   // 手动默认处理
        f.sensor_ids = j["SDRSensorIdHexes"].get<std::vector<std::string>>();
    }
    return f;
}
```

```rust
// Rust 使用 serde —— 编译时 schema，自动字段映射
use serde::{Serialize, Deserialize};

#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct Fan {
    pub logical_id: String,
    #[serde(rename = "SDRSensorIdHexes", default)]  // JSON 键 → Rust 字段
    pub sensor_ids: Vec<String>,                     // 缺失 → 空 Vec
    #[serde(default)]
    pub sensor_names: Vec<String>,                   // 缺失 → 空 Vec
}

// 一行代码替换整个解析函数：
let fan: Fan = serde_json::from_str(json_str)?;
```

### 关键 serde 属性（生产 Rust 代码的真实示例）

| **属性** | **用途** | **C++ 等价物** |
|--------------|------------|--------------------|
| `#[serde(default)]` | 对缺失字段使用 `Default::default()` | `if (j.contains(key)) { ... } else { default; }` |
| `#[serde(rename = "Key")]` | 映射 JSON 键名到 Rust 字段名 | 手动 `j.at("Key")` 访问 |
| `#[serde(flatten)]` | 吸收未知键到 `HashMap` | `for (auto& [k,v] : j.items()) { ... }` |
| `#[serde(skip)]` | 不序列化/反序列化此字段 | 不存储在 JSON 中 |
| `#[serde(tag = "type")]` | 内部标记 enum（判别字段） | `if (j["type"] == "gpu") { ... }` |

### 真实示例：完整配置结构体
```rust
// 示例：diag.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct DiagConfig {
    pub sku: SkuConfig,
    #[serde(default)]
    pub level: DiagLevel,            // 缺失 → DiagLevel::default()
    #[serde(default)]
    pub modules: ModuleConfig,       // 缺失 → ModuleConfig::default()
    #[serde(default)]
    pub output_dir: String,          // 缺失 → ""
    #[serde(default, flatten)]
    pub options: HashMap<String, serde_json::Value>,  // 吸收未知键
}

// 加载仅需 3 行（C++ 需要约 20+ 行，使用 nlohmann）：
let content = std::fs::read_to_string(path)?;
let config: DiagConfig = serde_json::from_str(&content)?;
Ok(config)
```

### 使用 `#[serde(tag = "type")]` 反序列化 Enum
```rust
// 示例：components.rs
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(tag = "type")]                   // JSON: {"type": "Gpu", "product": ...}
pub enum PcieDeviceKind {
    Gpu { product: GpuProduct, manufacturer: GpuManufacturer },
    Nic { product: NicProduct, manufacturer: NicManufacturer },
    NvmeDrive { drive_type: StorageDriveType, capacity_gb: u32 },
    // ... 还有 9 个变体
}
// serde 自动在 "type" 字段上分发 —— 无需手动 if/else 链
```
C++ 等价物是：`if (j["type"] == "Gpu") { parse_gpu(j); } else if (j["type"] == "Nic") { parse_nic(j); } ...`

# 练习：使用 serde 进行 JSON 反序列化

- 定义一个 `ServerConfig` 结构体，可从以下 JSON 反序列化：
```json
{
    "hostname": "diag-node-01",
    "port": 8080,
    "debug": true,
    "modules": ["accel_diag", "nic_diag", "cpu_diag"]
}
```
- 使用 `#[derive(Deserialize)]` 和 `serde_json::from_str()` 解析
- 为 `debug` 添加 `#[serde(default)]`，如果缺失则默认为 `false`
- **进阶**：添加一个 `enum DiagLevel { Quick, Full, Extended }` 字段，带 `#[serde(default)]`，默认为 `Quick`

**起始代码**（需要 `cargo add serde --features derive` 和 `cargo add serde_json`）：
```rust
use serde::Deserialize;

// 待完成：定义带 Default 实现的 DiagLevel enum

// 待完成：用 serde 属性定义 ServerConfig 结构体

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    // 待完成：反序列化并打印配置
    // 待完成：尝试解析缺失 "debug" 字段的 JSON —— 验证默认为 false
}
```

<details><summary>答案（点击展开）</summary>

```rust
use serde::Deserialize;

#[derive(Debug, Deserialize, Default)]
enum DiagLevel {
    #[default]
    Quick,
    Full,
    Extended,
}

#[derive(Debug, Deserialize)]
struct ServerConfig {
    hostname: String,
    port: u16,
    #[serde(default)]       // 缺失时默认为 false
    debug: bool,
    modules: Vec<String>,
    #[serde(default)]       // 缺失时默认为 DiagLevel::Quick
    level: DiagLevel,
}

fn main() {
    let json_input = r#"{
        "hostname": "diag-node-01",
        "port": 8080,
        "debug": true,
        "modules": ["accel_diag", "nic_diag", "cpu_diag"]
    }"#;

    let config: ServerConfig = serde_json::from_str(json_input)
        .expect("Failed to parse JSON");
    println!("{config:#?}");

    // 测试缺失可选字段
    let minimal = r#"{
        "hostname": "node-02",
        "port": 9090,
        "modules": []
    }"#;
    let config2: ServerConfig = serde_json::from_str(minimal)
        .expect("Failed to parse minimal JSON");
    println!("debug (default): {}", config2.debug);    // false
    println!("level (default): {:?}", config2.level);  // Quick
}
// 输出：
// ServerConfig {
//     hostname: "diag-node-01",
//     port: 8080,
//     debug: true,
//     modules: ["accel_diag", "nic_diag", "cpu_diag"],
//     level: Quick,
// }
// debug (default): false
// level (default): Quick
```

</details>

----
