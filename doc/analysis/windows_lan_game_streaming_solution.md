# Windows 11平台局域网游戏串流系统技术方案

## 1. 系统概述

本技术方案旨在实现一个基于Windows 11平台的局域网游戏串流系统，支持通过IP直连进行游戏画面的实时传输。系统充分利用Windows 11的新特性，包括DirectX 12 Ultimate、硬件调度器优化、Auto HDR等，提供更低延迟和更高质量的游戏体验。

### 1.1 Windows 11平台优势

- **DirectX 12 Ultimate**: 支持光线追踪、网格着色器、采样器反馈等高级特性
- **硬件调度器**: 更好的CPU/GPU协作，降低延迟
- **Auto HDR**: 自动为SDR游戏添加HDR效果
- **DirectStorage**: 极速加载游戏资源
- **更好的网络栈**: 优化的TCP/IP协议栈，降低网络延迟
- **内存管理改进**: 更高效的内存分配和回收
- **多显示器优化**: 更好的多显示器支持和窗口管理

### 1.2 系统架构

```
┌─────────────────────────────────────────────────────────────────┐
│                      发送端 (Sender) - Windows 11               │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 屏幕采集模块  │──│ 编码器模块    │──│ 网络传输模块  │           │
│  │ (DXGI 1.6)   │  │ (AV1/VP9/H264)│  │ (TCP/UDP)   │           │
│  │ DirectX 12   │  │ 硬件加速      │  │ Windows 11  │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│         │                 │                 │                   │
│         └─────────────────┴─────────────────┘                   │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │ 控制模块     │                              │
│                    │ (QoS管理)    │                              │
│                    │ 硬件调度器   │                              │
│                    └─────────────┘                              │
└───────────────────────────┬─────────────────────────────────────┘
                            │
                    局域网IP直连
                            │
┌───────────────────────────▼─────────────────────────────────────┐
│                    接收端 (Receiver) - Windows 11               │
├─────────────────────────────────────────────────────────────────┤
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐           │
│  │ 网络接收模块  │──│ 解码器模块    │──│ 渲染模块      │           │
│  │ (TCP/UDP)    │  │ (AV1/VP9/H264)│  │ DirectX 12  │           │
│  │ Windows 11   │  │ 硬件加速      │  │ Auto HDR    │           │
│  └──────────────┘  └──────────────┘  └──────────────┘           │
│         │                 │                 │                   │
│         └─────────────────┴─────────────────┘                   │
│                           │                                     │
│                    ┌──────▼──────┐                              │
│                    │ 控制模块     │                              │
│                    │ (输入转发)   │                              │
│                    │ 低延迟优化   │                              │
│                    └─────────────┘                              │
└─────────────────────────────────────────────────────────────────┘
```

## 2. 核心模块设计

### 2.1 屏幕采集模块 (Screen Capture)

#### 2.1.1 技术选型
- **DXGI Desktop Duplication API 1.6**: Windows 11原生支持，提供高性能屏幕捕获
- **DirectX 12**: 利用Windows 11的硬件调度器，降低延迟
- **优势**: 
  - 硬件加速，低CPU占用
  - 支持多显示器和HDR
  - 支持变化区域检测
  - 与DirectX 12无缝集成
  - 支持可变刷新率（VRR）

#### 2.1.2 实现逻辑

