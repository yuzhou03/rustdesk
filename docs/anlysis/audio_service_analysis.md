# audio_service.rs 模块详细分析

## 一、功能概述

`audio_service.rs` 是 RustDesk 项目中的音频服务模块，主要负责音频数据的采集、编码和传输。该模块支持跨平台音频处理，针对不同操作系统采用了不同的音频后端实现：

- **Linux/Android**: 使用 PulseAudio 实现
- **Windows/macOS**: 使用 CPAL (Cross-platform Audio Library) 实现

该模块的核心功能包括：
1. 音频输入设备的选择和管理
2. 音频数据的实时采集
3. 使用 Opus 编码器进行音频压缩
4. 音频数据的网络传输
5. 静音门控（Noise Gate）机制
6. 音频重采样和声道转换

## 二、核心模块说明

### 2.1 平台相关模块

#### 2.1.1 Linux/Android 实现 (`pa_impl`)

该模块针对 Linux 和 Android 平台实现音频服务：

**主要特点**：
- 使用 PulseAudio 进行音频采集
- 支持 IPC 通信与 `_pa` 服务交互
- 支持音频数据的 32 字节对齐
- Android 平台通过 FFI 调用原生接口获取音频数据

**关键函数**：
- `run()`: 异步运行音频服务的主循环
- `align_to_32()`: 音频数据 32 字节对齐处理

#### 2.1.2 跨平台实现 (`cpal_impl`)

该模块针对 Windows、macOS 等平台使用 CPAL 库实现音频服务：

**主要特点**：
- 使用 CPAL 库进行跨平台音频采集
- 支持多种音频格式（I8, I16, I32, I64, U8, U16, U32, U64, F32, F64）
- 支持 ScreenCaptureKit（macOS 特有）
- 支持音频回环录制（Loopback Recording）

**关键数据结构**：
- `State`: 音频流状态管理
- `INPUT_BUFFER`: 音频输入缓冲区（全局静态变量）

**关键函数**：
- `run()`: 运行音频服务
- `play()`: 创建并启动音频输入流
- `build_input_stream()`: 构建音频输入流
- `get_device()`: 获取音频输入设备
- `get_audio_input()`: 获取指定的音频输入设备
- `send()`: 发送音频数据

### 2.2 全局静态变量

```rust
pub const NAME: &'static str = "audio";
pub const AUDIO_DATA_SIZE_U8: usize = 960 * 4; // 10ms in 48000 stereo
static RESTARTING: AtomicBool = AtomicBool::new(false);
static ref VOICE_CALL_INPUT_DEVICE: Arc::<Mutex::<Option<String>>> = Default::default();
static ref INPUT_BUFFER: Arc<Mutex<std::collections::VecDeque<f32>>> = Default::default();
```

**说明**：
- `NAME`: 服务名称标识
- `AUDIO_DATA_SIZE_U8`: 音频数据大小（10ms，48kHz，立体声）
- `RESTARTING`: 重启标志，防止重复重启
- `VOICE_CALL_INPUT_DEVICE`: 语音通话输入设备配置
- `INPUT_BUFFER`: 音频输入缓冲区（cpal_impl 专用）

## 三、主要函数及参数说明

### 3.1 服务创建与管理

#### 3.1.1 `new()` - 创建音频服务

**函数签名**：
```rust
#[cfg(not(any(target_os = "linux", target_os = "android")))]
pub fn new() -> GenericService

#[cfg(any(target_os = "linux", target_os = "android"))]
pub fn new() -> GenericService
```

**功能**：创建音频服务实例

**返回值**：`GenericService` - 通用服务实例

**平台差异**：
- 非 Linux/Android：使用 `repeat` 模式，每 33ms 执行一次 `cpal_impl::run`
- Linux/Android：使用 `run` 模式，执行 `pa_impl::run`

#### 3.1.2 `restart()` - 重启音频服务

**函数签名**：
```rust
pub fn restart()
```

**功能**：触发音频服务重启

