# Video QoS 模块技术文档

## 1. 功能概述

`video_qos.rs` 是 RustDesk 视频服务质量（Quality of Service，QoS）控制模块，负责根据网络状况动态调整视频流的帧率（FPS）和质量比率，以提供最佳的用户体验。

### 核心功能

- **动态帧率调整**：根据网络延迟自动调整视频帧率（1-120 FPS）
- **自适应比特率（ABR）**：根据网络状况和屏幕变化动态调整视频质量
- **多用户支持**：支持多个用户同时连接，根据最差网络状况调整参数
- **RTT 估算**：通过滑动窗口算法估算往返时间（Round-Trip Time）
- **质量模式支持**：支持 Best、Balanced、Low 和自定义质量模式

## 2. 核心算法说明

### 2.1 FPS 调整算法

FPS 调整基于网络延迟的反馈机制，分为以下几个阶段：

```
网络延迟 < 50ms:   快速增加 FPS（每 3 次连续良好采样后增加 5 FPS）
网络延迟 < 100ms:  适度增加 FPS（每次增加 1 FPS）
网络延迟 < 150ms:  保持最低 FPS
网络延迟 >= 150ms: 根据延迟比例降低 FPS
```

**调整规则**：
- 新用户连接时，初始 FPS 设为 15
- 当延迟 < 150ms 时，FPS 始终高于最低 FPS，质量比率递增
- 当延迟 >= 150ms 时，FPS 始终低于最低 FPS，质量比率递减
- 响应超时（>2s）时，将 FPS 降至最低（2 FPS）

### 2.2 质量比率调整算法

质量比率每 3 秒调整一次，基于以下因素：

```
网络延迟 < 50ms:   动态屏幕时比率 × 1.15
网络延迟 < 100ms:  动态屏幕时比率 × 1.10
网络延迟 < 150ms:  动态屏幕时比率 × 1.05
网络延迟 < 200ms:  比率 × 0.95
网络延迟 < 300ms:  比率 × 0.90
网络延迟 < 500ms:  比率 × 0.85
网络延迟 >= 500ms: 比率 × 0.80
```

**动态屏幕判断**：如果在 3 秒内编码次数 >= 6 次，则认为是动态屏幕。

### 2.3 RTT 估算算法

使用滑动窗口算法估算 RTT：

- 保留最近 60 个延迟样本
- 计算历史最小 RTT 和窗口内最小 RTT
- 使用加权平均（α=0.5）计算平滑 RTT
- 需要至少 10 个样本才能提供有效估算

## 3. 关键数据结构定义

### 3.1 UserDelay

存储单个用户的延迟相关信息。

```rust
struct UserDelay {
    response_delayed: bool,           // 响应是否超时
    delay_history: VecDeque<u32>,     // 延迟历史（最近 2 个样本）
    fps: Option<u32>,                 // 该用户的建议 FPS
    rtt_calculator: RttCalculator,    // RTT 计算器
    quick_increase_fps_count: usize,  // 快速增加 FPS 计数器
    increase_fps_count: usize,        // 增加 FPS 计数器
}
```

**关键方法**：
- `add_delay(delay: u32)`: 添加延迟样本，更新 RTT
- `avg_delay() -> u32`: 计算平均延迟（减去 RTT）

### 3.2 UserData

存储单个用户的会话数据。

```rust
struct UserData {
    auto_adjust_fps: Option<u32>,    // 自动调整 FPS（保留兼容性）
    custom_fps: Option<u32>,         // 用户自定义 FPS
    quality: Option<(i64, Quality)>,  // 图像质量（时间戳，质量等级）
    delay: UserDelay,                // 延迟信息
    record: bool,                    // 是否在录制
}
```

### 3.3 DisplayData

存储显示器相关信息。

```rust
struct DisplayData {
    send_counter: usize,             // 周期内编码次数
    support_changing_quality: bool,  // 是否支持动态质量调整
}
```

### 3.4 VideoQoS

主 QoS 控制器结构。