```rust
use winapi::um::dxgi1_6::*;
use winapi::um::d3d12::*;
use winapi::shared::dxgiformat::*;

pub struct Capturer {
    device: *mut ID3D12Device,
    command_queue: *mut ID3D12CommandQueue,
    duplication: *mut IDXGIOutputDuplication,
    display: Display,
    frame: Frame,
    allocator: *mut ID3D12CommandAllocator,
    command_list: *mut ID3D12GraphicsCommandList,
}

impl Capturer {
    pub fn new(display: Display) -> io::Result<Capturer> {
        unsafe {
            let mut device = ptr::null_mut();
            D3D12CreateDevice(
                ptr::null_mut(),
                D3D_FEATURE_LEVEL_12_0,
                &IID_ID3D12Device,
                &mut device as *mut _ as *mut _,
            )?;
            
            let queue_desc = D3D12_COMMAND_QUEUE_DESC {
                Type: D3D12_COMMAND_LIST_TYPE_COPY,
                Flags: D3D12_COMMAND_QUEUE_FLAG_NONE,
                Priority: 0,
                NodeMask: 0,
            };
            
            let mut command_queue = ptr::null_mut();
            (*device).CreateCommandQueue(
                &queue_desc,
                &IID_ID3D12CommandQueue,
                &mut command_queue as *mut _ as *mut _,
            )?;
            
            let mut allocator = ptr::null_mut();
            (*device).CreateCommandAllocator(
                D3D12_COMMAND_LIST_TYPE_COPY,
                &IID_ID3D12CommandAllocator,
                &mut allocator as *mut _ as *mut _,
            )?;
            
            let mut command_list = ptr::null_mut();
            (*device).CreateCommandList(
                0,
                D3D12_COMMAND_LIST_TYPE_COPY,
                allocator,
                ptr::null(),
                &IID_ID3D12GraphicsCommandList,
                &mut command_list as *mut _ as *mut _,
            )?;
            
            let mut duplication = ptr::null_mut();
            (*display.inner.0).DuplicateOutput(
                command_queue as *mut _,
                &mut duplication,
            )?;
            
            Ok(Capturer {
                device,
                command_queue,
                duplication,
                display,
                frame: Frame::new(display.width, display.height),
                allocator,
                command_list,
            })
        }
    }
    
    pub fn capture(&mut self) -> io::Result<&Frame> {
        unsafe {
            let mut resource = ptr::null_mut();
            let mut frame_info = mem::zeroed();
            
            let result = (*self.duplication).AcquireNextFrame(
                16,
                &mut frame_info,
                &mut resource,
            );
            
            if result == DXGI_ERROR_WAIT_TIMEOUT {
                return Err(io::Error::new(
                    io::ErrorKind::WouldBlock,
                    "Timeout waiting for frame",
                ));
            }
            
            if result != 0 {
                return Err(io::Error::new(
                    io::ErrorKind::Other,
                    format!("AcquireNextFrame failed: 0x{:X}", result),
                ));
            }
            
            self.frame.copy_from_resource(
                self.device,
                self.command_list,
                resource,
            );
            
            (*self.duplication).ReleaseFrame();
            
            Ok(&self.frame)
        }
    }
}
```

#### 2.1.3 性能优化
- **变化区域检测**: 只传输变化的屏幕区域
- **帧率自适应**: 根据网络状况动态调整采集帧率
- **多线程处理**: 采集和编码在不同线程中并行执行
- **硬件调度器**: 利用Windows 11的硬件调度器优化CPU/GPU协作
- **低延迟模式**: 使用Windows 11的游戏模式优化

### 2.2 编码器模块 (Encoder)

#### 2.2.1 编码器选择
支持多种编码器，根据硬件能力自动选择：

1. **AV1** (优先级最高)
   - 最新一代编码标准
   - 压缩效率最高
   - 需要硬件支持（NVIDIA RTX 40系列、AMD RX 7000系列）
   - Windows 11原生支持AV1硬件解码

2. **VP9**
   - 开源免费
   - 压缩效率优于H.264
   - 硬件支持广泛
   - Windows 11优化了VP9解码性能

3. **H.264**
   - 硬件支持最广泛
   - 编解码速度快
   - 兼容性最佳
   - Windows 11优化了H.264编解码性能

#### 2.2.2 硬件加速编码器实现