**实现逻辑**：
1. 检查是否已在重启中（避免重复重启）
2. 设置 `RESTARTING` 标志为 `true`
3. 音频服务主循环检测到标志后停止当前流并重新初始化

### 3.2 设备管理

#### 3.2.1 `get_voice_call_input_device()` - 获取语音通话输入设备

**函数签名**：
```rust
#[inline]
pub fn get_voice_call_input_device() -> Option<String>
```

**功能**：获取当前配置的语音通话输入设备

**返回值**：`Option<String>` - 设备名称，未配置时返回 `None`

#### 3.2.2 `set_voice_call_input_device()` - 设置语音通话输入设备

**函数签名**：
```rust
#[inline]
pub fn set_voice_call_input_device(device: Option<String>, set_if_present: bool)
```

**参数**：
- `device`: 设备名称（`None` 表示清除配置）
- `set_if_present`: 是否仅在设备存在时设置（`false` 表示强制设置）

**功能**：设置语音通话输入设备并触发服务重启

**实现逻辑**：
1. 如果 `set_if_present` 为 `false` 且已有设备配置，则直接返回
2. 如果新设备与当前设备相同，则直接返回
3. 更新设备配置并调用 `restart()`

#### 3.2.3 `get_audio_input()` - 获取音频输入设备

**函数签名**：
```rust
pub fn get_audio_input() -> String
```

**功能**：获取当前使用的音频输入设备名称

**返回值**：`String` - 设备名称，优先使用语音通话设备，否则使用配置文件中的设备

### 3.3 音频数据处理

#### 3.3.1 `send_f32()` - 发送音频数据

**函数签名**：
```rust
fn send_f32(data: &[f32], encoder: &mut Encoder, sp: &GenericService)
```

**参数**：
- `data`: 音频数据（f32 格式）
- `encoder`: Opus 编码器引用
- `sp`: 通用服务引用

**功能**：将音频数据编码并发送

**实现逻辑**：
1. **静音门控**：
   - 检测音频数据是否全为零（静音）
   - 如果连续静音超过 `MAX_AUDIO_ZERO_COUNT`（800 帧），则停止发送
   - 静音门控攻击时间约为 3-8 秒（取决于平台）
2. **编码与发送**：
   - Android 平台：将数据分批编码（每批 960 样本）
   - 其他平台：直接编码全部数据
   - 创建 `AudioFrame` 消息并通过服务发送

**错误处理**：编码失败时静默忽略

#### 3.3.2 `create_format_msg()` - 创建格式消息

**函数签名**：
```rust
fn create_format_msg(sample_rate: u32, channels: u16) -> Message
```

**参数**：
- `sample_rate`: 采样率（Hz）
- `channels`: 声道数

**返回值**：`Message` - 包含音频格式信息的消息

**功能**：创建包含音频采样率和声道数的消息，用于通知客户端音频格式

### 3.4 CPAL 实现特定函数

#### 3.4.1 `run()` - 运行音频服务（CPAL）

**函数签名**：
```rust
pub fn run(sp: EmptyExtraFieldService, state: &mut State) -> ResultType<()>
```

**参数**：
- `sp`: 空字段服务实例
- `state`: 音频流状态

**返回值**：`ResultType<()>` - 操作结果

**功能**：根据重启标志执行不同的运行逻辑

**实现逻辑**：
- 如果未在重启中：执行 `run_serv_snapshot()`（快照模式）
- 如果正在重启：执行 `run_restart()`（重启模式）

#### 3.4.2 `play()` - 创建并启动音频流

**函数签名**：
```rust
fn play(sp: &GenericService) -> ResultType<(Box<dyn StreamTrait>, Arc<Message>)>
```

**参数**：
- `sp`: 通用服务引用

**返回值**：`ResultType<(Box<dyn StreamTrait>, Arc<Message>)>` - 音频流和格式消息

**功能**：创建音频输入流并启动