```rust
pub struct VideoQoS {
    fps: u32,                        // 当前 FPS
    ratio: f32,                      // 当前质量比率
    users: HashMap<i32, UserData>,    // 用户会话映射
    displays: HashMap<String, DisplayData>,  // 显示器映射
    bitrate_store: u32,              // 存储的比特率
    adjust_ratio_instant: Instant,   // 上次调整比率的时间
    abr_config: bool,                // ABR 配置是否启用
    new_user_instant: Instant,       // 新用户连接时间
}
```

### 3.5 RttCalculator

RTT 估算器。

```rust
struct RttCalculator {
    min_rtt: Option<u32>,            // 历史最小 RTT
    window_min_rtt: Option<u32>,     // 窗口内最小 RTT
    smoothed_rtt: Option<u32>,       // 平滑 RTT
    samples: VecDeque<u32>,          // 最近 60 个样本
}
```

## 4. 主要函数实现细节

### 4.1 基础功能函数

#### `spf() -> Duration`

**功能**：计算每帧的持续时间（Seconds Per Frame）

**返回值**：`Duration` 类型，表示每帧的时间间隔

**实现**：
```rust
pub fn spf(&self) -> Duration {
    Duration::from_secs_f32(1. / (self.fps() as f32))
}
```

#### `fps() -> u32`

**功能**：获取当前 FPS，确保在有效范围内

**返回值**：当前 FPS 值（1-120）

**实现**：
```rust
pub fn fps(&self) -> u32 {
    let fps = self.fps;
    if fps >= MIN_FPS && fps <= MAX_FPS {
        fps
    } else {
        FPS  // 默认 30 FPS
    }
}
```

#### `ratio() -> f32`

**功能**：获取当前质量比率，进行边界检查

**返回值**：当前质量比率（0.1-40.0）

**实现**：
```rust
pub fn ratio(&mut self) -> f32 {
    if self.ratio < BR_MIN_HIGH_RESOLUTION || self.ratio > BR_MAX {
        self.ratio = BR_BALANCED;  // 恢复默认值
    }
    self.ratio
}
```

#### `bitrate() -> u32`

**功能**：获取存储的比特率

**返回值**：当前比特率（kbps）

### 4.2 用户会话管理函数

#### `on_connection_open(id: i32)`

**功能**：初始化新用户会话

**参数**：
- `id`: 用户 ID

**实现逻辑**：
1. 插入新的用户数据到 `users` 映射
2. 读取配置文件中的 ABR 设置
3. 更新新用户连接时间

#### `on_connection_close(id: i32)`

**功能**：清理用户会话

**参数**：
- `id`: 用户 ID

**实现逻辑**：
1. 从 `users` 映射中移除用户
2. 如果没有用户了，重置为默认状态

#### `user_network_delay(id: i32, delay: u32)`

**功能**：处理用户网络延迟报告

**参数**：
- `id`: 用户 ID
- `delay`: 延迟值（毫秒）

**实现逻辑**：
1. 根据目标质量确定最低 FPS 和正常 FPS
2. 更新用户的延迟历史
3. 根据延迟范围调整 FPS：
   - < 50ms: 快速增加 FPS
   - < 100ms: 适度增加 FPS
   - < 150ms: 保持最低 FPS
   - >= 150ms: 按比例降低 FPS
4. 如果是首次延迟报告，调整质量比率
5. 调用 `adjust_fps()` 更新全局 FPS

#### `user_image_quality(id: i32, image_quality: i32)`

**功能**：处理用户图像质量设置

**参数**：
- `id`: 用户 ID
- `image_quality`: 图像质量值

**实现逻辑**：
1. 将图像质量值转换为 `Quality` 枚举：
   - `Balanced` -> `Quality::Balanced`
   - `Low` -> `Quality::Low`
   - `Best` -> `Quality::Best`
   - 自定义值 -> `Quality::Custom(ratio)`
2. 更新用户的质量设置
3. 直接更新全局质量比率

#### `user_custom_fps(id: i32, fps: u32)`

**功能**：设置用户自定义 FPS

**参数**：
- `id`: 用户 ID
- `fps`: 自定义 FPS 值（1-120）

**实现逻辑**：
1. 验证 FPS 范围
2. 更新用户的自定义 FPS

#### `user_record(id: i32, v: bool)`

**功能**：设置用户录制状态

**参数**：
- `id`: 用户 ID
- `v`: 是否在录制

### 4.3 调整函数

#### `adjust_fps()`