```rust
use windows::Win32::Graphics::Direct3D12::*;
use windows::Win32::Graphics::Dxgi::Common::*;

pub struct HardwareEncoder {
    device: *mut ID3D12Device,
    encoder: *mut ID3D12VideoEncoder,
    heap: *mut ID3D12Heap,
    codec: CodecType,
}

impl HardwareEncoder {
    pub fn new(device: *mut ID3D12Device, codec: CodecType) -> ResultType<Self> {
        unsafe {
            let heap_desc = D3D12_HEAP_DESC {
                SizeInBytes: 1024 * 1024 * 64,
                Properties: D3D12_HEAP_PROPERTIES {
                    Type: D3D12_HEAP_TYPE_DEFAULT,
                    CPUPageProperty: D3D12_CPU_PAGE_PROPERTY_UNKNOWN,
                    MemoryPoolPreference: D3D12_MEMORY_POOL_UNKNOWN,
                    CreationNodeMask: 1,
                    VisibleNodeMask: 1,
                },
                Alignment: 0,
                Flags: D3D12_HEAP_FLAG_ALLOW_ONLY_BUFFERS,
            };
            
            let mut heap = ptr::null_mut();
            (*device).CreateHeap(&heap_desc, &IID_ID3D12Heap, &mut heap as *mut _ as *mut _)?;
            
            let encoder_desc = D3D12_VIDEO_ENCODER_DESC {
                NodeMask: 0,
                Flags: D3D12_VIDEO_ENCODER_FLAG_NONE,
                Codec: match codec {
                    CodecType::AV1 => D3D12_VIDEO_ENCODER_CODEC_AV1,
                    CodecType::VP9 => D3D12_VIDEO_ENCODER_CODEC_VP9,
                    CodecType::H264 => D3D12_VIDEO_ENCODER_CODEC_H264,
                    _ => return Err(anyhow!("Unsupported codec")),
                },
                InputFormat: DXGI_FORMAT_NV12,
            };
            
            let mut encoder = ptr::null_mut();
            (*device).CreateVideoEncoder(&encoder_desc, &IID_ID3D12VideoEncoder, &mut encoder as *mut _ as *mut _)?;
            
            Ok(HardwareEncoder {
                device,
                encoder,
                heap,
                codec,
            })
        }
    }
    
    pub fn encode(&mut self, frame: &Frame) -> ResultType<Vec<u8>> {
        unsafe {
            let resource_desc = D3D12_RESOURCE_DESC {
                Dimension: D3D12_RESOURCE_DIMENSION_TEXTURE2D,
                Alignment: 0,
                Width: frame.width as u64,
                Height: frame.height,
                DepthOrArraySize: 1,
                MipLevels: 1,
                Format: DXGI_FORMAT_NV12,
                SampleDesc: DXGI_SAMPLE_DESC { Count: 1, Quality: 0 },
                Layout: D3D12_TEXTURE_LAYOUT_ROW_MAJOR,
                Flags: D3D12_RESOURCE_FLAG_NONE,
            };
            
            let mut resource = ptr::null_mut();
            (*self.device).CreateCommittedResource(
                &D3D12_HEAP_PROPERTIES {
                    Type: D3D12_HEAP_TYPE_DEFAULT,
                    CPUPageProperty: D3D12_CPU_PAGE_PROPERTY_UNKNOWN,
                    MemoryPoolPreference: D3D12_MEMORY_POOL_UNKNOWN,
                    CreationNodeMask: 1,
                    VisibleNodeMask: 1,
                },
                D3D12_HEAP_FLAG_NONE,
                &resource_desc,
                D3D12_RESOURCE_STATE_COPY_DEST,
                ptr::null(),
                &IID_ID3D12Resource,
                &mut resource as *mut _ as *mut _,
            )?;
            
            let mut output_data = Vec::new();
            
            Ok(output_data)
        }
    }
}
```

#### 2.2.3 编码参数配置

```rust
pub struct EncoderConfig {
    pub width: u32,
    pub height: u32,
    pub bitrate: u32,
    pub keyframe_interval: u32,
    pub codec: CodecType,
    pub hardware_acceleration: bool,
    pub low_latency_mode: bool,
    pub adaptive_bitrate: bool,
}

impl Default for EncoderConfig {
    fn default() -> Self {
        EncoderConfig {
            width: 1920,
            height: 1080,
            bitrate: 8000,
            keyframe_interval: 60,
            codec: CodecType::AV1,
            hardware_acceleration: true,
            low_latency_mode: true,
            adaptive_bitrate: true,
        }
    }
}
```

### 2.3 网络传输模块 (Network Transmission)

#### 2.3.1 协议设计
- **控制通道**: TCP连接，用于信令、配置、状态同步
- **数据通道**: UDP连接，用于视频数据传输（低延迟优先）
- **Windows 11优化**: 利用Windows 11优化的TCP/IP协议栈

#### 2.3.2 数据包格式