**实现逻辑**：
1. 获取音频设备和配置
2. 确定目标采样率（8000/12000/16000/24000/48000 Hz）
3. 确定声道数（单声道或立体声）
4. 根据音频格式构建输入流
5. 启动音频流
6. 返回流和格式消息

**采样率选择逻辑**：
```rust
let sample_rate = if sample_rate_0 < 12000 {
    8000
} else if sample_rate_0 < 16000 {
    12000
} else if sample_rate_0 < 24000 {
    16000
} else if sample_rate_0 < 48000 {
    24000
} else {
    48000
};
```

#### 3.4.3 `build_input_stream()` - 构建音频输入流

**函数签名**：
```rust
fn build_input_stream<T>(
    device: cpal::Device,
    config: &cpal::SupportedStreamConfig,
    sp: GenericService,
    sample_rate: u32,
    encode_channel: magnum_opus::Channels,
) -> ResultType<cpal::Stream>
where
    T: cpal::SizedSample + dasp::sample::ToSample<f32>
```

**泛型参数**：
- `T`: 音频样本类型（i8, i16, i32, i64, u8, u16, u32, u64, f32, f64）

**参数**：
- `device`: 音频设备
- `config`: 支持的流配置
- `sp`: 通用服务实例
- `sample_rate`: 目标采样率
- `encode_channel`: 编码声道数

**返回值**：`ResultType<cpal::Stream>` - 音频流

**功能**：构建并配置音频输入流

**实现逻辑**：
1. 清空输入缓冲区
2. 创建 Opus 编码器
3. 计算帧大小（10ms）
4. 配置流参数（声道数、采样率、缓冲区大小）
5. 构建输入流并设置回调
6. 回调函数中：
   - 将音频数据转换为 f32 格式
   - 存入缓冲区
   - 当缓冲区数据足够时，调用 `send()` 进行编码和发送

**关键参数**：
- `frame_size = sample_rate_0 as usize / 100`：10ms 的帧大小
- `encode_len = frame_size * encode_channel as usize`：编码长度
- `rechannel_len = encode_len * device_channel as usize / encode_channel as usize`：重声道长度

#### 3.4.4 `send()` - 发送音频数据（CPAL）

**函数签名**：
```rust
fn send(
    data: Vec<f32>,
    sample_rate0: u32,
    sample_rate: u32,
    device_channel: u16,
    encode_channel: u16,
    encoder: &mut Encoder,
    sp: &GenericService,
)
```

**参数**：
- `data`: 音频数据
- `sample_rate0`: 原始采样率
- `sample_rate`: 目标采样率
- `device_channel`: 设备声道数
- `encode_channel`: 编码声道数
- `encoder`: Opus 编码器
- `sp`: 通用服务

**功能**：处理音频数据（重采样、重声道）并发送

**实现逻辑**：
1. 如果采样率不同，进行重采样
2. 如果声道数不同，进行重声道
3. 调用 `send_f32()` 进行编码和发送

#### 3.4.5 `get_device()` - 获取音频设备

**函数签名**：
```rust
fn get_device() -> ResultType<(Device, SupportedStreamConfig)>
```

**返回值**：`ResultType<(Device, SupportedStreamConfig)>` - 音频设备和配置

**功能**：根据平台和配置获取合适的音频输入设备

**平台差异**：

**Windows**：
- 如果配置了输入设备，使用配置的设备
- 否则使用默认输出设备（支持回环录制）

**macOS（ScreenCaptureKit）**：
- 如果配置了输入设备，使用配置的设备
- 如果 ScreenCaptureKit 可用，使用其默认输入设备
- 否则使用默认输入设备

**其他平台**：
- 使用配置的输入设备或默认输入设备

#### 3.4.6 `get_audio_input()` - 获取指定音频输入设备

**函数签名**：
```rust
fn get_audio_input(audio_input: &str) -> ResultType<(Device, SupportedStreamConfig)>
```

**参数**：
- `audio_input`: 设备名称

