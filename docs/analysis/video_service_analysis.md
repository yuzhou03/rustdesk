# video_service.rs 技术分析文档

## 1. 功能概述

`video_service.rs` 是 RustDesk 远程桌面项目中负责视频捕获、编码和传输的核心服务模块。该模块实现了跨平台的屏幕/摄像头视频流处理，支持多种视频编码格式，并提供视频质量自适应、隐私模式、屏幕录制等功能。

### 1.1 主要功能

- **视频捕获**：支持显示器和摄像头两种视频源
- **视频编码**：支持多种编码格式（H.264、H.265、VP8、VP9、AV1）
- **硬件加速**：支持 VRAM 和 HWRAM 硬件编码器
- **质量控制**：动态调整视频质量和帧率（QoS）
- **隐私模式**：支持隐私模式下的屏幕保护
- **屏幕录制**：支持会话录制功能
- **截图功能**：支持远程截图
- **多显示器支持**：支持多显示器环境
- **跨平台支持**：Windows、Linux（X11/Wayland）、Android

## 2. 核心架构

### 2.1 模块依赖关系

```
video_service.rs
├── display_service (显示器服务)
├── video_qos (视频质量控制)
├── privacy_mode (隐私模式)
├── platform (平台相关功能)
│   ├── linux (Linux 平台)
│   └── windows (Windows 平台)
├── scrap (视频捕获库)
├── camera (摄像头管理)
└── hbb_common (通用工具库)
```

### 2.2 全局状态管理

```rust
lazy_static! {
    // 帧获取通知器：用于通知客户端帧已准备好
    static ref FRAME_FETCHED_NOTIFIERS: Mutex<HashMap<usize, (FrameFetchedNotifierSender, FrameFetchedNotifierReceiver)>>

    // 显示器-连接ID映射：记录哪些连接需要接收特定显示器的帧
    static ref DISPLAY_CONN_IDS: Arc<Mutex<HashMap<usize, HashSet<i32>>>>

    // 视频质量控制
    pub static ref VIDEO_QOS: Arc<Mutex<VideoQoS>>

    // UAC 进程状态（Windows）
    pub static ref IS_UAC_RUNNING: Arc<Mutex<bool>>

    // 前台窗口提升状态（Windows）
    pub static ref IS_FOREGROUND_WINDOW_ELEVATED: Arc<Mutex<bool>>

    // 截图请求队列
    static ref SCREENSHOTS: Mutex<HashMap<usize, Screenshot>>
}
```

## 3. 关键数据结构

### 3.1 VideoSource 枚举

```rust
#[derive(Clone, Copy, Debug, PartialEq, Eq)]
pub enum VideoSource {
    Monitor,  // 显示器视频源
    Camera,   // 摄像头视频源
}
```

### 3.2 VideoService 结构体

```rust
#[derive(Clone)]
pub struct VideoService {
    sp: GenericService,      // 通用服务模板
    idx: usize,             // 显示器/摄像头索引
    source: VideoSource,    // 视频源类型
}
```

### 3.3 CapturerInfo 结构体

```rust
pub(super) struct CapturerInfo {
    pub origin: (i32, i32),              // 原点坐标
    pub width: usize,                    // 宽度
    pub height: usize,                   // 高度
    pub ndisplay: usize,                 // 显示器/摄像头总数
    pub current: usize,                  // 当前索引
    pub privacy_mode_id: i32,            // 隐私模式连接ID
    pub _capturer_privacy_mode_id: i32,  // 捕获器隐私模式ID
    pub capturer: Box<dyn TraitCapturer>, // 捕获器实例
}
```

### 3.4 VideoFrameController 结构体

```rust
struct VideoFrameController {
    display_idx: usize,                  // 显示器索引
    cur: Instant,                        // 当前时间
    send_conn_ids: HashSet<i32>,         // 需要发送的连接ID集合
}
```

### 3.5 Raii 结构体（资源管理）

```rust
struct Raii {
    display_idx: usize,  // 显示器索引
    name: String,       // 服务名称
    try_vram: bool,     // 是否尝试使用 VRAM
}
```

## 4. 核心算法与流程

### 4.1 主服务运行流程（run 函数）