```rust
#[repr(C)]
pub struct VideoPacket {
    pub magic: u32,
    pub version: u8,
    pub packet_type: u8,
    pub frame_index: u32,
    pub packet_index: u16,
    pub total_packets: u16,
    pub timestamp: u64,
    pub codec: CodecType,
    pub data: Vec<u8>,
}

#[repr(u8)]
pub enum PacketType {
    KeyFrame = 0x01,
    DeltaFrame = 0x02,
    Config = 0x03,
    Ack = 0x04,
    Nack = 0x05,
    QoSReport = 0x06,
}

#[repr(u8)]
pub enum CodecType {
    AV1 = 0x01,
    VP9 = 0x02,
    H264 = 0x03,
}
```

#### 2.3.3 Windows 11网络优化

```rust
use std::net::UdpSocket;
use windows::Win32::Networking::WinSock::*;

pub struct VideoSender {
    socket: UdpSocket,
    receiver_ip: String,
    receiver_port: u16,
    mtu: usize,
}

impl VideoSender {
    pub fn new(receiver_ip: &str, receiver_port: u16) -> io::Result<Self> {
        let socket = UdpSocket::bind("0.0.0.0:0")?;
        
        unsafe {
            let mut reuse_addr: i32 = 1;
            setsockopt(
                socket.as_raw_socket(),
                SOL_SOCKET,
                SO_REUSEADDR,
                &reuse_addr as *const _ as *const _,
                std::mem::size_of_val(&reuse_addr) as i32,
            );
            
            let mut buffer_size: i32 = 1024 * 1024 * 10;
            setsockopt(
                socket.as_raw_socket(),
                SOL_SOCKET,
                SO_RCVBUF,
                &buffer_size as *const _ as *const _,
                std::mem::size_of_val(&buffer_size) as i32,
            );
            
            setsockopt(
                socket.as_raw_socket(),
                SOL_SOCKET,
                SO_SNDBUF,
                &buffer_size as *const _ as *const _,
                std::mem::size_of_val(&buffer_size) as i32,
            );
        }
        
        Ok(VideoSender {
            socket,
            receiver_ip: receiver_ip.to_string(),
            receiver_port,
            mtu: 1400,
        })
    }
    
    pub fn send_frame(&self, frame: VideoPacket) -> io::Result<()> {
        let data = bincode::serialize(&frame)?;
        
        let chunks = data.chunks(self.mtu);
        let total_chunks = chunks.len();
        
        for (i, chunk) in chunks.enumerate() {
            let mut packet = frame.clone();
            packet.packet_index = i as u16;
            packet.total_packets = total_chunks as u16;
            packet.data = chunk.to_vec();
            
            let packet_data = bincode::serialize(&packet)?;
            self.socket.send_to(
                &packet_data,
                &format!("{}:{}", self.receiver_ip, self.receiver_port),
            )?;
        }
        
        Ok(())
    }
}
```

### 2.4 渲染模块 (Renderer)

#### 2.4.1 技术选型
- **DirectX 12 Ultimate**: Windows 11原生支持，提供最高性能
- **Auto HDR**: 自动为SDR游戏添加HDR效果
- **优势**:
  - 极低延迟渲染
  - 硬件加速
  - 支持HDR和可变刷新率
  - 与Windows 11深度集成
  - 支持光线追踪（未来扩展）

#### 2.4.2 DirectX 12渲染器实现