**返回值**：`ResultType<(Device, SupportedStreamConfig)>` - 音频设备和配置

**功能**：根据设备名称获取音频设备

**实现逻辑**：
1. 如果启用了 ScreenCaptureKit 且设备名称不为空，在 ScreenCaptureKit 主机中搜索设备
2. 如果未找到且设备名称不为空，在默认主机中搜索设备
3. 如果未找到或设备名称为空，使用默认输入设备
4. 获取设备的默认配置并返回

### 3.5 PulseAudio 实现特定函数

#### 3.5.1 `run()` - 运行音频服务（PulseAudio）

**函数签名**：
```rust
#[tokio::main(flavor = "current_thread")]
pub async fn run(sp: EmptyExtraFieldService) -> ResultType<()>
```

**参数**：
- `sp`: 空字段服务实例

**返回值**：`ResultType<()>` - 操作结果

**功能**：异步运行音频服务主循环

**实现逻辑**：
1. 等待 100ms 以等待 `_pa` IPC 服务启动
2. 重置重启标志
3. 连接到 `_pa` IPC 服务（仅 Linux）
4. 创建 Opus 编码器
5. 发送音频输入配置（仅 Linux）
6. 进入主循环：
   - 发送格式消息
   - 接收音频数据（Linux）或通过 FFI 获取（Android）
   - 对齐音频数据到 32 字节
   - 转换为 f32 格式
   - 调用 `send_f32()` 编码和发送

**错误处理**：
- 如果接收到空数据，发送零音频帧
- 如果数据长度不正确，跳过处理

#### 3.5.2 `align_to_32()` - 32 字节对齐

**函数签名**：
```rust
unsafe fn align_to_32(data: Vec<u8>) -> Vec<u8>
```

**参数**：
- `data`: 原始音频数据

**返回值**：`Vec<u8>` - 对齐后的音频数据

**功能**：将音频数据对齐到 32 字节边界

**实现逻辑**：
1. 检查数据指针是否已对齐
2. 如果已对齐，直接返回
3. 否则，创建对齐的缓冲区并复制数据

**安全性**：使用 `unsafe` 代码，需要确保 `hbb_common::mem::aligned_u8_vec` 的约束条件

### 3.6 辅助函数

#### 3.6.1 `is_screen_capture_kit_available()` - 检查 ScreenCaptureKit 可用性

**函数签名**：
```rust
#[cfg(feature = "screencapturekit")]
pub fn is_screen_capture_kit_available() -> bool
```

**返回值**：`bool` - ScreenCaptureKit 是否可用

**功能**：检查系统是否支持 ScreenCaptureKit（macOS 特有）

## 四、返回值类型

### 4.1 主要返回值类型

1. **`GenericService`**: 通用服务类型，用于服务管理和消息传递
2. **`ResultType<()>`**: 操作结果类型，用于错误处理
3. **`Option<String>`**: 可选字符串，用于设备名称
4. **`Message`**: 消息类型，用于网络通信
5. **`Box<dyn StreamTrait>`**: 音频流类型（CPAL）
6. **`Arc<Message>`**: 共享消息引用
7. **`cpal::Stream`**: CPAL 音频流

### 4.2 错误类型

使用 `anyhow::Result` 和 `hbb_common::ResultType` 进行错误处理，支持上下文信息：

```rust
.with_context(|| "Failed to get default input device for loopback")
```

## 五、错误处理机制

### 5.1 错误处理策略

1. **使用 `ResultType`**：大多数函数返回 `ResultType<T>`，允许调用者处理错误
2. **上下文信息**：使用 `.with_context()` 添加错误上下文信息
3. **静默忽略**：某些非关键错误（如编码失败）被静默忽略
4. **日志记录**：使用 `log::trace!()`、`log::debug!()`、`log::info!()` 记录不同级别的日志

### 5.2 关键错误处理点

1. **设备获取失败**：
   ```rust
   .with_context(|| "Failed to get default input device for loopback")
   ```

