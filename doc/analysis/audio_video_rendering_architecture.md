# RustDesk 音视频传输渲染链路技术文档

## 文档概述

本文档详细阐述了 RustDesk 远程桌面系统中音视频传输渲染的完整实现逻辑，涵盖发送端屏幕采集、编解码处理以及接收端对象渲染的核心技术实现。文档基于 `/home/zy/repo/rustdesk/src` 目录下的源代码分析，为开发人员提供深入的技术理解和实现参考。

---

## 目录

1. [发送端屏幕采集模块](#1-发送端屏幕采集模块)
2. [编解码流程实现](#2-编解码流程实现)
3. [接收端渲染流程](#3-接收端渲染流程)
4. [模块间交互关系](#4-模块间交互关系)
5. [核心算法与技术难点](#5-核心算法与技术难点)
6. [性能优化策略](#6-性能优化策略)

---

## 1. 发送端屏幕采集模块

### 1.1 技术架构

屏幕采集模块是 RustDesk 远程桌面系统的核心组件，负责实时捕获屏幕内容并转换为可传输的图像数据。该模块采用平台特定的采集技术，确保在不同操作系统上都能高效运行。

**核心文件**：
- [video_service.rs](file:///home/zy/repo/rustdesk/src/server/video_service.rs) - 视频服务主逻辑
- [display_service.rs](file:///home/zy/repo/rustdesk/src/server/display_service.rs) - 显示器管理
- [platform/windows.rs](file:///home/zy/repo/rustdesk/src/platform/windows.rs) - Windows 平台实现
- [platform/linux.rs](file:///home/zy/repo/rustdesk/src/platform/linux.rs) - Linux 平台实现
- [platform/macos.rs](file:///home/zy/repo/rustdesk/src/platform/macos.rs) - macOS 平台实现

### 1.2 采集方式实现

#### 1.2.1 跨平台采集器创建

采集器创建函数 `create_capturer()` 实现了跨平台的屏幕采集能力：

```rust
fn create_capturer(
    privacy_mode_id: i32,
    display: Display,
    _current: usize,
    _portable_service_running: bool,
) -> ResultType<Box<dyn TraitCapturer>> {
    #[cfg(not(windows))]
    let c: Option<Box<dyn TraitCapturer>> = None;
    #[cfg(windows)]
    let mut c: Option<Box<dyn TraitCapturer>> = None;
    
    if privacy_mode_id > 0 {
        #[cfg(windows)]
        {
            if let Some(c1) = crate::privacy_mode::win_mag::create_capturer(
                privacy_mode_id,
                display.origin(),
                display.width(),
                display.height(),
            )? {
                c = Some(Box::new(c1));
            }
        }
    }

    match c {
        Some(c1) => return Ok(c1),
        None => {
            #[cfg(windows)]
            {
                log::debug!("Create capturer dxgi|gdi");
                return crate::portable_service::client::create_capturer(
                    _current,
                    display,
                    _portable_service_running,
                );
            }
            #[cfg(not(windows))]
            {
                log::debug!("Create capturer from scrap");
                return Ok(Box::new(
                    Capturer::new(display).with_context(|| "Failed to create capturer")?,
                ));
            }
        }
    };
}
```

**技术要点**：
- 使用 `scrap::Capturer` 作为统一的采集接口
- Windows 平台支持 DXGI 和 GDI 两种采集方式
- 隐私模式下使用 Magnifier API 进行屏幕捕获
- 采集器对象创建开销大，避免频繁创建

#### 1.2.2 平台特定采集技术

**Windows 平台**：
- **DXGI (DirectX Graphics Infrastructure)**：高性能采集，支持硬件加速
- **GDI (Graphics Device Interface)**：兼容性好，但性能较低
- **Magnifier API**：隐私模式专用，用于 UAC 提升场景

**Linux 平台**：
- **X11**：使用 X11 API 进行屏幕捕获
- **Wayland**：使用 PipeWire 和 Portal API（通过 `wayland.rs` 模块）

**macOS 平台**：
- **ScreenCapture API**：现代 macOS 屏幕捕获接口
- **CGDisplayStream**：传统的显示流捕获

### 1.3 帧率控制机制

帧率控制通过 VideoQoS 模块实现，根据网络状况动态调整采集帧率：

```rust
let mut video_qos = VIDEO_QOS.lock().unwrap();
let mut spf = video_qos.spf(); // Seconds Per Frame
let mut quality = video_qos.ratio();
```

**关键参数**：
- `FPS: u32 = 30` - 目标帧率
- `MIN_FPS: u32 = 1` - 最小帧率
- `MAX_FPS: u32 = 120` - 最大帧率
- `INIT_FPS: u32 = 15` - 初始帧率

**帧率调整策略**：
1. 新用户连接时设置为 `INIT_FPS`
2. 根据 `TestDelay` 接收的网络延迟更新 FPS
3. 网络延迟 < 150ms：提高帧率
4. 网络延迟 >= 150ms：降低帧率

### 1.4 区域选择机制

#### 1.4.1 多显示器支持

系统支持多显示器采集，通过 `Display` 结构体管理：

```rust
pub(super) struct CapturerInfo {
    pub origin: (i32, i32),      // 显示器原点坐标
    pub width: usize,            // 宽度
    pub height: usize,           // 高度
    pub ndisplay: usize,         // 显示器总数
    pub current: usize,          // 当前显示器索引
    pub privacy_mode_id: i32,    // 隐私模式 ID
    pub _capturer_privacy_mode_id: i32,
    pub capturer: Box<dyn TraitCapturer>, // 采集器实例
}
```

#### 1.4.2 显示器变化检测

`display_service.rs` 实现了显示器变化的实时检测：

```rust
pub(super) fn check_display_changed(
    ndisplay: usize,
    idx: usize,
    (x, y, w, h): (i32, i32, usize, usize),
) -> Option<DisplayInfo> {
    // 检测显示器数量变化
    // 检测显示器分辨率变化
    // 触发显示器切换通知
}
```

**检测机制**：
- 定期检查显示器配置（每秒一次）
- 比较显示器数量和分辨率
- 检测到变化时发送 `SwitchDisplay` 消息

#### 1.4.3 自定义分辨率支持

客户端可以请求自定义分辨率：

```rust
pub fn get_custom_resolution(&self, display: i32) -> Option<(i32, i32)> {
    self.config
        .custom_resolutions
        .get(&display.to_string())
        .map(|r| (r.w, r.h))
}
```

### 1.5 采集循环实现

主采集循环在 `run()` 函数中实现：

```rust
fn run(vs: VideoService) -> ResultType<()> {
    // 初始化采集器和编码器
    let mut c = get_capturer(vs.source, display_idx, last_portable_service_running)?;
    
    while sp.ok() {
        // 检查 QoS 参数
        check_qos(
            &mut encoder,
            &mut quality,
            &mut spf,
            client_record,
            &mut send_counter,
            &mut second_instant,
            &sp.name(),
        )?;
        
        // 检查显示器变化
        if vs.source.is_monitor() && last_check_displays.elapsed().as_millis() > 1000 {
            last_check_displays = now;
            try_broadcast_display_changed(&sp, display_idx, &c, false)?;
        }
        
        // 捕获帧
        let res = match c.frame(spf) {
            Ok(frame) => {
                if frame.valid() {
                    // 处理有效帧
                    let frame = frame.to(encoder.yuvfmt(), &mut yuv, &mut mid_data)?;
                    let send_conn_ids = handle_one_frame(
                        display_idx,
                        &sp,
                        frame,
                        ms,
                        &mut encoder,
                        recorder.clone(),
                        &mut encode_fail_counter,
                        &mut first_frame,
                        capture_width,
                        capture_height,
                    )?;
                }
                Ok(())
            }
            Err(err) => Err(err),
        };
        
        // 处理采集错误
        match res {
            Err(ref e) if e.kind() == WouldBlock => {
                // 无数据可采集，继续等待
            }
            Err(e) => {
                log::error!("Capture error: {:?}", e);
                return Err(e);
            }
            _ => {}
        }
    }
}
```

**关键特性**：
- 非阻塞采集：使用 `WouldBlock` 错误类型处理无数据情况
- 帧有效性检查：`frame.valid()` 确保只处理有效帧
- 格式转换：将捕获的帧转换为编码器所需的 YUV 格式
- 错误恢复：编码失败时自动重试（最多 10 次）

### 1.6 隐私模式实现

隐私模式通过 Magnifier API 实现，用于 UAC 提升场景：

```rust
fn check_uac_switch(privacy_mode_id: i32, capturer_privacy_mode_id: i32) -> ResultType<()> {
    if capturer_privacy_mode_id != INVALID_PRIVACY_MODE_CONN_ID
        && is_current_privacy_mode_impl(PRIVACY_MODE_IMPL_WIN_MAG)
    {
        if !is_installed() {
            if privacy_mode_id != capturer_privacy_mode_id {
                if !is_process_consent_running()? {
                    bail!("consent.exe is not running");
                }
            }
            if is_process_consent_running()? {
                bail!("consent.exe is running");
            }
        }
    }
    Ok(())
}
```

**工作原理**：
- 检测 `consent.exe` 进程（UAC 提升对话框）
- 切换到 Magnifier API 进行屏幕捕获
- 避免捕获 UAC 对话框内容

---

## 2. 编解码流程实现

### 2.1 编码器架构

RustDesk 支持多种编码格式和编码器类型，采用分层架构设计：

**编码器类型**：
- **软件编码器**：VP8, VP9, AV1 (libaom)
- **硬件编码器**：H264, H265 (通过 VRamEncoder 和 HwRamEncoder)
- **混合编码器**：根据硬件能力自动选择

**编码器配置结构**：

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
) -> ResultType<(
    Encoder,
    EncoderCfg,
    CodecFormat,
    bool,
    Arc<Mutex<Option<Recorder>>>,
)> {
    let encoder_cfg = get_encoder_config(
        &c,
        name.to_string(),
        quality,
        client_record || record_incoming,
        last_portable_service_running,
        source,
    );
    Encoder::set_fallback(&encoder_cfg);
    let codec_format = Encoder::negotiated_codec();
    let recorder = get_recorder(record_incoming, display_idx, source == VideoSource::Camera);
    let use_i444 = Encoder::use_i444(&encoder_cfg);
    let encoder = Encoder::new(encoder_cfg.clone(), use_i444)?;
    Ok((encoder, encoder_cfg, codec_format, use_i444, recorder))
}
```

### 2.2 编码格式选择

编码格式选择基于协商机制和硬件能力：

```rust
fn get_encoder_config(
    c: &CapturerInfo,
    _name: String,
    quality: f32,
    record: bool,
    _portable_service: bool,
    _source: VideoSource,
) -> EncoderCfg {
    let negotiated_codec = Encoder::negotiated_codec();
    match negotiated_codec {
        CodecFormat::H264 | CodecFormat::H265 => {
            // 优先尝试硬件编码
            #[cfg(feature = "vram")]
            if let Some(feature) = VRamEncoder::try_get(&c.device(), negotiated_codec) {
                return EncoderCfg::VRAM(VRamEncoderConfig {
                    device: c.device(),
                    width: c.width,
                    height: c.height,
                    quality,
                    feature,
                    keyframe_interval,
                });
            }
            // 尝试硬件 RAM 编码
            #[cfg(feature = "hwcodec")]
            if let Some(hw) = HwRamEncoder::try_get(negotiated_codec) {
                return EncoderCfg::HWRAM(HwRamEncoderConfig {
                    name: hw.name,
                    mc_name: hw.mc_name,
                    width: c.width,
                    height: c.height,
                    quality,
                    keyframe_interval,
                });
            }
            // 回退到 VP9 软件编码
            EncoderCfg::VPX(VpxEncoderConfig {
                width: c.width as _,
                height: c.height as _,
                quality,
                codec: VpxVideoCodecId::VP9,
                keyframe_interval,
            })
        }
        format @ (CodecFormat::VP8 | CodecFormat::VP9) => EncoderCfg::VPX(VpxEncoderConfig {
            width: c.width as _,
            height: c.height as _,
            quality,
            codec: if format == CodecFormat::VP8 {
                VpxVideoCodecId::VP8
            } else {
                VpxVideoCodecId::VP9
            },
            keyframe_interval,
        }),
        CodecFormat::AV1 => EncoderCfg::AOM(AomEncoderConfig {
            width: c.width as _,
            height: c.height as _,
            quality,
            keyframe_interval,
        }),
        _ => EncoderCfg::VPX(VpxEncoderConfig {
            width: c.width as _,
            height: c.height as _,
            quality,
            codec: VpxVideoCodecId::VP9,
            keyframe_interval,
        }),
    }
}
```

**选择策略**：
1. 优先使用客户端协商的编码格式
2. H264/H265：优先尝试 VRAM 硬件编码，其次 HWRAM，最后回退到 VP9
3. VP8/VP9：使用 libvpx 软件编码
4. AV1：使用 libaom 软件编码
5. 其他格式：默认使用 VP9

### 2.3 压缩算法实现

#### 2.3.1 帧编码流程

```rust
let frame = frame.to(encoder.yuvfmt(), &mut yuv, &mut mid_data)?;
let send_conn_ids = handle_one_frame(
    display_idx,
    &sp,
    frame,
    ms,
    &mut encoder,
    recorder.clone(),
    &mut encode_fail_counter,
    &mut first_frame,
    capture_width,
    capture_height,
)?;
```

**编码步骤**：
1. **格式转换**：将捕获的帧转换为编码器所需的 YUV 格式
2. **编码处理**：调用编码器进行压缩
3. **失败重试**：编码失败时自动重试（最多 10 次）
4. **记录保存**：如果启用了录制，保存原始帧

#### 2.3.2 关键帧间隔控制

关键帧间隔根据录制状态动态调整：

```rust
let keyframe_interval = if record { Some(240) } else { None };
```

- **录制模式**：每 240 帧插入一个关键帧（约 8 秒）
- **正常模式**：使用编码器默认的关键帧间隔

### 2.4 码率控制策略

码率控制通过 VideoQoS 模块实现，采用自适应算法：

#### 2.4.1 码率参数

```rust
const BR_MAX: f32 = 40.0;                    // 最大码率倍数 (2000 * 2 / 100)
const BR_MIN: f32 = 0.2;                     // 最小码率倍数
const BR_MIN_HIGH_RESOLUTION: f32 = 0.1;    // 高分辨率最小码率
const MAX_BR_MULTIPLE: f32 = 1.0;            // 最大码率倍数
```

#### 2.4.2 码率调整算法

```rust
impl UserDelay {
    fn add_delay(&mut self, delay: u32) {
        self.rtt_calculator.update(delay);
        if self.delay_history.len() > HISTORY_DELAY_LEN {
            self.delay_history.pop_front();
        }
        self.delay_history.push_back(delay);
    }

    fn avg_delay(&self) -> u32 {
        let len = self.delay_history.len();
        if len > 0 {
            let avg_delay = self.delay_history.iter().sum::<u32>() / len as u32;

            // 减去 RTT 得到实际网络延迟
            if let Some(rtt) = self.rtt_calculator.get_rtt() {
                if avg_delay > rtt {
                    avg_delay - rtt
                } else {
                    avg_delay
                }
            } else {
                avg_delay
            }
        } else {
            DELAY_THRESHOLD_150MS
        }
    }
}
```

**调整策略**：
1. **延迟计算**：使用历史延迟平均值减去 RTT
2. **网络条件判断**：
   - 延迟 < 150ms：网络良好，提高码率和帧率
   - 延迟 >= 150ms：网络较差，降低码率和帧率
3. **动态调整**：每 3 秒调整一次质量比率
4. **屏幕活动检测**：如果一秒内编码超过 2 次，允许提高质量

#### 2.4.3 FPS 与码率平衡

```rust
const DELAY_THRESHOLD_150MS: u32 = 150;

adjust between FPS and ratio:
    When network delay < DELAY_THRESHOLD_150MS:
        - fps is always higher than the minimum fps
        - ratio is increasing
    When network delay >= DELAY_THRESHOLD_150MS:
        - fps is always lower than the minimum fps
        - ratio is decreasing
```

**平衡策略**：
- 网络良好时：优先保证流畅度（高 FPS），逐步提高质量
- 网络较差时：优先保证质量（降低 FPS），逐步降低码率

### 2.5 硬件加速支持

#### 2.5.1 VRAM 编码器

```rust
#[cfg(feature = "vram")]
if let Some(feature) = VRamEncoder::try_get(&c.device(), negotiated_codec) {
    return EncoderCfg::VRAM(VRamEncoderConfig {
        device: c.device(),
        width: c.width,
        height: c.height,
        quality,
        feature,
        keyframe_interval,
    });
}
```

**特性**：
- 直接使用 GPU 显存进行编码
- 最小化 CPU 和 GPU 之间的数据传输
- 适用于支持 DirectX 12 的 Windows 系统

#### 2.5.2 HWRAM 编码器

```rust
#[cfg(feature = "hwcodec")]
if let Some(hw) = HwRamEncoder::try_get(negotiated_codec) {
    return EncoderCfg::HWRAM(HwRamEncoderConfig {
        name: hw.name,
        mc_name: hw.mc_name,
        width: c.width,
        height: c.height,
        quality,
        keyframe_interval,
    });
}
```

**特性**：
- 使用硬件编码器（如 Intel Quick Sync, NVIDIA NVENC）
- 数据在系统内存中传输
- 兼容性更好，但性能略低于 VRAM 编码器

### 2.6 编码失败处理

```rust
let mut repeat_encode_counter = 0;
const repeat_encode_max = 10;
let mut encode_fail_counter = 0;

// 编码失败重试逻辑
if !encoder.latency_free() && yuv.len() > 0 {
    if repeat_encode_counter < repeat_encode_max {
        repeat_encode_counter += 1;
        let res = encoder.encode(
            EncodeInput::YUV(&yuv),
            &mut encode_fail_counter,
        );
        // 处理编码结果
    }
}
```

**失败处理策略**：
1. **重试机制**：最多重试 10 次
2. **失败计数器**：跟踪连续失败次数
3. **自动回退**：硬件编码失败时自动切换到软件编码
4. **日志记录**：记录失败原因和次数

### 2.7 编码器切换机制

```rust
if codec_format != Encoder::negotiated_codec() {
    log::info!(
        "switch due to codec changed, {:?} -> {:?}",
        codec_format,
        Encoder::negotiated_codec()
    );
    bail!("SWITCH");
}

if Encoder::use_i444(&encoder_cfg) != use_i444 {
    log::info!("switch due to i444 changed");
    bail!("SWITCH");
}

#[cfg(all(windows, feature = "vram"))]
if c.is_gdi() && encoder.input_texture() {
    log::info!("changed to gdi when using vram");
    VRamEncoder::set_fallback_gdi(sp.name(), true);
    bail!("SWITCH");
}
```

**切换触发条件**：
1. 编码格式协商变更
2. I444 色彩空间设置变更
3. 采集方式变更（GDI ↔ DXGI）
4. 便携服务运行状态变更

---

## 3. 接收端渲染流程

### 3.1 渲染引擎架构

接收端渲染采用多线程架构，将解码和渲染分离：

**核心组件**：
- **VideoHandler**：视频帧处理器，负责解码
- **VideoThread**：视频处理线程，管理视频队列
- **ImageRgb**：RGB 图像缓冲区
- **ImageTexture**：纹理缓冲区（硬件加速）

### 3.2 解码器实现

#### 3.2.1 VideoHandler 结构

```rust
pub struct VideoHandler {
    decoder: Decoder,                          // 解码器实例
    pub rgb: ImageRgb,                         // RGB 图像缓冲区
    pub texture: ImageTexture,                 // 纹理缓冲区
    recorder: Arc<Mutex<Option<Recorder>>>,   // 录制器
    record: bool,                              // 录制状态
    _display: usize,                           // 显示器索引
    fail_counter: usize,                       // 失败计数器
    first_frame: bool,                         // 首帧标志
}
```

#### 3.2.2 解码器初始化

```rust
pub fn new(format: CodecFormat, _display: usize) -> Self {
    let luid = Self::get_adapter_luid();
    log::info!("new video handler for display #{_display}, format: {format:?}, luid: {luid:?}");
    let rgba_format =
        if cfg!(feature = "flutter") && (cfg!(windows) || cfg!(target_os = "linux")) {
            ImageFormat::ABGR
        } else {
            ImageFormat::ARGB
        };
    VideoHandler {
        decoder: Decoder::new(format, luid),
        rgb: ImageRgb::new(rgba_format, crate::get_dst_align_rgba()),
        texture: Default::default(),
        recorder: Default::default(),
        record: false,
        _display,
        fail_counter: 0,
        first_frame: true,
    }
}
```

**初始化要点**：
- 根据平台选择 RGBA 格式（ABGR 或 ARGB）
- 获取 GPU 适配器 LUID（用于硬件解码）
- 初始化 RGB 和纹理缓冲区

#### 3.2.3 视频帧处理

```rust
pub fn handle_frame(
    &mut self,
    vf: VideoFrame,
    pixelbuffer: &mut bool,
    chroma: &mut Option<Chroma>,
) -> ResultType<bool> {
    let format = CodecFormat::from(&vf);
    if format != self.decoder.format() {
        self.reset(Some(format));
    }
    match &vf.union {
        Some(frame) => {
            let res = self.decoder.handle_video_frame(
                frame,
                &mut self.rgb,
                &mut self.texture,
                pixelbuffer,
                chroma,
            );
            if res.as_ref().is_ok_and(|x| *x) {
                self.fail_counter = 0;
            } else {
                if self.fail_counter < usize::MAX {
                    if self.first_frame && self.fail_counter < MAX_DECODE_FAIL_COUNTER {
                        log::error!("decode first frame failed");
                        self.fail_counter = MAX_DECODE_FAIL_COUNTER;
                    } else {
                        self.fail_counter += 1;
                    }
                    log::error!(
                        "Failed to handle video frame, fail counter: {}",
                        self.fail_counter
                    );
                }
            }
            self.first_frame = false;
            if self.record {
                self.recorder.lock().unwrap().as_mut().map(|r| {
                    let (w, h) = if *pixelbuffer {
                        (self.rgb.w, self.rgb.h)
                    } else {
                        (self.texture.w, self.texture.h)
                    };
                    r.write_frame(frame, w, h).ok();
                });
            }
            res
        }
        _ => Ok(false),
    }
}
```

**处理流程**：
1. **格式检查**：检测编码格式变化，必要时重置解码器
2. **解码处理**：调用解码器处理视频帧
3. **失败处理**：跟踪失败次数，首帧失败时快速重试
4. **录制保存**：如果启用录制，保存解码后的帧

### 3.3 视频队列管理

#### 3.3.1 VideoThread 结构

```rust
struct VideoThread {
    video_queue: Arc<RwLock<ArrayQueue<VideoFrame>>>,  // 视频帧队列
    video_sender: Sender<MediaData>,                    // 消息发送器
    decode_fps: Arc<RwLock<Option<usize>>>,             // 解码帧率
    frame_count: Arc<RwLock<0>>,                        // 帧计数器
    fps_control: Default,                               // FPS 控制
    discard_queue: Arc<RwLock<bool>>,                    // 队列丢弃标志
}
```

#### 3.3.2 队列创建

```rust
fn new_video_thread(&mut self, display: usize) {
    let video_queue = Arc::new(RwLock::new(ArrayQueue::new(client::VIDEO_QUEUE_SIZE)));
    let (video_sender, video_receiver) = std::sync::mpsc::channel::<MediaData>();
    let decode_fps = Arc::new(RwLock::new(None));
    let frame_count = Arc::new(RwLock::new(0));
    let discard_queue = Arc::new(RwLock::new(false));
    
    let video_thread = VideoThread {
        video_queue: video_queue.clone(),
        video_sender,
        decode_fps: decode_fps.clone(),
        frame_count: frame_count.clone(),
        fps_control: Default::default(),
        discard_queue: discard_queue.clone(),
    };
    
    // 启动视频处理线程
    crate::client::start_video_thread(
        self.handler.clone(),
        display,
        video_receiver,
        video_queue,
        decode_fps,
        self.chroma.clone(),
        discard_queue,
        move |display: usize,
              data: &mut scrap::ImageRgb,
              _texture: *mut c_void,
              pixelbuffer: bool| {
            *frame_count.write().unwrap() += 1;
            if pixelbuffer {
                handler.on_rgba(display, data);
            } else {
                #[cfg(all(feature = "vram", feature = "flutter"))]
                handler.on_texture(display, _texture);
            }
        },
    );
    
    self.video_threads.insert(display, video_thread);
}
```

**队列特性**：
- **固定大小**：`VIDEO_QUEUE_SIZE = 120`
- **无锁队列**：使用 `ArrayQueue` 实现高性能队列
- **多显示器支持**：每个显示器独立队列

### 3.4 渲染性能优化

#### 3.4.1 帧率自适应控制

```rust
let min_decode_fps = self
    .video_threads
    .values()
    .map(|v| *v.1.decode_fps.read().unwrap())
    .min();

let Some(min_decode_fps) = min_decode_fps else {
    return;
};

let limited_fps = if min_decode_fps < 30 {
    min_decode_fps * 9 / 10 // 30 got 27
} else {
    min_decode_fps * 4 / 5 // 30 got 24
};

let len = thread.video_queue.read().unwrap().len();
let decode_fps = thread.decode_fps.read().unwrap().clone()?;

if len > 1 && last_auto_fps > limited_fps || len > std::cmp::max(1, decode_fps / 2) {
    // 降低 FPS
}
```

**优化策略**：
1. **解码帧率监控**：实时监控每个显示器的解码帧率
2. **队列长度检查**：根据队列长度动态调整 FPS
3. **自适应限制**：
   - 低帧率（< 30fps）：限制为当前帧率的 90%
   - 高帧率（>= 30fps）：限制为当前帧率的 80%

#### 3.4.2 队列丢弃策略

```rust
let video_queue = thread.video_queue.read().unwrap();
let tolerable = std::cmp::min(min_decode_fps, video_queue.capacity() / 2);

if !thread.discard_queue.read().unwrap()
    && (video_queue.len() > tolerable
        || thread.video_sender.send(MediaData::Discard).is_err())
{
    drop(video_queue);
    thread.discard_queue.write().unwrap().true;
}
```

**丢弃策略**：
1. **阈值判断**：队列长度超过 `min(decode_fps, capacity/2)`
2. **发送丢弃信号**：通知解码线程丢弃旧帧
3. **状态标志**：使用 `discard_queue` 标志避免重复丢弃

#### 3.4.3 强制推送机制

```rust
let video_queue = thread.video_queue.read().unwrap();
if video_queue.force_push(vf).is_some() {
    drop(video_queue);
    thread.discard_queue.write().unwrap().true;
}
```

**强制推送**：
- 当队列满时，强制丢弃最旧的帧
- 确保新帧能够进入队列
- 触发丢弃标志，通知解码线程

### 3.5 渲染回调机制

#### 3.5.1 RGB 渲染

```rust
move |display: usize,
      data: &mut scrap::ImageRgb,
      _texture: *mut c_void,
      pixelbuffer: bool| {
    *frame_count.write().unwrap() += 1;
    if pixelbuffer {
        handler.on_rgba(display, data);
    } else {
        #[cfg(all(feature = "vram", feature = "flutter"))]
        handler.on_texture(display, _texture);
    }
}
```

**渲染路径**：
1. **PixelBuffer 模式**：使用 `on_rgba()` 回调渲染 RGB 数据
2. **Texture 模式**：使用 `on_texture()` 回调渲染纹理数据（硬件加速）

#### 3.5.2 硬件加速渲染

```rust
#[cfg(all(feature = "vram", feature = "flutter"))]
handler.on_texture(display, _texture);
```

**硬件加速特性**：
- 直接使用 GPU 纹理
- 避免数据拷贝到 CPU
- 适用于 Flutter 平台

### 3.6 解码器重置机制

```rust
pub fn reset(&mut self, format: Option<CodecFormat>) {
    log::info!(
        "reset video handler for display #{}, format: {format:?}",
        self._display
    );
    #[cfg(target_os = "macos")]
    self.rgb.set_align(crate::get_dst_align_rgba());
    let luid = Self::get_adapter_luid();
    let format = format.unwrap_or(self.decoder.format());
    self.decoder = Decoder::new(format, luid);
    self.fail_counter = 0;
    self.first_frame = true;
}
```

**重置场景**：
1. 编码格式变更
2. 解码失败次数过多
3. 显卡适配器变更

### 3.7 录制功能

```rust
pub fn record_screen(&mut self, start: bool, id: String, display_idx: usize, camera: bool) {
    self.record = false;
    if start {
        self.recorder = Recorder::new(RecorderContext {
            server: false,
            id,
            dir: crate::ui_interface::video_save_directory(false),
            display_idx,
            camera,
            tx: None,
        })
        .map_or(Default::default(), |r| Arc::new(Mutex::new(Some(r))));
    } else {
        self.recorder = Default::default();
    }

    self.record = start;
}
```

**录制实现**：
- 使用 `Recorder` 组件保存解码后的帧
- 支持显示器和摄像头录制
- 可配置保存目录

---

## 4. 模块间交互关系

### 4.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        发送端 (Server)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │ Display      │───▶│ Capturer     │───▶│ Encoder      │    │
│  │ Service      │    │ (Platform)   │    │ (Codec)      │    │
│  └──────────────┘    └──────────────┘    └──────────────┘    │
│         │                   │                   │              │
│         │                   │                   │              │
│         ▼                   ▼                   ▼              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │ Multi-       │    │ Frame        │    │ Bitrate      │    │
│  │ Display      │    │ Rate Control │    │ Control      │    │
│  │ Management   │    │ (QoS)        │    │ (QoS)        │    │
│  └──────────────┘    └──────────────┘    └──────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                              │
                              │ Network (TCP/UDP)
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                        接收端 (Client)                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │ Video        │───▶│ Decoder      │───▶│ Renderer     │    │
│  │ Queue        │    │ (Codec)      │    │ (Platform)   │    │
│  └──────────────┘    └──────────────┘    └──────────────┘    │
│         │                   │                   │              │
│         │                   │                   │              │
│         ▼                   ▼                   ▼              │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    │
│  │ Frame        │    │ Decode       │    │ Display      │    │
│  │ Discard      │    │ FPS Control  │    │ Management   │    │
│  │ Strategy     │    │              │    │              │    │
│  └──────────────┘    └──────────────┘    └──────────────┘    │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 数据流向图

```
屏幕内容
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 1. 屏幕采集 (Capturer)                                      │
│    - 平台特定 API (DXGI/GDI/X11/ScreenCapture)              │
│    - 帧率控制 (QoS)                                         │
│    - 区域选择 (多显示器)                                    │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 2. 格式转换 (Frame::to)                                     │
│    - 转换为 YUV 格式                                        │
│    - 色彩空间转换 (I420/I444)                               │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 3. 编码压缩 (Encoder)                                       │
│    - 编码格式选择 (VP8/VP9/H264/H265/AV1)                   │
│    - 硬件加速 (VRAM/HWRAM)                                  │
│    - 码率控制 (QoS)                                         │
│    - 关键帧插入                                              │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 4. 网络传输 (Network)                                       │
│    - TCP/UDP 协议                                           │
│    - 数据包封装                                              │
│    - 拥塞控制                                                │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 5. 视频队列 (VideoQueue)                                    │
│    - 固定大小队列 (120 帧)                                   │
│    - 队列丢弃策略                                            │
│    - 强制推送机制                                            │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 6. 解码处理 (Decoder)                                       │
│    - 编码格式检测                                            │
│    - 硬件/软件解码                                           │
│    - 失败重试机制                                            │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────┐
│ 7. 渲染显示 (Renderer)                                       │
│    - RGB/Texture 模式                                        │
│    - 硬件加速渲染                                            │
│    - 帧率自适应控制                                          │
└─────────────────────────────────────────────────────────────┘
    │
    ▼
显示输出
```

### 4.3 关键模块交互

#### 4.3.1 采集与编码交互

```rust
// 采集循环
while sp.ok() {
    // 检查 QoS 参数
    check_qos(&mut encoder, &mut quality, &mut spf, ...)?;
    
    // 捕获帧
    let res = match c.frame(spf) {
        Ok(frame) => {
            if frame.valid() {
                // 格式转换
                let frame = frame.to(encoder.yuvfmt(), &mut yuv, &mut mid_data)?;
                
                // 编码处理
                let send_conn_ids = handle_one_frame(
                    display_idx, &sp, frame, ms,
                    &mut encoder, recorder.clone(),
                    &mut encode_fail_counter,
                    &mut first_frame,
                    capture_width, capture_height,
                )?;
            }
            Ok(())
        }
        Err(err) => Err(err),
    };
}
```

#### 4.3.2 解码与渲染交互

```rust
// 视频处理线程
crate::client::start_video_thread(
    self.handler.clone(),
    display,
    video_receiver,
    video_queue,
    decode_fps,
    self.chroma.clone(),
    discard_queue,
    move |display: usize,
          data: &mut scrap::ImageRgb,
          _texture: *mut c_void,
          pixelbuffer: bool| {
        *frame_count.write().unwrap() += 1;
        if pixelbuffer {
            handler.on_rgba(display, data);
        } else {
            #[cfg(all(feature = "vram", feature = "flutter"))]
            handler.on_texture(display, _texture);
        }
    },
);
```

#### 4.3.3 QoS 控制交互

```rust
// 发送端 QoS
fn check_qos(
    encoder: &mut Encoder,
    quality: &mut f32,
    spf: &mut Duration,
    client_record: bool,
    send_counter: &mut i32,
    second_instant: &mut Instant,
    name: &str,
) -> ResultType<()> {
    // 更新码率和帧率
    let mut video_qos = VIDEO_QOS.lock().unwrap();
    *spf = video_qos.spf();
    *quality = video_qos.ratio();
    
    // 更新编码器参数
    encoder.update_bitrate(*quality);
    
    Ok(())
}

// 接收端 QoS
let min_decode_fps = self
    .video_threads
    .values()
    .map(|v| *v.1.decode_fps.read().unwrap())
    .min();

let limited_fps = if min_decode_fps < 30 {
    min_decode_fps * 9 / 10
} else {
    min_decode_fps * 4 / 5
};
```

---

## 5. 核心算法与技术难点

### 5.1 自适应码率控制算法

#### 5.1.1 算法原理

自适应码率控制算法基于网络延迟动态调整编码参数，实现流畅度和质量的平衡。

**算法流程**：

```
1. 收集网络延迟数据
   ├─ 计算平均延迟
   ├─ 减去 RTT 得到实际网络延迟
   └─ 维护历史延迟队列（长度 2）

2. 判断网络条件
   ├─ 延迟 < 150ms：网络良好
   └─ 延迟 >= 150ms：网络较差

3. 调整编码参数
   ├─ 网络良好：
   │  ├─ 提高帧率（不超过最大 FPS）
   │  ├─ 提高质量比率（不超过最大码率）
   │  └─ 每 3 秒调整一次
   └─ 网络较差：
      ├─ 降低帧率（不低于最小 FPS）
      ├─ 降低质量比率（不低于最小码率）
      └─ 每 3 秒调整一次

4. FPS 与码率平衡
   ├─ 网络良好时：优先保证流畅度
   └─ 网络较差时：优先保证质量
```

#### 5.1.2 代码实现

```rust
impl UserDelay {
    fn add_delay(&mut self, delay: u32) {
        self.rtt_calculator.update(delay);
        if self.delay_history.len() > HISTORY_DELAY_LEN {
            self.delay_history.pop_front();
        }
        self.delay_history.push_back(delay);
    }

    fn avg_delay(&self) -> u32 {
        let len = self.delay_history.len();
        if len > 0 {
            let avg_delay = self.delay_history.iter().sum::<u32>() / len as u32;

            if let Some(rtt) = self.rtt_calculator.get_rtt() {
                if avg_delay > rtt {
                    avg_delay - rtt
                } else {
                    avg_delay
                }
            } else {
                avg_delay
            }
        } else {
            DELAY_THRESHOLD_150MS
        }
    }
}
```

#### 5.1.3 技术难点

1. **延迟计算准确性**
   - 需要准确计算 RTT
   - 区分网络延迟和编码延迟
   - 处理延迟抖动

2. **参数调整平滑性**
   - 避免频繁调整导致画面闪烁
   - 使用历史数据平滑调整
   - 设置调整间隔（3 秒）

3. **多用户场景**
   - 每个用户独立跟踪延迟
   - 选择最保守的参数
   - 动态调整适应新用户

### 5.2 硬件加速编码

#### 5.2.1 技术原理

硬件加速编码利用 GPU 的专用编码单元进行视频压缩，大幅降低 CPU 占用。

**编码器类型**：

1. **VRAM 编码器**
   - 直接使用 GPU 显存
   - 最小化数据传输
   - 需要 DirectX 12 支持

2. **HWRAM 编码器**
   - 使用硬件编码器 API
   - 数据在系统内存传输
   - 兼容性更好

#### 5.2.2 编码器选择逻辑

```rust
fn get_encoder_config(...) -> EncoderCfg {
    let negotiated_codec = Encoder::negotiated_codec();
    match negotiated_codec {
        CodecFormat::H264 | CodecFormat::H265 => {
            // 优先尝试 VRAM 编码
            #[cfg(feature = "vram")]
            if let Some(feature) = VRamEncoder::try_get(&c.device(), negotiated_codec) {
                return EncoderCfg::VRAM(VRamEncoderConfig { ... });
            }
            
            // 尝试 HWRAM 编码
            #[cfg(feature = "hwcodec")]
            if let Some(hw) = HwRamEncoder::try_get(negotiated_codec) {
                return EncoderCfg::HWRAM(HwRamEncoderConfig { ... });
            }
            
            // 回退到软件编码
            EncoderCfg::VPX(VpxEncoderConfig { ... })
        }
        // ... 其他格式
    }
}
```

#### 5.2.3 技术难点

1. **硬件能力检测**
   - 检测 GPU 编码能力
   - 查询支持的编码格式
   - 处理驱动兼容性问题

2. **编码器切换**
   - 硬件编码失败时自动切换到软件编码
   - 避免频繁切换导致画面中断
   - 记录失败原因用于调试

3. **性能优化**
   - 最小化 CPU-GPU 数据传输
   - 使用异步编码避免阻塞
   - 合理设置编码参数

### 5.3 视频队列管理

#### 5.3.1 队列设计

视频队列采用固定大小的无锁队列，平衡内存占用和性能。

**队列特性**：
- 容量：120 帧
- 数据结构：`ArrayQueue`
- 线程安全：`Arc<RwLock<>>`

#### 5.3.2 丢弃策略

```rust
let video_queue = thread.video_queue.read().unwrap();
let tolerable = std::cmp::min(min_decode_fps, video_queue.capacity() / 2);

if !thread.discard_queue.read().unwrap()
    && (video_queue.len() > tolerable
        || thread.video_sender.send(MediaData::Discard).is_err())
{
    drop(video_queue);
    thread.discard_queue.write().unwrap().true;
}
```

**丢弃条件**：
1. 队列长度超过 `min(decode_fps, capacity/2)`
2. 发送丢弃信号失败

#### 5.3.3 技术难点

1. **队列溢出处理**
   - 避免内存泄漏
   - 确保新帧能够进入队列
   - 使用强制推送机制

2. **帧同步**
   - 处理乱序帧
   - 关键帧优先
   - 避免画面撕裂

3. **性能优化**
   - 使用无锁队列减少锁竞争
   - 批量处理帧减少上下文切换
   - 异步解码避免阻塞

### 5.4 多显示器支持

#### 5.4.1 显示器管理

```rust
pub(super) struct CapturerInfo {
    pub origin: (i32, i32),
    pub width: usize,
    pub height: usize,
    pub ndisplay: usize,
    pub current: usize,
    pub privacy_mode_id: i32,
    pub _capturer_privacy_mode_id: i32,
    pub capturer: Box<dyn TraitCapturer>,
}
```

#### 5.4.2 显示器变化检测

```rust
pub(super) fn check_display_changed(
    ndisplay: usize,
    idx: usize,
    (x, y, w, h): (i32, i32, usize, usize),
) -> Option<DisplayInfo> {
    // 检测显示器数量变化
    // 检测显示器分辨率变化
    // 触发显示器切换通知
}
```

#### 5.4.3 技术难点

1. **显示器热插拔**
   - 实时检测显示器变化
   - 重新初始化采集器
   - 通知客户端更新

2. **多显示器同步**
   - 确保所有显示器帧率一致
   - 处理不同分辨率
   - 协调编码参数

3. **性能优化**
   - 每个显示器独立采集线程
   - 共享编码器资源
   - 负载均衡

### 5.5 隐私模式实现

#### 5.5.1 技术原理

隐私模式通过 Magnifier API 实现，避免捕获 UAC 提升对话框。

**工作流程**：
1. 检测 `consent.exe` 进程
2. 切换到 Magnifier API
3. 避免捕获敏感内容
4. UAC 对话框关闭后恢复

#### 5.5.2 代码实现

```rust
fn check_uac_switch(privacy_mode_id: i32, capturer_privacy_mode_id: i32) -> ResultType<()> {
    if capturer_privacy_mode_id != INVALID_PRIVACY_MODE_CONN_ID
        && is_current_privacy_mode_impl(PRIVACY_MODE_IMPL_WIN_MAG)
    {
        if !is_installed() {
            if privacy_mode_id != capturer_privacy_mode_id {
                if !is_process_consent_running()? {
                    bail!("consent.exe is not running");
                }
            }
            if is_process_consent_running()? {
                bail!("consent.exe is running");
            }
        }
    }
    Ok(())
}
```

#### 5.5.3 技术难点

1. **进程检测**
   - 准确检测 UAC 对话框
   - 处理进程启动延迟
   - 避免误判

2. **采集器切换**
   - 无缝切换采集方式
   - 避免画面中断
   - 保持编码参数一致

3. **性能影响**
   - Magnifier API 性能较低
   - 需要优化采集频率
   - 减少内存占用

---

## 6. 性能优化策略

### 6.1 发送端优化

#### 6.1.1 采集优化

1. **避免频繁创建采集器**
   ```rust
   // Capturer object is expensive, avoiding to create it frequently.
   fn create_capturer(...) -> ResultType<Box<dyn TraitCapturer>> {
       // 采集器创建逻辑
   }
   ```

2. **使用平台特定 API**
   - Windows: DXGI 优先于 GDI
   - Linux: X11 优先于 Wayland
   - macOS: ScreenCapture API

3. **帧率自适应**
   - 根据网络状况动态调整
   - 避免不必要的帧采集
   - 降低 CPU 占用

#### 6.1.2 编码优化

1. **硬件加速**
   - 优先使用 GPU 编码
   - 减少 CPU 占用
   - 提高编码速度

2. **关键帧间隔优化**
   - 录制模式：240 帧
   - 正常模式：编码器默认
   - 减少关键帧数量降低码率

3. **编码失败重试**
   - 最多重试 10 次
   - 自动切换编码器
   - 避免频繁切换

#### 6.1.3 内存优化

1. **缓冲区复用**
   ```rust
   let mut yuv = Vec::new();
   let mut mid_data = Vec::new();
   // 复用缓冲区避免频繁分配
   ```

2. **帧数据共享**
   - 使用引用传递
   - 避免不必要的拷贝
   - 使用 Arc 共享数据

### 6.2 接收端优化

#### 6.2.1 解码优化

1. **硬件解码**
   - 使用 GPU 解码
   - 减少 CPU 占用
   - 提高解码速度

2. **解码失败处理**
   - 快速重试首帧
   - 跟踪失败次数
   - 自动重置解码器

3. **格式缓存**
   - 缓存解码格式
   - 避免频繁查询
   - 减少格式转换

#### 6.2.2 队列优化

1. **无锁队列**
   - 使用 `ArrayQueue`
   - 减少锁竞争
   - 提高并发性能

2. **队列丢弃策略**
   - 动态调整阈值
   - 避免队列溢出
   - 确保新帧进入

3. **批量处理**
   - 批量解码帧
   - 减少上下文切换
   - 提高吞吐量

#### 6.2.3 渲染优化

1. **硬件加速渲染**
   - 使用 GPU 纹理
   - 避免数据拷贝
   - 提高渲染速度

2. **帧率自适应**
   - 根据解码帧率调整
   - 避免过度渲染
   - 降低 GPU 占用

3. **渲染回调优化**
   - 异步渲染
   - 避免阻塞解码线程
   - 提高响应速度

### 6.3 网络优化

#### 6.3.1 拥塞控制

1. **自适应码率**
   - 根据网络延迟调整
   - 避免网络拥塞
   - 保证流畅度

2. **帧率控制**
   - 动态调整帧率
   - 降低带宽占用
   - 提高响应速度

#### 6.3.2 数据包优化

1. **数据包封装**
   - 合理设置包大小
   - 减少包头开销
   - 提高传输效率

2. **关键帧优先**
   - 优先传输关键帧
   - 确保画面同步
   - 减少延迟

### 6.4 跨平台优化

#### 6.4.1 平台特定优化

1. **Windows**
   - 使用 DXGI 采集
   - 硬件加速编码/解码
   - 优化内存管理

2. **Linux**
   - X11 优先于 Wayland
   - 使用 PipeWire 采集
   - 优化 DRM 支持

3. **macOS**
   - 使用 ScreenCapture API
   - 优化 Metal 渲染
   - 减少内存占用

#### 6.4.2 兼容性优化

1. **编码器回退**
   - 硬件编码失败时回退到软件编码
   - 确保功能可用性
   - 记录失败原因

2. **格式协商**
   - 支持多种编码格式
   - 自动选择最佳格式
   - 处理格式不兼容

---

## 7. 总结

RustDesk 的音视频传输渲染链路是一个复杂而高效的系统，通过以下关键技术实现了高质量的远程桌面体验：

### 7.1 核心技术

1. **跨平台屏幕采集**：支持 Windows/Linux/macOS，使用平台特定 API 优化性能
2. **自适应编解码**：根据网络状况和硬件能力动态选择最佳编码格式
3. **智能码率控制**：基于网络延迟的自适应算法，平衡流畅度和质量
4. **硬件加速**：充分利用 GPU 进行编码、解码和渲染
5. **多显示器支持**：完整的多显示器采集和显示能力
6. **隐私保护**：通过 Magnifier API 实现 UAC 场景的隐私保护

### 7.2 性能优化

1. **发送端优化**：采集器复用、硬件加速、缓冲区复用
2. **接收端优化**：无锁队列、批量处理、硬件渲染
3. **网络优化**：自适应码率、帧率控制、关键帧优先
4. **跨平台优化**：平台特定优化、编码器回退、格式协商

### 7.3 技术难点

1. **自适应码率控制**：准确计算网络延迟、平滑参数调整、多用户场景处理
2. **硬件加速编码**：硬件能力检测、编码器切换、性能优化
3. **视频队列管理**：队列溢出处理、帧同步、性能优化
4. **多显示器支持**：显示器热插拔、多显示器同步、性能优化
5. **隐私模式实现**：进程检测、采集器切换、性能影响控制

### 7.4 架构优势

1. **模块化设计**：采集、编码、解码、渲染各模块独立，易于维护和扩展
2. **跨平台支持**：统一的接口设计，平台特定实现，代码复用率高
3. **性能优化**：充分利用硬件加速，优化算法和数据结构
4. **可扩展性**：支持多种编码格式，易于添加新的编码器
5. **稳定性**：完善的错误处理和重试机制，确保系统稳定运行

通过以上技术实现，RustDesk 提供了流畅、高效、安全的远程桌面体验，满足了不同场景下的使用需求。

---

## 附录

### A. 关键文件索引

| 文件路径 | 功能描述 |
|---------|---------|
| [video_service.rs](file:///home/zy/repo/rustdesk/src/server/video_service.rs) | 视频服务主逻辑，采集和编码 |
| [display_service.rs](file:///home/zy/repo/rustdesk/src/server/display_service.rs) | 显示器管理，多显示器支持 |
| [video_qos.rs](file:///home/zy/repo/rustdesk/src/server/video_qos.rs) | QoS 控制，码率和帧率调整 |
| [client.rs](file:///home/zy/repo/rustdesk/src/client.rs) | 客户端主逻辑，解码和渲染 |
| [io_loop.rs](file:///home/zy/repo/rustdesk/src/client/io_loop.rs) | I/O 循环，网络通信和队列管理 |
| [platform/windows.rs](file:///home/zy/repo/rustdesk/src/platform/windows.rs) | Windows 平台实现 |
| [platform/linux.rs](file:///home/zy/repo/rustdesk/src/platform/linux.rs) | Linux 平台实现 |
| [platform/macos.rs](file:///home/zy/repo/rustdesk/src/platform/macos.rs) | macOS 平台实现 |

### B. 术语表

| 术语 | 说明 |
|-----|------|
| Capturer | 屏幕采集器，负责捕获屏幕内容 |
| Encoder | 编码器，将视频帧压缩为编码数据 |
| Decoder | 解码器，将编码数据解压为视频帧 |
| QoS | Quality of Service，服务质量控制 |
| VRAM | Video RAM，显存编码器 |
| HWRAM | Hardware RAM，硬件内存编码器 |
| DXGI | DirectX Graphics Infrastructure，Windows 图形接口 |
| GDI | Graphics Device Interface，Windows 图形设备接口 |
| RTT | Round-Trip Time，往返时间 |
| FPS | Frames Per Second，每秒帧数 |
| I420 | YUV 4:2:0 色彩空间 |
| I444 | YUV 4:4:4 色彩空间 |

### C. 参考资源

- [scrap 库文档](https://github.com/rustdesk/scrap) - 屏幕采集和编解码库
- [libvpx 文档](https://www.webmproject.org/docs/encoder-parameters/) - VP8/VP9 编码器
- [libaom 文档](https://aomedia.googlesource.org/aom/) - AV1 编码器
- [DirectX 文档](https://docs.microsoft.com/en-us/windows/win32/direct3d12/) - Windows DirectX API

---

**文档版本**：1.0  
**最后更新**：2026-01-07  
**维护者**：RustDesk 开发团队