```rust
use windows::Win32::Graphics::Direct3D12::*;
use windows::Win32::Graphics::Dxgi::Common::*;

pub struct Renderer {
    device: *mut ID3D12Device,
    command_queue: *mut ID3D12CommandQueue,
    swap_chain: *mut IDXGISwapChain3,
    render_target_view_heap: *mut ID3D12DescriptorHeap,
    command_allocator: *mut ID3D12CommandAllocator,
    command_list: *mut ID3D12GraphicsCommandList,
    fence: *mut ID3D12Fence,
    fence_value: u64,
    fence_event: HANDLE,
}

impl Renderer {
    pub fn new(hwnd: HWND, width: u32, height: u32) -> ResultType<Self> {
        unsafe {
            let mut device = ptr::null_mut();
            D3D12CreateDevice(
                ptr::null_mut(),
                D3D_FEATURE_LEVEL_12_0,
                &IID_ID3D12Device,
                &mut device as *mut _ as *mut _,
            )?;
            
            let queue_desc = D3D12_COMMAND_QUEUE_DESC {
                Type: D3D12_COMMAND_LIST_TYPE_DIRECT,
                Flags: D3D12_COMMAND_QUEUE_FLAG_NONE,
                Priority: 0,
                NodeMask: 0,
            };
            
            let mut command_queue = ptr::null_mut();
            (*device).CreateCommandQueue(
                &queue_desc,
                &IID_ID3D12CommandQueue,
                &mut command_queue as *mut _ as *mut _,
            )?;
            
            let mut dxgi_factory = ptr::null_mut();
            CreateDXGIFactory1(&IID_IDXGIFactory4, &mut dxgi_factory as *mut _ as *mut _)?;
            
            let swap_chain_desc = DXGI_SWAP_CHAIN_DESC1 {
                Width: width,
                Height: height,
                Format: DXGI_FORMAT_R8G8B8A8_UNORM,
                Stereo: FALSE.into(),
                SampleDesc: DXGI_SAMPLE_DESC { Count: 1, Quality: 0 },
                BufferUsage: DXGI_USAGE_RENDER_TARGET_OUTPUT,
                BufferCount: 2,
                Scaling: DXGI_SCALING_STRETCH,
                SwapEffect: DXGI_SWAP_EFFECT_FLIP_DISCARD,
                AlphaMode: DXGI_ALPHA_MODE_IGNORE,
                Flags: 0,
            };
            
            let mut swap_chain = ptr::null_mut();
            (*(dxgi_factory as *mut IDXGIFactory4)).CreateSwapChainForHwnd(
                command_queue as *mut _,
                hwnd,
                &swap_chain_desc,
                ptr::null(),
                ptr::null_mut(),
                &mut swap_chain as *mut _ as *mut _,
            )?;
            
            let swap_chain3 = swap_chain as *mut IDXGISwapChain3;
            
            let mut allocator = ptr::null_mut();
            (*device).CreateCommandAllocator(
                D3D12_COMMAND_LIST_TYPE_DIRECT,
                &IID_ID3D12CommandAllocator,
                &mut allocator as *mut _ as *mut _,
            )?;
            
            let mut command_list = ptr::null_mut();
            (*device).CreateCommandList(
                0,
                D3D12_COMMAND_LIST_TYPE_DIRECT,
                allocator,
                ptr::null(),
                &IID_ID3D12GraphicsCommandList,
                &mut command_list as *mut _ as *mut _,
            )?;
            
            let mut fence = ptr::null_mut();
            (*device).CreateFence(0, D3D12_FENCE_FLAG_NONE, &IID_ID3D12Fence, &mut fence as *mut _ as *mut _)?;
            
            let fence_event = CreateEventW(ptr::null_mut(), FALSE.into(), FALSE.into(), None)?;
            
            Ok(Renderer {
                device,
                command_queue,
                swap_chain: swap_chain3,
                render_target_view_heap: ptr::null_mut(),
                command_allocator: allocator,
                command_list,
                fence,
                fence_value: 0,
                fence_event,
            })
        }
    }
    
    pub fn render_frame(&mut self, texture: *mut ID3D12Resource) -> ResultType<()> {
        unsafe {
            let command_allocator = self.command_allocator;
            (*command_allocator).Reset();
            
            let command_list = self.command_list;
            (*command_list).Reset(command_allocator, ptr::null());
            
            let barrier = D3D12_RESOURCE_BARRIER {
                Type: D3D12_RESOURCE_BARRIER_TYPE_TRANSITION,
                Flags: D3D12_RESOURCE_BARRIER_FLAG_NONE,
                Transition: D3D12_RESOURCE_TRANSITION_BARRIER {
                    pResource: texture,
                    Subresource: D3D12_RESOURCE_BARRIER_ALL_SUBRESOURCES,
                    StateBefore: D3D12_RESOURCE_STATE_COPY_SOURCE,
                    StateAfter: D3D12_RESOURCE_STATE_PIXEL_SHADER_RESOURCE,
                },
            };
            
            (*command_list).ResourceBarrier(1, &barrier);
            
            (*command_list).Close();
            
            let command_lists = [command_list as *mut ID3D12CommandList];
            (*self.command_queue).ExecuteCommandLists(1, command_lists.as_ptr());
            
            let fence_value = self.fence_value + 1;
            (*self.command_queue).Signal(self.fence, fence_value);
            
            if (*self.fence).GetCompletedValue() < fence_value {
                (*self.fence).SetEventOnCompletion(fence_value, self.fence_event);
                WaitForSingleObject(self.fence_event, INFINITE);
            }
            
            self.fence_value = fence_value;
            
            (*self.swap_chain).Present(1, 0);
            
            Ok(())
        }
    }
}
```

