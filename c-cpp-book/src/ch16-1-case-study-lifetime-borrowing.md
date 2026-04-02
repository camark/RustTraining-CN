# 案例研究 3：框架通信 → 生命周期借用

> **你将学到什么：** 如何将 C++ 原始指针框架通信模式转换为 Rust 基于生命周期的借用系统，消除悬空指针风险，同时保持零成本抽象。

## C++ 模式：指向框架的原始指针
```cpp
// C++ 原始：每个诊断模块存储一个指向框架的原始指针
class DiagBase {
protected:
    DiagFramework* m_pFramework;  // 原始指针 —— 谁拥有这个？
public:
    DiagBase(DiagFramework* fw) : m_pFramework(fw) {}
    
    void LogEvent(uint32_t code, const std::string& msg) {
        m_pFramework->GetEventLog()->Record(code, msg);  // 希望它还活着！
    }
};
// 问题：m_pFramework 是原始指针，没有生命周期保证
// 如果模块仍然引用它时框架被销毁 → UB
```

## Rust 解决方案：带生命周期借用的 DiagContext
```rust
// 示例：module.rs —— 借用，不要存储

/// 在执行期间传递给诊断模块的上下文。
/// 生命周期 'a 保证框架比上下文活得长。
pub struct DiagContext<'a> {
    pub der_log: &'a mut EventLogManager,
    pub config: &'a ModuleConfig,
    pub framework_opts: &'a HashMap<String, String>,
}

/// 模块接收上下文作为参数 —— 从不存储框架指针
pub trait DiagModule {
    fn id(&self) -> &str;
    fn execute(&mut self, ctx: &mut DiagContext) -> DiagResult<()>;
    fn pre_execute(&mut self, _ctx: &mut DiagContext) -> DiagResult<()> {
        Ok(())
    }
    fn post_execute(&mut self, _ctx: &mut DiagContext) -> DiagResult<()> {
        Ok(())
    }
}
```

### 关键洞察
- C++ 模块**存储**指向框架的指针（危险：如果框架先被销毁怎么办？）
- Rust 模块**接收**上下文作为函数参数 —— 借用检查器保证在调用期间框架是存活的
- 无原始指针，无生命周期歧义，无"希望它还活着"

----

# 案例研究 4：上帝对象 → 可组合状态

## C++ 模式：单一框架类
```cpp
// C++ 原始：框架是上帝对象
class DiagFramework {
    // 健康监控陷阱处理
    std::vector<AlertTriggerInfo> m_alertTriggers;
    std::vector<WarnTriggerInfo> m_warnTriggers;
    bool m_healthMonHasBootTimeError;
    uint32_t m_healthMonActionCounter;
    
    // GPU 诊断
    std::map<uint32_t, GpuPcieInfo> m_gpuPcieMap;
    bool m_isRecoveryContext;
    bool m_healthcheckDetectedDevices;
    // ... 30+ 个 GPU 相关字段
    
    // PCIe 树
    std::shared_ptr<CPcieTreeLinux> m_pPcieTree;
    
    // 事件日志
    CEventLogMgr* m_pEventLogMgr;
    
    // ... 其他几个方法
    void HandleGpuEvents();
    void HandleNicEvents();
    void RunGpuDiag();
    // 一切依赖一切
};
```

## Rust 解决方案：可组合状态结构体
```rust
// 示例：main.rs —— 状态分解为专注的结构体

#[derive(Default)]
struct HealthMonitorState {
    alert_triggers: Vec<AlertTriggerInfo>,
    warn_triggers: Vec<WarnTriggerInfo>,
    health_monitor_action_counter: u32,
    health_monitor_has_boot_time_error: bool,
    // 仅健康监控相关字段
}

#[derive(Default)]
struct GpuDiagState {
    gpu_pcie_map: HashMap<u32, GpuPcieInfo>,
    is_recovery_context: bool,
    healthcheck_detected_devices: bool,
    // 仅 GPU 相关字段
}

/// 框架组合这些状态而不是扁平地拥有一切
struct DiagFramework {
    ctx: DiagContext,             // 执行上下文
    args: Args,                   // CLI 参数
    pcie_tree: Option<DeviceTree>,  // 不需要 shared_ptr
    event_log_mgr: EventLogManager,   // 自有，不是原始指针
    fc_manager: FcManager,        // 故障码管理
    health: HealthMonitorState,   // 健康监控状态 —— 它自己的结构体
    gpu: GpuDiagState,           // GPU 状态 —— 它自己的结构体
}
```

### 关键洞察
- **可测试性**：每个状态结构体可以独立进行单元测试
- **可读性**：`self.health.alert_triggers` vs `m_alertTriggers` —— 清晰的所有权
- **无畏重构**：更改 `GpuDiagState` 不会意外影响健康监控处理
- **无方法汤**：只需要健康监控状态的函数接受 `&mut HealthMonitorState`，而不是整个框架