2. **配置获取失败**：
   ```rust
   .with_context(|| "Failed to get default output format")
   ```

3. **流构建失败**：
   ```rust
   bail!("unsupported audio format: {:?}", f)
   ```

4. **编码失败**：
   ```rust
   Err(_) => {}
   ```

5. **流错误回调**：
   ```rust
   let err_fn = move |err| {
       log::trace!("an error occurred on stream: {}", err);
   };
   ```

### 5.3 重启机制

使用 `RESTARTING` 原子布尔值防止重复重启：

```rust
pub fn restart() {
    if RESTARTING.load(Ordering::SeqCst) {
        return;
    }
    RESTARTING.store(true, Ordering::SeqCst);
}
```

## 六、关键算法和逻辑流程

### 6.1 静音门控（Noise Gate）

**目的**：减少静音时的数据传输，节省带宽

**实现**：
```rust
const MAX_AUDIO_ZERO_COUNT: u16 = 800;
static mut AUDIO_ZERO_COUNT: u16 = 0;

fn send_f32(data: &[f32], encoder: &mut Encoder, sp: &GenericService) {
    if data.iter().filter(|x| **x != 0.).next().is_some() {
        unsafe {
            AUDIO_ZERO_COUNT = 0;
        }
    } else {
        unsafe {
            if AUDIO_ZERO_COUNT > MAX_AUDIO_ZERO_COUNT {
                if AUDIO_ZERO_COUNT == MAX_AUDIO_ZERO_COUNT + 1 {
                    log::debug!("Audio Zero Gate Attack");
                    AUDIO_ZERO_COUNT += 1;
                }
                return;
            }
            AUDIO_ZERO_COUNT += 1;
        }
    }
    // 编码和发送...
}
```

**工作原理**：
1. 检测音频数据是否全为零
2. 如果有非零数据，重置计数器
3. 如果连续静音超过 800 帧（约 3-8 秒），停止发送
4. 首次触发时记录日志

### 6.2 音频重采样

**目的**：将不同采样率的音频转换为目标采样率

**实现**：
```rust
if sample_rate0 != sample_rate {
    data = crate::common::audio_resample(&data, sample_rate0, sample_rate, device_channel);
}
```

**调用外部函数**：`crate::common::audio_resample()`

### 6.3 音频重声道

**目的**：将不同声道数的音频转换为目标声道数

**实现**：
```rust
if device_channel != encode_channel {
    data = crate::common::audio_rechannel(
        data,
        sample_rate,
        sample_rate,
        device_channel,
        encode_channel,
    )
}
```

**调用外部函数**：`crate::common::audio_rechannel()`

### 6.4 Opus 编码

**编码器初始化**：
```rust
let mut encoder = Encoder::new(sample_rate, encode_channel, LowDelay)?;
```

**编码参数**：
- 应用类型：`LowDelay`（低延迟）
- 声道数：`Mono` 或 `Stereo`
- 采样率：8000/12000/16000/24000/48000 Hz

**编码调用**：
```rust
match encoder.encode_vec_float(data, data.len() * 6) {
    Ok(data) => {
        let mut msg_out = Message::new();
        msg_out.set_audio_frame(AudioFrame {
            data: data.into(),
            ..Default::default()
        });
        sp.send(msg_out);
    }
    Err(_) => {}
}
```

**Android 平台特殊处理**：
```rust
const BATCH_SIZE: usize = 960;
let input_size = data.len();
if input_size > BATCH_SIZE && input_size % BATCH_SIZE == 0 {
    let n = input_size / BATCH_SIZE;
    for i in 0..n {
        match encoder.encode_vec_float(&data[i * BATCH_SIZE..(i + 1) * BATCH_SIZE], BATCH_SIZE) {
            Ok(data) => {
                let mut msg_out = Message::new();
                msg_out.set_audio_frame(AudioFrame {
                    data: data.into(),
                    ..Default::default()
                });
                sp.send(msg_out);
            }
            Err(_) => {}
        }
    }
}
```