```
1. 初始化阶段
   ├─ Wayland 环境初始化（Linux）
   ├─ 创建捕获器（Capturer）
   ├─ 设置视频编码器（Encoder）
   ├─ 初始化帧控制器（VideoFrameController）
   └─ 启动 UAC 提升检查（Windows）

2. 主循环
   ├─ 检查 UAC 切换状态（Windows）
   ├─ 检查 QoS 参数变化
   ├─ 检查显示器变化
   ├─ 检查隐私模式变化
   ├─ 捕获视频帧
   │   ├─ 处理截图请求
   │   ├─ 转换帧格式
   │   └─ 编码并发送帧
   ├─ 等待客户端确认（最多3秒）
   └─ 帧率控制
```

### 4.2 视频捕获流程

#### 4.2.1 创建捕获器

```rust
fn create_capturer(
    privacy_mode_id: i32,
    display: Display,
    _current: usize,
    _portable_service_running: bool,
) -> ResultType<Box<dyn TraitCapturer>>
```

**流程**：
1. 检查隐私模式，如果启用则创建隐私模式捕获器（Windows）
2. 根据平台创建捕获器：
   - Windows：使用 DXGI 或 GDI
   - Linux：使用 X11 或 Wayland
   - 其他平台：使用 scrap 库

#### 4.2.2 获取捕获器信息

```rust
fn get_capturer_monitor(
    current: usize,
    portable_service_running: bool,
) -> ResultType<CapturerInfo>

fn get_capturer_camera(current: usize) -> ResultType<CapturerInfo>
```

**关键步骤**：
1. 获取所有显示器/摄像头列表
2. 验证索引有效性
3. 获取显示器/摄像头属性（分辨率、位置等）
4. 检查隐私模式状态
5. 创建捕获器实例

### 4.3 视频编码流程

#### 4.3.1 编码器配置选择

```rust
fn get_encoder_config(
    c: &CapturerInfo,
    _name: String,
    quality: f32,
    record: bool,
    _portable_service: bool,
    _source: VideoSource,
) -> EncoderCfg
```

**编码器选择优先级**：
1. **H.264/H.265**：
   - VRAM 编码器（如果可用）
   - HWRAM 编码器（如果可用）
   - 回退到 VP9
2. **VP8/VP9**：使用 VPX 编码器
3. **AV1**：使用 AOM 编码器
4. **默认**：VP9 编码器

#### 4.3.2 编码器设置

```rust
fn setup_encoder(
    c: &CapturerInfo,
    name: String,
    quality: f32,
    client_record: bool,
    record_incoming: bool,
    last_portable_service_running: bool,
    source: VideoSource,
    display_idx: usize,
) -> ResultType<(...)>
```

**关键参数**：
- `quality`：视频质量（0.0-1.0）
- `keyframe_interval`：关键帧间隔（录制时为240帧）
- `use_i444`：是否使用 I444 色彩格式

### 4.4 帧处理流程

#### 4.4.1 单帧处理

```rust
fn handle_one_frame(
    display: usize,
    sp: &GenericService,
    frame: EncodeInput,
    ms: i64,
    encoder: &mut Encoder,
    recorder: Arc<Mutex<Option<Recorder>>>,
    encode_fail_counter: &mut usize,
    first_frame: &mut bool,
    width: usize,
    height: usize,
) -> ResultType<HashSet<i32>>
```

**处理步骤**：
1. 检查新订阅者（需要切换）
2. 编码帧为视频消息
3. 写入录制器（如果启用）
4. 发送视频帧到所有订阅者
5. 返回成功发送的连接ID集合

#### 4.4.2 帧同步机制

```rust
impl VideoFrameController {
    async fn try_wait_next(&mut self, fetched_conn_ids: &mut HashSet<i32>, timeout_millis: u64)
}
```

**同步策略**：
- 等待客户端确认帧已接收
- 超时时间：3秒
- 每次等待最多300ms
- 所有客户端确认后才继续下一帧

### 4.5 视频质量控制（QoS）

#### 4.5.1 QoS 检查

```rust
fn check_qos(
    encoder: &mut Encoder,
    ratio: &mut f32,
    spf: &mut Duration,
    client_record: bool,
    send_counter: &mut usize,
    second_instant: &mut Instant,
    name: &str,
) -> ResultType<()>
```

**控制参数**：
- `ratio`：质量比例
- `spf`：每帧时间（Seconds Per Frame）
- `send_counter`：发送帧计数器
- `client_record`：客户端录制状态

**自适应策略**：
1. 质量变化时：
   - 如果编码器支持动态调整：直接调整质量
   - 如果不支持：触发服务切换