**功能**：根据所有用户的延迟信息调整全局 FPS

**实现逻辑**：
1. 获取所有用户的最高 FPS 限制
2. 计算所有用户建议 FPS 的最小值
3. 如果有用户响应超时，将 FPS 降至最低+1
4. 如果是新连接（<1秒），限制 FPS 为初始值（15）
5. 确保 FPS 在有效范围内

#### `adjust_ratio(dynamic_screen: bool)`

**功能**：根据网络延迟和屏幕变化调整质量比率

**参数**：
- `dynamic_screen`: 是否为动态屏幕

**实现逻辑**：
1. 检查 ABR 是否启用
2. 获取所有用户的最大延迟
3. 根据目标质量设置最小比率：
   - Best: 确保至少 1Mbps（高分辨率）
   - Balanced: 适中最小比率
   - Low/Custom: 使用最低比率
4. 根据延迟范围调整比率：
   - < 150ms: 动态屏幕时增加比率
   - >= 150ms: 降低比率
5. 限制质量增加速率（每次最多增加 150kbps 对应的比率）
6. 确保比率在有效范围内

#### `highest_fps() -> u32`

**功能**：获取所有用户的最高 FPS 限制

**返回值**：最高 FPS 值（1-120）

**实现逻辑**：
1. 遍历所有用户
2. 对于每个用户，选择自定义 FPS 或自动调整 FPS 中的较小值
3. 返回所有用户 FPS 的最小值（取最保守的限制）

#### `latest_quality() -> Quality`

**功能**：获取所有用户的最新质量设置

**返回值**：最新的质量设置

**实现逻辑**：
1. 遍历所有用户的质量设置
2. 返回时间戳最新的质量设置

### 4.4 显示器管理函数

#### `new_display(video_service_name: String)`

**功能**：注册新的显示器

**参数**：
- `video_service_name`: 显示器名称

#### `remove_display(video_service_name: &str)`

**功能**：移除显示器

**参数**：
- `video_service_name`: 显示器名称

#### `update_display_data(video_service_name: &str, send_counter: usize)`

**功能**：更新显示器数据并触发调整

**参数**：
- `video_service_name`: 显示器名称
- `send_counter`: 编码次数

**实现逻辑**：
1. 更新显示器的编码计数器
2. 调用 `adjust_fps()` 调整 FPS
3. 如果 ABR 启用且距离上次调整 >= 3 秒：
   - 判断是否为动态屏幕
   - 重置编码计数器
   - 调用 `adjust_ratio()` 调整质量比率
4. 如果 ABR 未启用，使用最新质量设置

#### `set_support_changing_quality(video_service_name: &str, support: bool)`

**功能**：设置显示器是否支持动态质量调整

**参数**：
- `video_service_name`: 显示器名称
- `support`: 是否支持

### 4.5 RTT 计算函数

#### `RttCalculator::update(&mut self, delay: u32)`

**功能**：更新 RTT 估算

**参数**：
- `delay`: 新的延迟样本

**实现逻辑**：
1. 更新历史最小 RTT
2. 维护最近 60 个样本的滑动窗口
3. 计算窗口内最小 RTT
4. 如果有足够样本，计算平滑 RTT（加权平均）

#### `RttCalculator::get_rtt(&self) -> Option<u32>`

**功能**：获取当前 RTT 估算

**返回值**：
- `Some(rtt)`: 有效的 RTT 估算
- `None`: 没有足够的样本进行估算

**实现逻辑**：
1. 优先返回平滑 RTT
2. 如果没有平滑 RTT 但有足够样本，返回历史最小 RTT
3. 否则返回 None

## 5. 常量定义

### FPS 相关常量

```rust
pub const FPS: u32 = 30;           // 默认 FPS
pub const MIN_FPS: u32 = 1;         // 最小 FPS
pub const MAX_FPS: u32 = 120;       // 最大 FPS
pub const INIT_FPS: u32 = 15;       // 初始 FPS
```

### 比特率比率常量

```rust
const BR_MAX: f32 = 40.0;                          // 最大比率（2000 * 2 / 100）
const BR_MIN: f32 = 0.2;                          // 最小比率
const BR_MIN_HIGH_RESOLUTION: f32 = 0.1;         // 高分辨率最小比率
const MAX_BR_MULTIPLE: f32 = 1.0;                 // 最大比率倍数
```