### 6.5 音频流回调处理（CPAL）

**回调函数**：
```rust
move |data: &[T], _: &InputCallbackInfo| {
    let buffer: Vec<f32> = data.iter().map(|s| T::to_sample(*s)).collect();
    let mut lock = INPUT_BUFFER.lock().unwrap();
    lock.extend(buffer);
    while lock.len() >= rechannel_len {
        let frame: Vec<f32> = lock.drain(0..rechannel_len).collect();
        send(
            frame,
            sample_rate_0,
            sample_rate,
            device_channel,
            encode_channel as _,
            &mut encoder,
            &sp,
        );
    }
}
```

**处理流程**：
1. 将音频数据转换为 f32 格式
2. 存入输入缓冲区
3. 当缓冲区数据足够时，提取一帧数据
4. 调用 `send()` 进行重采样、重声道、编码和发送

### 6.6 服务重启流程

**触发重启**：
```rust
pub fn restart() {
    log::info!("restart the audio service, freezing now...");
    if RESTARTING.load(Ordering::SeqCst) {
        return;
    }
    RESTARTING.store(true, Ordering::SeqCst);
}
```

**检测重启（CPAL）**：
```rust
pub fn run(sp: EmptyExtraFieldService, state: &mut State) -> ResultType<()> {
    if !RESTARTING.load(Ordering::SeqCst) {
        run_serv_snapshot(sp, state)
    } else {
        run_restart(sp, state)
    }
}
```

**执行重启**：
```rust
fn run_restart(sp: EmptyExtraFieldService, state: &mut State) -> ResultType<()> {
    state.reset();
    sp.snapshot(|_sps: ServiceSwap<_>| Ok(()))?;
    match &state.stream {
        None => {
            state.stream = Some(play(&sp)?);
        }
        _ => {}
    }
    if let Some((_, format)) = &state.stream {
        sp.send_shared(format.clone());
    }
    RESTARTING.store(false, Ordering::SeqCst);
    Ok(())
}
```

## 七、与其他模块的交互关系

### 7.1 依赖的外部模块

#### 7.1.1 `hbb_common`
- **用途**：通用工具库
- **使用**：
  - `anyhow::anyhow`: 错误处理
  - `mem::aligned_u8_vec`: 内存对齐
  - `sleep()`: 异步睡眠
  - `ResultType`: 结果类型

#### 7.1.2 `magnum_opus`
- **用途**：Opus 音频编码器
- **使用**：
  - `Encoder`: Opus 编码器
  - `Application::LowDelay`: 低延迟应用类型
  - `Channels::Mono/Stereo`: 声道配置

#### 7.1.3 `cpal`
- **用途**：跨平台音频库
- **使用**：
  - `Host`: 音频主机
  - `Device`: 音频设备
  - `Stream`: 音频流
  - `StreamConfig`: 流配置
  - `SupportedStreamConfig`: 支持的流配置
  - `SampleFormat`: 音频格式
  - `BufferSize`: 缓冲区大小

#### 7.1.4 `dasp`
- **用途**：数字音频信号处理
- **使用**：
  - `sample::ToSample<f32>`: 样本类型转换

#### 7.1.5 `lazy_static`
- **用途**：静态变量延迟初始化
- **使用**：
  - 全局静态变量的定义

#### 7.1.6 `tokio`
- **用途**：异步运行时
- **使用**：
  - `#[tokio::main]`: 异步主函数
  - `sleep()`: 异步睡眠

### 7.2 内部模块交互

#### 7.2.1 `crate::ipc`
- **用途**：进程间通信
- **使用**：
  - `connect()`: 连接到 IPC 服务
  - `Data::Config`: 配置数据
  - `next_raw()`: 接收原始数据

#### 7.2.2 `crate::platform`
- **用途**：平台相关配置
- **使用**：
  - `PA_SAMPLE_RATE`: PulseAudio 采样率