2. 录制状态变化：触发服务切换
3. 每秒更新显示数据

### 4.6 隐私模式处理

#### 4.6.1 隐私模式检查

```rust
fn check_privacy_mode_changed(
    sp: &GenericService,
    display_idx: usize,
    ci: &CapturerInfo,
) -> ResultType<()>
```

**处理逻辑**：
1. 检查隐私模式连接ID是否变化
2. 如果其他连接启用隐私模式：
   - 发送隐私模式通知
   - 广播显示器变化
   - 触发服务切换

#### 4.6.2 UAC 提升（Windows）

```rust
fn check_uac_switch(privacy_mode_id: i32, capturer_privacy_mode_id: i32) -> ResultType<()>
```

**检查内容**：
- consent.exe 进程状态
- 隐私模式实现类型
- 安装状态

### 4.7 截图处理

#### 4.7.1 截图请求

```rust
pub fn set_take_screenshot(display_idx: usize, sid: String, tx: Sender)
```

**流程**：
1. 将截图请求加入队列
2. 在帧捕获时处理
3. 异步编码为 PNG
4. 发送响应

#### 4.7.2 RGBA 转换

```rust
fn get_rgba_from_pixelbuf<'a>(pixbuf: &scrap::PixelBuffer<'a>) -> ResultType<Vec<u8>>
```

**关键处理**：
- 处理 stride 不等于 width * 4 的情况
- BGRA 到 RGBA 转换
- 逐行复制像素数据

## 5. 平台特定实现

### 5.1 Windows 平台

#### 5.1.1 捕获器选择

**优先级**：
1. DXGI（Desktop Duplication API）
2. GDI（Graphics Device Interface）

**回退机制**：
- DXGI 失败时自动切换到 GDI
- 内存泄漏问题处理（NVidia 驱动）

#### 5.1.2 UAC 处理

```rust
fn start_uac_elevation_check()
```

**监控内容**：
- consent.exe 进程状态
- 前台窗口提升状态
- 每秒检查一次

#### 5.1.3 便携服务

- 支持便携服务模式
- 动态检测便携服务运行状态
- 状态变化时触发切换

### 5.2 Linux 平台

#### 5.2.1 显示系统检测

```rust
use crate::platform::linux::is_x11;
```

**支持**：
- X11：使用 MIT-SHM 扩展
- Wayland：使用 PipeWire 和 xdg-desktop-portal

#### 5.2.2 Wayland 处理

```rust
#[cfg(target_os = "linux")]
super::wayland::ensure_inited()?;
```

**特性**：
- 单显示器捕获限制
- 活动显示器计数管理
- 自动清理资源

#### 5.2.3 WouldBlock 处理

```rust
would_block_count += 1;
if would_block_count >= 100 {
    // Wayland 捕获器问题
}
```

### 5.3 Android 平台

#### 5.3.1 缩放检查

```rust
fn check_change_scale(hardware: bool) -> ResultType<()>
```

**功能**：
- 检测屏幕尺寸变化
- 软件编码时使用半缩放
- 硬件编码时使用全缩放

#### 5.3.2 编码失败处理

- 硬件编码器最多允许9次失败
- 软件编码器最多允许3次失败
- 首帧失败特殊处理

## 6. 消息处理

### 6.1 显示器变化消息

```rust
pub fn make_display_changed_msg(
    display_idx: usize,
    opt_display: Option<DisplayInfo>,
    source: VideoSource,
) -> Option<Message>
```

**消息内容**：
- 显示器索引
- 位置和尺寸
- 光标嵌入状态
- 支持的分辨率列表
- 原始分辨率

### 6.2 视频帧消息

```protobuf
message VideoFrame {
    uint64 timestamp = 1;
    bytes data = 2;
    uint32 display = 3;
    // ... 其他字段
}
```

## 7. 性能优化

### 7.1 帧率控制

```rust
if elapsed < spf {
    std::thread::sleep(spf - elapsed);
}
```

**目标帧率**：
- 24 FPS：最低可接受帧率
- 30 FPS：标准帧率
- 60 FPS：游戏/视频编辑场景

### 7.2 重复编码

```rust
repeat_encode_max = 10;
if repeat_encode_counter < repeat_encode_max {
    // 重复编码上一帧
}
```

**用途**：
- WouldBlock 时保持帧率
- 避免画面卡顿

### 7.3 硬件加速