### 2.5 安全验证模块 (Security)

#### 2.5.1 Windows 11安全特性
- **Windows Hello**: 生物识别认证
- **Device Encryption**: 设备级加密
- **Credential Guard**: 凭据保护

#### 2.5.2 密码验证

```rust
use argon2::{self, Config, ThreadMode, Variant, Version};

pub fn hash_password(password: &str, salt: &[u8]) -> Result<Vec<u8>> {
    let config = Config {
        variant: Variant::Argon2id,
        version: Version::Version13,
        mem_cost: 65536,
        time_cost: 3,
        lanes: 4,
        thread_mode: ThreadMode::Parallel,
        secret: &[],
        ad: &[],
        hash_length: 32,
    };
    
    argon2::hash_encoded(password.as_bytes(), salt, &config)
        .map_err(|e| anyhow!("Password hashing failed: {}", e))
}

pub fn verify_password(hash: &str, password: &str) -> Result<bool> {
    argon2::verify_encoded(hash, password.as_bytes())
        .map_err(|e| anyhow!("Password verification failed: {}", e))
}
```

#### 2.5.3 加密通信

```rust
use aes_gcm::{Aes256Gcm, Key, Nonce};
use aes_gcm::aead::{Aead, NewAead};

pub struct Encryption {
    cipher: Aes256Gcm,
}

impl Encryption {
    pub fn new(key: &[u8; 32]) -> Self {
        let key = Key::from_slice(key);
        let cipher = Aes256Gcm::new(key);
        Encryption { cipher }
    }
    
    pub fn encrypt(&self, nonce: &[u8; 12], plaintext: &[u8]) -> Result<Vec<u8>> {
        let nonce = Nonce::from_slice(nonce);
        self.cipher
            .encrypt(nonce, plaintext)
            .map_err(|e| anyhow!("Encryption failed: {}", e))
    }
    
    pub fn decrypt(&self, nonce: &[u8; 12], ciphertext: &[u8]) -> Result<Vec<u8>> {
        let nonce = Nonce::from_slice(nonce);
        self.cipher
            .decrypt(nonce, ciphertext)
            .map_err(|e| anyhow!("Decryption failed: {}", e))
    }
}
```

## 3. 实现流程

### 3.1 发送端流程

```
1. 初始化
   ├── 创建DirectX 12设备
   ├── 初始化DXGI 1.6屏幕采集
   ├── 初始化硬件加速编码器（AV1/VP9/H264）
   ├── 创建UDP socket（Windows 11优化）
   ├── 启动硬件调度器
   └── 启动QoS监控线程

2. 主循环
   ├── 屏幕采集（60-144 FPS）
   │   ├── 获取屏幕帧（DXGI 1.6）
   │   ├── 检测变化区域
   │   ├── 支持HDR采集
   │   └── 准备编码数据
   │
   ├── 视频编码
   │   ├── 根据QoS调整编码参数
   │   ├── 硬件加速编码
   │   ├── 生成关键帧（定期）
   │   └── 支持AV1编码
   │
   ├── 网络传输
   │   ├── 分包处理（MTU 1400）
   │   ├── 添加时间戳和序号
   │   ├── UDP发送（Windows 11优化）
   │   └── 自适应发送速率
   │
   └── QoS管理
       ├── 监控网络状况
       ├── 动态调整码率
       ├── 硬件调度器优化
       └── 低延迟模式
```

### 3.2 接收端流程

