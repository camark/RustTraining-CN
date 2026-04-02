## 日志和追踪：syslog/printf → `log` + `tracing`

> **你将学到什么：** Rust 的双层日志架构（facade + backend）、`log` 和 `tracing` crate、带 span 的结构化日志，以及这如何替换 `printf`/`syslog` 调试。

C++ 诊断代码通常使用 `printf`、`syslog` 或自定义日志框架。Rust 有标准化的双层日志架构：**facade** crate（`log` 或 `tracing`）和 **backend**（实际的日志器实现）。

### `log` facade —— Rust 的通用日志 API

`log` crate 提供镜像 syslog 严重级别的宏。库使用 `log` 宏；二进制文件选择 backend：

```rust
// Cargo.toml
// [dependencies]
// log = "0.4"
// env_logger = "0.11"    # 众多 backend 之一

use log::{info, warn, error, debug, trace};

fn check_sensor(id: u32, temp: f64) {
    trace!("Reading sensor {id}");           // 最细粒度
    debug!("Sensor {id} raw value: {temp}"); // 开发时细节

    if temp > 85.0 {
        warn!("Sensor {id} high temperature: {temp}°C");
    }
    if temp > 95.0 {
        error!("Sensor {id} CRITICAL: {temp}°C — initiating shutdown");
    }
    info!("Sensor {id} check complete");     // 正常运行
}

fn main() {
    // 初始化 backend —— 通常在 main() 中完成一次
    env_logger::init();  // 由 RUST_LOG 环境变量控制

    check_sensor(0, 72.5);
    check_sensor(1, 91.0);
}
```

```bash
# 通过环境变量控制日志级别
RUST_LOG=debug cargo run          # 显示 debug 及以上
RUST_LOG=warn cargo run           # 仅显示 warn 和 error
RUST_LOG=my_crate=trace cargo run # 每模块过滤
RUST_LOG=my_crate::gpu=debug,warn cargo run  # 混合级别
```

### C++ 对比

| C++ | Rust (`log`) | 注释 |
|-----|-------------|-------|
| `printf("DEBUG: %s\n", msg)` | `debug!("{msg}")` | 编译时检查格式 |
| `syslog(LOG_ERR, "...")` | `error!("...")` | Backend 决定输出位置 |
| `#ifdef DEBUG` 围绕日志调用 | `trace!` / `debug!` 在 max_level 时被裁剪 | 禁用时零成本 |
| 自定义 `Logger::log(level, msg)` | `log::info!("...")` —— 所有 crate 使用相同 API | 通用 facade，可替换 backend |
| 每文件日志详细度 | `RUST_LOG=crate::module=level` | 基于环境，无需重新编译 |

### `tracing` crate —— 带 span 的结构化日志

`tracing` 扩展 `log` 带**结构化字段**和**span**（定时作用域）。
这对于想要跟踪上下文的诊断代码特别有用：

```rust
// Cargo.toml
// [dependencies]
// tracing = "0.1"
// tracing-subscriber = { version = "0.3", features = ["env-filter"] }

use tracing::{info, warn, error, instrument, info_span};

#[instrument(skip(data), fields(gpu_id = gpu_id, data_len = data.len()))]
fn run_gpu_test(gpu_id: u32, data: &[u8]) -> Result<(), String> {
    info!("Starting GPU test");

    let span = info_span!("ecc_check", gpu_id);
    let _guard = span.enter();  // 此作用域内所有日志包含 gpu_id

    if data.is_empty() {
        error!(gpu_id, "No test data provided");
        return Err("empty data".to_string());
    }

    // 结构化字段 —— 机器可解析，不仅是字符串插值
    info!(
        gpu_id,
        temp_celsius = 72.5,
        ecc_errors = 0,
        "ECC check passed"
    );

    Ok(())
}

fn main() {
    // 初始化 tracing subscriber
    tracing_subscriber::fmt()
        .with_env_filter("debug")  // 或使用 RUST_LOG 环境变量
        .with_target(true)          // 显示模块路径
        .with_thread_ids(true)      // 显示线程 ID
        .init();

    let _ = run_gpu_test(0, &[1, 2, 3]);
}
```

带 `tracing-subscriber` 的输出：
```rust
2026-02-15T10:30:00.123Z DEBUG ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}: my_crate: Starting GPU test
2026-02-15T10:30:00.124Z  INFO ThreadId(01) run_gpu_test{gpu_id=0 data_len=3}:ecc_check{gpu_id=0}: my_crate: ECC check passed gpu_id=0 temp_celsius=72.5 ecc_errors=0
```

### `#[instrument]` —— 自动创建 span

`#[instrument]` 属性自动用函数名及其参数创建 span：

```rust
use tracing::instrument;

#[instrument]
fn parse_sel_record(record_id: u16, sensor_type: u8, data: &[u8]) -> Result<(), String> {
    // 此函数内所有日志自动包含：
    // record_id、sensor_type 和 data（如果实现 Debug）
    tracing::debug!("Parsing SEL record");
    Ok(())
}

// skip：排除大型/敏感参数出 span
// fields：添加计算字段
#[instrument(skip(raw_buffer), fields(buf_len = raw_buffer.len()))]
fn decode_ipmi_response(raw_buffer: &[u8]) -> Result<Vec<u8>, String> {
    tracing::trace!("Decoding {} bytes", raw_buffer.len());
    Ok(raw_buffer.to_vec())
}
```

### `log` vs `tracing` —— 使用哪个

| 方面 | `log` | `tracing` |
|--------|-------|-----------|
| **复杂性** | 简单 —— 5 个宏 | 更丰富 —— span、字段、instrument |
| **结构化数据** | 仅字符串插值 | 键值字段：`info!(gpu_id = 0, "msg")` |
| **计时 / span** | 无 | 有 —— `#[instrument]`、`span.enter()` |
| **异步支持** | 基础 | 一流 —— span 跨 `.await` 传播 |
| **兼容性** | 通用 facade | 兼容 `log`（有 `log` 桥接） |
| **何时使用** | 简单应用、库 | 诊断工具、异步代码、可观测性 |

> **推荐**：对生产诊断风格项目（带结构化输出的诊断工具）使用 `tracing`。对希望最小依赖的简单库使用 `log`。`tracing` 包含兼容性层，因此使用 `log` 宏的库仍然可以与 `tracing` subscriber 一起工作。

### Backend 选项

| Backend Crate | 输出 | 用例 |
|--------------|--------|----------|
| `env_logger` | stderr，彩色 | 开发、简单 CLI 工具 |
| `tracing-subscriber` | stderr，格式化 | 生产环境带 `tracing` |
| `syslog` | 系统 syslog | Linux 系统服务 |
| `tracing-journald` | systemd journal | systemd 管理服务 |
| `tracing-appender` | 旋转日志文件 | 长运行守护进程 |
| `tracing-opentelemetry` | OpenTelemetry 收集器 | 分布式追踪 |

----