#### 7.3.1 VRAM 编码器

```rust
#[cfg(feature = "vram")]
if let Some(feature) = VRamEncoder::try_get(&c.device(), negotiated_codec) {
    return EncoderCfg::VRAM(...);
}
```

**优势**：
- 直接使用显存
- 零拷贝传输
- 低延迟

#### 7.3.2 HWRAM 编码器

```rust
#[cfg(feature = "hwcodec")]
if let Some(hw) = HwRamEncoder::try_get(negotiated_codec) {
    return EncoderCfg::HWRAM(...);
}
```

**优势**：
- 硬件编码加速
- CPU 占用低
- 支持多种格式

### 7.4 内存管理

#### 7.4.1 缓冲区重用

```rust
let mut yuv = Vec::new();
let mut mid_data = Vec::new();
```

**策略**：
- 重用 YUV 缓冲区
- 避免频繁内存分配
- 减少内存碎片

#### 7.4.2 RAII 资源管理

```rust
impl Drop for Raii {
    fn drop(&mut self) {
        // 清理资源
        VIDEO_QOS.lock().unwrap().remove_display(&self.name);
        DISPLAY_CONN_IDS.lock().unwrap().remove(&self.display_idx);
    }
}
```

## 8. 错误处理

### 8.1 捕获错误

```rust
Err(err) => {
    if vs.source.is_monitor() {
        try_broadcast_display_changed(&sp, display_idx, &c, true)?;
    }
    #[cfg(windows)]
    if !c.is_gdi() {
        c.set_gdi();
        log::info!("dxgi error, fall back to gdi: {:?}", err);
        continue;
    }
    return Err(err.into());
}
```

**处理策略**：
- 检查显示器变化
- Windows：回退到 GDI
- 其他：返回错误

### 8.2 编码错误

```rust
Err(e) => {
    *encode_fail_counter += 1;
    let max_fail_times = if cfg!(target_os = "android") && encoder.is_hardware() {
        9
    } else {
        3
    };
    if (first && !repeat) || *encode_fail_counter >= max_fail_times {
        if encoder.is_hardware() {
            encoder.disable();
            bail!("SWITCH");
        }
    }
}
```

**处理策略**：
- 计数失败次数
- 超过阈值时禁用硬件编码器
- 触发服务切换

### 8.3 服务切换

**切换条件**：
- OPTION_REFRESH 选项设置
- 编码格式变化
- I444 格式变化
- 便携服务状态变化
- 隐私模式变化
- 显示器变化
- 编码失败
- UAC 状态变化
- 缩放变化（Android）

## 9. 关键函数调用关系

### 9.1 初始化流程

```
new()
  ├─ FRAME_FETCHED_NOTIFIERS 初始化
  ├─ VideoService 创建
  └─ GenericService::run()
      └─ run()
          ├─ get_capturer()
          │   ├─ get_capturer_monitor() / get_capturer_camera()
          │   └─ create_capturer()
          ├─ setup_encoder()
          │   ├─ get_encoder_config()
          │   ├─ get_recorder()
          │   └─ Encoder::new()
          └─ VideoFrameController::new()
```

### 9.2 主循环流程

```
run() 主循环
  ├─ check_uac_switch()
  ├─ check_qos()
  ├─ check_privacy_mode_changed()
  ├─ c.frame() [捕获]
  ├─ handle_one_frame()
  │   ├─ encoder.encode_to_message()
  │   ├─ recorder.write_message()
  │   └─ sp.send_video_frame()
  ├─ frame_controller.try_wait_next()
  └─ 帧率控制 sleep
```

### 9.3 编码器选择流程

```
get_encoder_config()
  ├─ negotiated_codec 检查
  ├─ H.264/H.265
  │   ├─ VRamEncoder::try_get()
  │   ├─ HwRamEncoder::try_get()
  │   └─ 回退 VP9
  ├─ VP8/VP9
  │   └─ VPX 编码器
  ├─ AV1
  │   └─ AOM 编码器
  └─ 默认 VP9
```

## 10. 配置选项

### 10.1 OPTION_REFRESH

- **用途**：强制刷新显示器
- **触发**：客户端请求
- **处理**：广播显示器变化并切换服务

### 10.2 allow-auto-record-incoming

- **用途**：自动录制传入会话
- **类型**：布尔值
- **影响**：创建录制器实例

### 10.3 enable-directx-capture