### 调整间隔常量

```rust
const HISTORY_DELAY_LEN: usize = 2;               // 延迟历史长度
const ADJUST_RATIO_INTERVAL: usize = 3;           // 质量比率调整间隔（秒）
const DYNAMIC_SCREEN_THRESHOLD: usize = 2;        // 动态屏幕阈值（每秒编码次数）
const DELAY_THRESHOLD_150MS: u32 = 150;           // 网络状况阈值（毫秒）
```

### RTT 计算常量

```rust
const WINDOW_SAMPLES: usize = 60;                 // RTT 样本窗口大小
const MIN_SAMPLES: usize = 10;                   // 最小样本数
const ALPHA: f32 = 0.5;                          // 平滑因子
```

## 6. 与其他模块的交互

### 6.1 与 video_service.rs 的交互

- `video_service.rs` 调用 `VideoQoS` 的方法来获取当前 FPS 和质量比率
- `video_service.rs` 在编码完成后调用 `update_display_data()` 更新统计数据
- `video_service.rs` 在用户连接/断开时调用相应的会话管理方法

### 6.2 与 codec 模块的交互

- 使用 `scrap::codec` 模块中的 `Quality` 枚举和常量
- 质量比率传递给编码器用于控制视频质量

### 6.3 与配置系统的交互

- 从配置文件读取 `enable-abr` 选项
- 根据配置决定是否启用自适应比特率

### 6.4 与网络模块的交互

- 接收网络延迟报告
- 基于延迟调整视频参数

## 7. 使用示例

### 7.1 初始化 QoS 控制器

```rust
let mut qos = VideoQoS::default();
qos.new_display("display1".to_string());
```

### 7.2 处理用户连接

```rust
// 用户连接
qos.on_connection_open(user_id);

// 设置用户质量
qos.user_image_quality(user_id, ImageQuality::Balanced.value());

// 设置自定义 FPS
qos.user_custom_fps(user_id, 60);
```

### 7.3 处理网络延迟

```rust
// 接收到网络延迟报告
qos.user_network_delay(user_id, 120);

// 获取当前 FPS
let fps = qos.fps();

// 获取当前质量比率
let ratio = qos.ratio();
```

### 7.4 更新显示器数据

```rust
// 编码完成后更新
qos.update_display_data("display1", 1);

// 获取每帧时间
let spf = qos.spf();
```

### 7.5 清理用户会话

```rust
// 用户断开连接
qos.on_connection_close(user_id);

// 移除显示器
qos.remove_display("display1");
```

## 8. 性能优化要点

### 8.1 避免频繁调整

- 质量比率每 3 秒调整一次，避免频繁变化
- FPS 在每次延迟报告时调整，但通过计数器机制限制增加速度

### 8.2 滑动窗口算法

- RTT 估算使用滑动窗口（60 个样本）
- 延迟历史只保留最近 2 个样本，减少内存占用

### 8.3 边界检查

- 所有 FPS 和比率调整都进行边界检查，确保在有效范围内
- 使用 `clamp()` 方法简化边界检查逻辑

### 8.4 最保守策略

- 多用户场景下，使用所有用户的最差网络状况
- FPS 取所有用户建议值的最小值
- 延迟取所有用户的最大值

## 9. 注意事项

1. **新用户连接**：新用户连接后 1 秒内，FPS 限制为初始值（15），确保稳定性
2. **响应超时**：如果用户响应超时（>2s），FPS 会降至最低（2）
3. **ABR 配置**：只有当所有显示器都支持动态质量调整时，ABR 才会生效
4. **高分辨率**：对于高分辨率场景，使用更低的比率下限（0.1）
5. **质量增加限制**：质量比率每次最多增加 150kbps 对应的比率，避免过快增加

## 10. 未来改进方向

1. **机器学习预测**：使用机器学习算法预测网络状况变化
2. **更精细的延迟分段**：增加更多延迟阈值，实现更精细的调整
3. **用户偏好学习**：学习用户的使用习惯，个性化调整策略
4. **多显示器优化**：针对多显示器场景进行专门优化
5. **带宽估算**：增加带宽估算功能，更准确地控制比特率