```
1. 初始化
   ├── 创建DirectX 12设备
   ├── 创建渲染窗口
   ├── 初始化硬件加速解码器
   ├── 创建UDP socket（Windows 11优化）
   ├── 连接到发送端
   └── 启用Auto HDR

2. 主循环
   ├── 网络接收
   │   ├── 接收UDP数据包（Windows 11优化）
   │   ├── 重组帧数据
   │   └── 处理丢包重传
   │
   ├── 视频解码
   │   ├── 硬件加速解码
   │   ├── 处理关键帧
   │   ├── 支持AV1解码
   │   └── 输出渲染纹理
   │
   ├── 渲染显示
   │   ├── 更新纹理数据
   │   ├── DirectX 12渲染
   │   ├── Auto HDR处理
   │   ├── 可变刷新率支持
   │   └── 垂直同步优化
   │
   └── 输入转发
       ├── 捕获鼠标键盘事件
       ├── 打包输入数据
       └── 发送到发送端
```

## 4. QoS (Quality of Service) 管理

### 4.1 自适应码率控制

```rust
pub struct VideoQoS {
    target_bitrate: u32,
    current_bitrate: u32,
    min_bitrate: u32,
    max_bitrate: u32,
    frame_rate: u32,
    packet_loss_rate: f32,
    rtt: u32,
    hardware_scheduler_enabled: bool,
}

impl VideoQoS {
    pub fn adjust_bitrate(&mut self, network_stats: &NetworkStats) {
        self.packet_loss_rate = network_stats.packet_loss_rate;
        self.rtt = network_stats.rtt;
        
        if self.packet_loss_rate > 0.05 {
            self.current_bitrate = (self.current_bitrate * 80) / 100;
        } else if self.packet_loss_rate < 0.01 && self.rtt < 50 {
            self.current_bitrate = (self.current_bitrate * 110) / 100;
        }
        
        self.current_bitrate = self.current_bitrate
            .max(self.min_bitrate)
            .min(self.max_bitrate);
        
        if self.hardware_scheduler_enabled {
            self.optimize_for_hardware_scheduler();
        }
    }
    
    fn optimize_for_hardware_scheduler(&mut self) {
        if self.rtt < 30 {
            self.frame_rate = 144;
        } else if self.rtt < 50 {
            self.frame_rate = 120;
        } else if self.rtt < 100 {
            self.frame_rate = 90;
        } else {
            self.frame_rate = 60;
        }
    }
}
```

### 4.2 网络状况监控

```rust
pub struct NetworkStats {
    pub packet_loss_rate: f32,
    pub rtt: u32,
    pub bandwidth: u32,
    pub jitter: u32,
}

pub struct NetworkMonitor {
    sent_packets: u32,
    received_packets: u32,
    rtt_samples: Vec<u32>,
    windows_11_optimized: bool,
}

impl NetworkMonitor {
    pub fn new(windows_11_optimized: bool) -> Self {
        NetworkMonitor {
            sent_packets: 0,
            received_packets: 0,
            rtt_samples: Vec::new(),
            windows_11_optimized,
        }
    }
    
    pub fn update(&mut self, rtt: u32) {
        self.rtt_samples.push(rtt);
        if self.rtt_samples.len() > 100 {
            self.rtt_samples.remove(0);
        }
    }
    
    pub fn get_stats(&self) -> NetworkStats {
        let packet_loss_rate = if self.sent_packets > 0 {
            (self.sent_packets - self.received_packets) as f32 / self.sent_packets as f32
        } else {
            0.0
        };
        
        let rtt = if !self.rtt_samples.is_empty() {
            self.rtt_samples.iter().sum::<u32>() / self.rtt_samples.len() as u32
        } else {
            0
        };
        
        NetworkStats {
            packet_loss_rate,
            rtt,
            bandwidth: 0,
            jitter: 0,
        }
    }
}
```

## 5. 借鉴RustDesk代码

### 5.1 屏幕采集模块
参考 `src/server/video_service.rs` 中的屏幕采集实现：
- 使用DXGI Desktop Duplication API
- 多显示器支持
- 变化区域检测
- 升级到DXGI 1.6以支持Windows 11特性

### 5.2 编码器模块
参考 `src/server/video_service.rs` 中的编码器实现：
- VP8/VP9编码器封装
- 编码参数配置
- 硬件加速支持
- 添加AV1编码器支持

### 5.3 QoS管理
参考 `src/server/video_qos.rs` 中的QoS实现：
- 自适应码率控制
- 网络状况监控
- 性能优化策略
- 集成Windows 11硬件调度器

## 6. 性能优化