----

# 案例研究 5：Trait 对象 —— 何时它们是正确的

- 不是所有东西都应该是 enum！**诊断模块插件系统**是 trait 对象的真正用例
- 为什么？因为诊断模块**开放扩展** —— 新模块可以在不修改框架的情况下添加

```rust
// 示例：framework.rs —— Vec<Box<dyn DiagModule>> 在这里是正确的
pub struct DiagFramework {
    modules: Vec<Box<dyn DiagModule>>,        // 运行时多态
    pre_diag_modules: Vec<Box<dyn DiagModule>>,
    event_log_mgr: EventLogManager,
    // ...
}

impl DiagFramework {
    /// 注册诊断模块 —— 任何实现 DiagModule 的类型
    pub fn register_module(&mut self, module: Box<dyn DiagModule>) {
        info!("注册模块：{}", module.id());
        self.modules.push(module);
    }
}
```

### 何时使用每种模式

| **用例** | **模式** | **为什么** |
|-------------|-----------|--------|
| 编译时已知的固定变体集合 | `enum` + `match` | 穷尽检查，无 vtable |
| 硬件事件类型（Degrade、Fatal、Boot、...） | `enum GpuEventKind` | 所有变体已知，性能重要 |
| PCIe 设备类型（GPU、NIC、Switch、...） | `enum PcieDeviceKind` | 固定集合，每个变体有不同数据 |
| 插件/模块系统（开放扩展） | `Box<dyn Trait>` | 新模块可以在不修改框架的情况下添加 |
| 测试 mocking | `Box<dyn Trait>` | 注入测试替身 |

### 练习：翻译前思考
给定这段 C++ 代码：
```cpp
class Shape { public: virtual double area() = 0; };
class Circle : public Shape { double r; double area() override { return 3.14*r*r; } };
class Rect : public Shape { double w, h; double area() override { return w*h; } };
std::vector<std::unique_ptr<Shape>> shapes;
```
**问题**：Rust 翻译应该使用 `enum Shape` 还是 `Vec<Box<dyn Shape>>`？

<details><summary>答案（点击展开）</summary>

**答案**：`enum Shape` —— 因为形状的集合是**封闭的**（编译时已知）。你只应该在用户可以在运行时添加新形状类型时使用 `Box<dyn Shape>`。

```rust
// 正确的 Rust 翻译：
enum Shape {
    Circle { r: f64 },
    Rect { w: f64, h: f64 },
}

impl Shape {
    fn area(&self) -> f64 {
        match self {
            Shape::Circle { r } => std::f64::consts::PI * r * r,
            Shape::Rect { w, h } => w * h,
        }
    }
}

fn main() {
    let shapes: Vec<Shape> = vec![
        Shape::Circle { r: 5.0 },
        Shape::Rect { w: 3.0, h: 4.0 },
    ];
    for shape in &shapes {
        println!("面积：{:.2}", shape.area());
    }
}
// 输出：
// 面积：78.54
// 面积：12.00
```

</details>

----

# 翻译指标和经验教训

## 我们学到的
1. **默认使用 enum 派** —— 在约 10 万行 C++ 中，只有约 25 次 `Box<dyn Trait>` 的使用是真正需要的（插件系统、测试 mocking）。其他约 900 个虚方法成为带 match 的 enum
2. **Arena 模式消除引用循环** —— `shared_ptr` 和 `enable_shared_from_this` 是所有权不清晰的症状。先考虑谁**拥有**数据
3. **传递上下文，不要存储指针** —— 生命周期限制的 `DiagContext<'a>` 比在每个模块中存储 `Framework*` 更安全、更清晰
4. **分解上帝对象** —— 如果结构体有 30+ 个字段，它可能是 3-4 个结构体伪装的
5. **编译器是你的结对编程伙伴** —— 约 400 次 `dynamic_cast` 调用意味着约 400 个潜在的运行时失败。Rust 中零 `dynamic_cast` 等价物意味着零运行时类型错误

## 最难的部分
- **生命周期标注**：当你习惯原始指针时，搞定借用需要时间 —— 但一旦编译，它就是正确的
- **与借用检查器斗争**：想在两个地方同时使用 `&mut self`。解决方案：将状态分解为独立的结构体
- **抵制直接翻译**：无处不在的 `Vec<Box<dyn Base>>` 的诱惑。问自己："这个变体集合是封闭的吗？" → 如果是，使用 enum

## 给 C++ 团队的推荐
1. 从一个小的、自包含的模块开始（不是上帝对象）
2. 先翻译数据结构，然后是行为
3. 让编译器指导你 —— 它的错误消息非常出色
4. 在 `dyn Trait` 之前优先使用 `enum`
5. 在集成之前使用 [Rust playground](https://play.rust-lang.org/) 原型化模式

----