- **用途**：启用 DirectX 捕获（Windows）
- **类型**：布尔值
- **影响**：选择 DXGI 或 GDI

## 11. 日志记录

### 11.1 关键日志点

```rust
log::debug!("Create capturer dxgi|gdi");
log::info!("initial quality: {quality:?}");
log::info!("switch due to codec changed");
log::error!("encode fail: {e:?}, times: {}", *encode_fail_counter);
```

### 11.2 性能日志

```rust
log::trace!("Channel recv latency: {}", tm.elapsed().as_secs_f32());
log::trace!("{:?} {:?}", time::Instant::now(), elapsed);
```

## 12. 并发与同步

### 12.1 通知机制

```rust
type FrameFetchedNotifierSender = UnboundedSender<(i32, Option<Instant>)>;
type FrameFetchedNotifierReceiver = Arc<TokioMutex<UnboundedReceiver<(i32, Option<Instant>)>>;
```

**特点**：
- 无界通道
- 异步接收
- 跨线程安全

### 12.2 互斥锁保护

```rust
static ref VIDEO_QOS: Arc<Mutex<VideoQoS>>
static ref DISPLAY_CONN_IDS: Arc<Mutex<HashMap<usize, HashSet<i32>>>>
static ref SCREENSHOTS: Mutex<HashMap<usize, Screenshot>>
```

**保护对象**：
- QoS 参数
- 连接映射
- 截图队列

## 13. 扩展点

### 13.1 自定义编码器

通过 `Encoder::set_fallback()` 和 `Encoder::update()` 扩展

### 13.2 自定义捕获器

实现 `TraitCapturer` trait

### 13.3 自定义 QoS 策略

通过 `VideoQoS` 模块扩展

## 14. 最佳实践

### 14.1 资源管理

- 使用 RAII 模式管理资源
- 及时释放捕获器和编码器
- 避免频繁创建/销毁对象

### 14.2 错误处理

- 优雅降级（硬件→软件）
- 合理的重试机制
- 详细的错误日志

### 14.3 性能优化

- 重用缓冲区
- 批量处理消息
- 异步耗时操作

## 15. 已知问题与限制

### 15.1 Windows

- NVidia 驱动 DXGI 内存泄漏
- UAC 窗口处理复杂
- 便携服务状态检测

### 15.2 Linux

- Wayland 多显示器限制
- MIT-SHM 扩展兼容性
- WouldBlock 处理

### 15.3 Android

- 硬件编码器稳定性
- 缩放切换延迟
- 首帧编码问题

## 16. 未来改进方向

### 16.1 性能优化

- 进一步优化内存使用
- 改进帧同步机制
- 优化编码参数

### 16.2 功能增强

- 支持更多编码格式
- 改进多显示器支持
- 增强隐私模式

### 16.3 跨平台

- 统一平台接口
- 简化平台特定代码
- 改进错误处理

## 17. 参考资源

### 17.1 外部项目

- [PHZ76/DesktopSharing](https://github.com/PHZ76/DesktopSharing)：音频捕获、硬件编码、光标绘制
- [slhck.info/video](https://slhck.info/video/2017/03/01/rate-control.html)：码率控制

### 17.2 技术文档

- [Desktop Duplication API](https://docs.microsoft.com/zh-cn/windows/win32/direct3ddxgi/desktop-dup-api)
- [xdg-desktop-portal](https://flatpak.github.io/xdg-desktop-portal/docs/)

### 17.3 问题讨论

- [DXGI 内存泄漏](https://stackoverflow.com/questions/47801238/memory-leak-in-creating-direct2d-device)
- [NVIDIA DXGI 问题](https://forums.developer.nvidia.com/t/dxgi-outputduplication-memory-leak-when-using-nv-but-not-amd-drivers/108582)

## 18. 总结

`video_service.rs` 是 RustDesk 视频传输的核心模块，实现了完整的视频捕获、编码、传输流程。该模块具有以下特点：

1. **跨平台支持**：支持 Windows、Linux、Android 等多个平台
2. **高性能**：支持硬件加速、内存优化、帧率控制
3. **可扩展**：模块化设计，易于扩展新功能
4. **健壮性**：完善的错误处理和降级机制
5. **灵活性**：支持多种编码格式和质量控制策略

该模块是 RustDesk 实现流畅远程桌面体验的关键组件，其设计思路和实现细节对类似项目具有重要的参考价值。