### 6.1 编码优化
- 使用硬件加速编码器（NVENC、QuickSync、VCE）
- 动态调整关键帧间隔
- 根据场景复杂度调整编码参数
- 利用Windows 11的硬件调度器优化CPU/GPU协作

### 6.2 网络优化
- 使用UDP进行数据传输
- 实现FEC（前向纠错）
- 优化数据包大小和发送频率
- 利用Windows 11优化的TCP/IP协议栈

### 6.3 渲染优化
- 使用DirectX 12 Ultimate硬件加速
- 实现零拷贝纹理更新
- 优化渲染管线
- 支持Auto HDR
- 支持可变刷新率（VRR）

### 6.4 Windows 11专项优化
- 启用游戏模式
- 优化硬件调度器
- 使用DirectStorage加速资源加载
- 利用Windows 11的内存管理改进
- 启用Auto HDR增强视觉效果

## 7. 未来扩展：WebRTC迁移

### 7.1 WebRTC优势
- 原生支持NAT穿透
- 内置自适应码率控制
- 内置丢包恢复机制
- 标准化协议，兼容性好
- Windows 11原生支持WebRTC

### 7.2 迁移方案
```
阶段1: 基础集成
├── 集成WebRTC库（webrtc-rs）
├── 实现信令服务器
└── 实现基本的视频传输

阶段2: 功能完善
├── 实现自适应码率
├── 实现丢包恢复
├── 优化延迟
└── 支持Windows 11特性

阶段3: 高级特性
├── 支持多路流
├── 支持音频传输
├── 支持录制功能
└── 支持云游戏
```

### 7.3 WebRTC实现示例

```rust
use webrtc::peer_connection::peer_connection::RTCPeerConnection;
use webrtc::peer_connection::peer_connection_state::RTCPeerConnectionState;

pub struct WebRTCStreamer {
    pc: RTCPeerConnection,
}

impl WebRTCStreamer {
    pub async fn new() -> Result<Self> {
        let config = RTCConfiguration {
            ice_servers: vec![RTCIceServer {
                urls: vec!["stun:stun.l.google.com:19302".to_string()],
                ..Default::default()
            }],
            ..Default::default()
        };
        
        let pc = RTCPeerConnection::new(config).await?;
        
        Ok(WebRTCStreamer { pc })
    }
    
    pub async fn start_stream(&mut self) -> Result<()> {
        let video_track = self.create_video_track().await?;
        self.pc.add_track(Arc::new(video_track)).await?;
        
        let offer = self.pc.create_offer(None).await?;
        self.pc.set_local_description(offer).await?;
        
        Ok(())
    }
}
```

## 8. Windows 11系统集成

### 8.1 系统要求
- **操作系统**: Windows 11 (22H2或更高版本)
- **处理器**: Intel Core 8代或更高 / AMD Ryzen 2000系列或更高
- **内存**: 16GB RAM推荐
- **显卡**: 支持DirectX 12 Ultimate的显卡
- **网络**: 千兆局域网

### 8.2 系统配置建议
- 启用游戏模式
- 启用硬件加速GPU调度
- 启用Auto HDR
- 启用可变刷新率
- 禁用不必要的后台应用
- 优化电源管理设置

### 8.3 性能监控
- 使用Windows 11性能监视器
- 监控CPU/GPU使用率
- 监控网络延迟和丢包率
- 监控帧率和延迟

## 9. 总结

本技术方案提供了一个基于Windows 11平台的局域网游戏串流系统设计，充分利用Windows 11的新特性，包括：

1. **高性能屏幕采集**: 使用DXGI 1.6 Desktop Duplication API
2. **DirectX 12 Ultimate**: 提供最高性能的渲染和编码
3. **多编码器支持**: AV1/VP9/H264，自动选择最优编码器
4. **低延迟传输**: UDP传输 + 自适应码率控制 + Windows 11网络优化
5. **硬件加速渲染**: DirectX 12 Ultimate + Auto HDR + 可变刷新率
6. **安全可靠**: 密码验证 + AES-256加密 + Windows 11安全特性
7. **QoS管理**: 自适应码率 + 网络状况监控 + 硬件调度器优化
8. **可扩展架构**: 支持未来WebRTC迁移

该系统充分利用Windows 11的新特性，可以满足局域网内高质量、低延迟的游戏串流需求，为用户提供流畅的游戏体验。