#### 7.2.3 `crate::common`
- **用途**：通用音频处理
- **使用**：
  - `audio_resample()`: 音频重采样
  - `audio_rechannel()`: 音频重声道

#### 7.2.4 `crate::scrap::android::ffi`
- **用途**：Android FFI 接口
- **使用**：
  - `get_audio_raw()`: 获取原始音频数据

#### 7.2.5 `super::*`
- **用途**：父模块导入
- **使用**：
  - `EmptyExtraFieldService`: 空字段服务
  - `GenericService`: 通用服务
  - `Message`: 消息类型
  - `AudioFormat`: 音频格式
  - `AudioFrame`: 音频帧
  - `Misc`: 杂项消息
  - `service::Reset`: 服务重置 trait
  - `service::ServiceSwap`: 服务交换类型
  - `Config`: 配置管理

### 7.3 消息流程

```
音频设备 → 音频采集 → 格式转换 → 重采样/重声道 → Opus 编码 → AudioFrame → Message → 网络传输
```

### 7.4 服务生命周期

```
new() → run() → play()/build_input_stream() → 音频回调 → send_f32() → 编码发送 → restart() → 重新初始化
```

## 八、平台差异总结

### 8.1 Linux/Android
- **音频后端**：PulseAudio
- **通信方式**：IPC（`_pa` 服务）
- **数据对齐**：32 字节对齐
- **Android 特有**：FFI 调用原生接口

### 8.2 Windows
- **音频后端**：CPAL（WASAPI）
- **回环录制**：使用默认输出设备
- **支持格式**：I8, I16, I32, I64, U8, U16, U32, U64, F32, F64

### 8.3 macOS
- **音频后端**：CPAL（CoreAudio）
- **ScreenCaptureKit**：可选支持（需要编译特性）
- **回环录制**：需要第三方驱动（如 BlackHole）

### 8.4 其他平台
- **音频后端**：CPAL
- **默认行为**：使用默认输入设备

## 九、性能优化

### 9.1 静音门控
- 减少静音时的数据传输
- 节省网络带宽

### 9.2 批量编码（Android）
- 将大数据分批编码
- 减少单次编码压力

### 9.3 缓冲区管理
- 使用 `VecDeque` 作为输入缓冲区
- 高效的数据插入和删除

### 9.4 低延迟编码
- 使用 `LowDelay` 应用类型
- 10ms 帧大小

## 十、使用示例

### 10.1 创建音频服务

```rust
let audio_service = audio_service::new();
```

### 10.2 设置语音通话输入设备

```rust
audio_service::set_voice_call_input_device(Some("Microphone (Realtek)".to_string()), true);
```

### 10.3 重启音频服务

```rust
audio_service::restart();
```

### 10.4 获取当前设备

```rust
let device = audio_service::get_voice_call_input_device();
```

## 十一、注意事项

1. **线程安全**：使用 `Mutex` 和 `AtomicBool` 保证线程安全
2. **内存安全**：部分 `unsafe` 代码需要确保约束条件
3. **平台兼容性**：不同平台使用不同的音频后端
4. **错误处理**：某些错误被静默忽略，需要关注日志
5. **性能考虑**：音频处理在回调中执行，需要快速处理
6. **重启机制**：避免频繁重启，使用标志防止重复重启
7. **采样率限制**：Opus 编码器仅支持特定采样率（8000/12000/16000/24000/48000 Hz）

## 十二、总结

`audio_service.rs` 是一个功能完善的音频服务模块，具有以下特点：

1. **跨平台支持**：针对不同操作系统使用不同的音频后端
2. **高效编码**：使用 Opus 低延迟编码器
3. **智能优化**：静音门控、批量编码等优化策略
4. **灵活配置**：支持多种音频输入设备和格式
5. **健壮性**：完善的错误处理和重启机制
6. **可扩展性**：模块化设计，易于扩展新功能

该模块是 RustDesk 音频功能的核心组件，为远程桌面音频传输提供了可靠的基础。
