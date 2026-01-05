# RustDesk 技术架构文档

## 1 项目概述

RustDesk 是一款开源的远程桌面软件，采用 Rust 语言开发，支持 Windows、macOS、Linux、Android 和 iOS 等多个平台。项目采用模块化设计，具有良好的代码组织和跨平台兼容性。

| 属性 | 值 |
|------|-----|
| 项目名称 | RustDesk |
| 当前版本 | 1.4.4 |
| 编程语言 | Rust |
| 最低 Rust 版本 | 1.75 |
| 编辑器版本 | 2021 |
| 项目类型 | 远程桌面软件 |
| 开源协议 | GPL v3 |

## 2 项目结构

### 2.1 目录结构概览

```
rustdesk/
├── src/                   # 主程序源代码
│   ├── main.rs            # 程序入口
│   ├── lib.rs             # 库入口
│   ├── client.rs          # 客户端逻辑
│   ├── server.rs          # 服务端逻辑
│   ├── ui.rs              # UI 模块
│   ├── platform/          # 平台相关代码
│   ├── server/            # 服务端子模块
│   ├── ui/                # UI 组件
│   ├── whiteboard/        # 白板功能
│   ├── privacy_mode/      # 隐私模式
│   ├── plugin/            # 插件系统
│   ├── hbbs_http/         # HBBS HTTP 通信
│   └── lang/              # 多语言资源（40+ 语言）
├── libs/                  # 公共库
│   ├── scrap/             # 屏幕捕获库
│   ├── hbb_common/        # 公共功能库
│   ├── enigo/             # 输入模拟库
│   ├── clipboard/         # 剪贴板库
│   ├── virtual_display/   # 虚拟显示库
│   ├── portable/          # 便携软件库
│   └── remote_printer/    # 远程打印库
└── docs/                  # 文档目录
```

### 2.2 主程序模块 (src/)

#### 2.2.1 核心模块

| 文件 | 功能描述 |
|------|----------|
| [main.rs](file:///home/zy/repo/rustdesk/src/main.rs) | 程序入口，处理不同平台的启动逻辑 |
| [lib.rs](file:///home/zy/repo/rustdesk/src/lib.rs) | 定义库接口和公共导出 |
| [core_main.rs](file:///home/zy/repo/rustdesk/src/core_main.rs) | 核心初始化和主流程控制 |
| [common.rs](file:///home/zy/repo/rustdesk/src/common.rs) | 公共功能集合 |

#### 2.2.2 客户端模块

| 文件/目录 | 功能描述 |
|-----------|----------|
| [client.rs](file:///home/zy/repo/rustdesk/src/client.rs) | 客户端主逻辑 |
| [client/](file:///home/zy/repo/rustdesk/src/client/) | 客户端子模块目录 |
| [client/io_loop.rs](file:///home/zy/repo/rustdesk/src/client/io_loop.rs) | IO 事件循环 |
| [client/file_trait.rs](file:///home/zy/repo/rustdesk/src/client/file_trait.rs) | 文件传输接口 |
| [client/screenshot.rs](file:///home/zy/repo/rustdesk/src/client/screenshot.rs) | 截图功能 |
| [client/helper.rs](file:///home/zy/repo/rustdesk/src/client/helper.rs) | 客户端辅助函数 |

#### 2.2.3 服务端模块 (server/)

服务端模块负责处理远程连接的服务端功能，包括输入、显示、音频等核心服务。

| 文件 | 功能描述 |
|------|----------|
| [server.rs](file:///home/zy/repo/rustdesk/src/server.rs) | 服务端主入口 |
| [server/connection.rs](file:///home/zy/repo/rustdesk/src/server/connection.rs) | 连接管理 |
| [server/input_service.rs](file:///home/zy/repo/rustdesk/src/server/input_service.rs) | 输入事件处理服务 |
| [server/display_service.rs](file:///home/zy/repo/rustdesk/src/server/display_service.rs) | 显示服务 |
| [server/audio_service.rs](file:///home/zy/repo/rustdesk/src/server/audio_service.rs) | 音频传输服务 |
| [server/clipboard_service.rs](file:///home/zy/repo/rustdesk/src/server/clipboard_service.rs) | 剪贴板同步服务 |
| [server/video_service.rs](file:///home/zy/repo/rustdesk/src/server/video_service.rs) | 视频流服务 |
| [server/printer_service.rs](file:///home/zy/repo/rustdesk/src/server/printer_service.rs) | 远程打印服务 |
| [server/terminal_service.rs](file:///home/zy/repo/rustdesk/src/server/terminal_service.rs) | 终端仿真服务 |
| [server/dbus.rs](file:///home/zy/repo/rustdesk/src/server/dbus.rs) | D-Bus 服务（Linux） |
| [server/uinput.rs](file:///home/zy/repo/rustdesk/src/server/uinput.rs) | uinput 设备支持（Linux） |
| [server/portable_service.rs](file:///home/zy/repo/rustdesk/src/server/portable_service.rs) | 便携模式服务 |
| [server/rdp_input.rs](file:///home/zy/repo/rustdesk/src/server/rdp_input.rs) | RDP 输入处理 |
| [server/wayland.rs](file:///home/zy/repo/rustdesk/src/server/wayland.rs) | Wayland 协议支持 |
| [server/video_qos.rs](file:///home/zy/repo/rustdesk/src/server/video_qos.rs) | 视频质量控制 |

#### 2.2.4 UI 模块

| 文件/目录 | 功能描述 |
|-----------|----------|
| [ui.rs](file:///home/zy/repo/rustdesk/src/ui.rs) | UI 主模块 |
| [ui/](file:///home/zy/repo/rustdesk/src/ui/) | UI 组件目录 |
| [ui/remote.rs](file:///home/zy/repo/rustdesk/src/ui/remote.rs) | 远程桌面 UI |
| [ui/cm.rs](file:///home/zy/repo/rustdesk/src/ui/cm.rs) | 连接管理 UI |
| [ui_cm_interface.rs](file:///home/zy/repo/rustdesk/src/ui_cm_interface.rs) | 连接管理界面接口 |
| [ui_session_interface.rs](file:///home/zy/repo/rustdesk/src/ui_session_interface.rs) | 会话界面接口 |

#### 2.2.5 平台模块 (platform/)

平台模块封装了各操作系统的特定功能，提供统一的抽象接口。

| 文件 | 功能描述 |
|------|----------|
| [platform/mod.rs](file:///home/zy/repo/rustdesk/src/platform/mod.rs) | 平台模块入口 |
| [platform/windows.rs](file:///home/zy/repo/rustdesk/src/platform/windows.rs) | Windows 平台实现 |
| [platform/macos.rs](file:///home/zy/repo/rustdesk/src/platform/macos.rs) | macOS 平台实现 |
| [platform/linux.rs](file:///home/zy/repo/rustdesk/src/platform/linux.rs) | Linux 平台实现 |
| [platform/win_device.rs](file:///home/zy/repo/rustdesk/src/platform/win_device.rs) | Windows 设备管理 |
| [platform/gtk_sudo.rs](file:///home/zy/repo/rustdesk/src/platform/gtk_sudo.rs) | GTK sudo 认证 |
| [platform/linux_desktop_manager.rs](file:///home/zy/repo/rustdesk/src/platform/linux_desktop_manager.rs) | Linux 桌面管理 |
| [platform/delegate.rs](file:///home/zy/repo/rustdesk/src/platform/delegate.rs) | 委托处理 |

#### 2.2.6 通信模块

| 文件/目录 | 功能描述 |
|-----------|----------|
| [rendezvous_mediator.rs](file:///home/zy/repo/rustdesk/src/rendezvous_mediator.rs) | Rendezvous 中继服务 |
| [hbbs_http/](file:///home/zy/repo/rustdesk/src/hbbs_http/) | HBBS HTTP 通信模块 |
| [ipc.rs](file:///home/zy/repo/rustdesk/src/ipc.rs) | 进程间通信 |
| [kcp_stream.rs](file:///home/zy/repo/rustdesk/src/kcp_stream.rs) | KCP 协议流处理 |
| [port_forward.rs](file:///home/zy/repo/rustdesk/src/port_forward.rs) | 端口转发功能 |

#### 2.2.7 白板模块 (whiteboard/)

白板模块提供实时协作标注功能，支持跨平台绘制和标注。

| 文件 | 功能描述 |
|------|----------|
| [whiteboard/mod.rs](file:///home/zy/repo/rustdesk/src/whiteboard/mod.rs) | 白板模块入口 |
| [whiteboard/client.rs](file:///home/zy/repo/rustdesk/src/whiteboard/client.rs) | 白板客户端 |
| [whiteboard/server.rs](file:///home/zy/repo/rustdesk/src/whiteboard/server.rs) | 白板服务端 |
| [whiteboard/windows.rs](file:///home/zy/repo/rustdesk/src/whiteboard/windows.rs) | Windows 平台实现 |
| [whiteboard/macos.rs](file:///home/zy/repo/rustdesk/src/whiteboard/macos.rs) | macOS 平台实现 |
| [whiteboard/linux.rs](file:///home/zy/repo/rustdesk/src/whiteboard/linux.rs) | Linux 平台实现 |

#### 2.2.8 隐私模式模块 (privacy_mode/)

隐私模式模块提供高级隐私保护功能，支持隐藏屏幕、限制输入等安全特性。

| 文件 | 功能描述 |
|------|----------|
| [privacy_mode.rs](file:///home/zy/repo/rustdesk/src/privacy_mode.rs) | 隐私模式主模块 |
| [privacy_mode/win_virtual_display.rs](file:///home/zy/repo/rustdesk/src/privacy_mode/win_virtual_display.rs) | Windows 虚拟显示 |
| [privacy_mode/win_exclude_from_capture.rs](file:///home/zy/repo/rustdesk/src/privacy_mode/win_exclude_from_capture.rs) | 捕获排除功能 |
| [privacy_mode/win_topmost_window.rs](file:///home/zy/repo/rustdesk/src/privacy_mode/win_topmost_window.rs) | 最顶层窗口 |
| [privacy_mode/win_input.rs](file:///home/zy/repo/rustdesk/src/privacy_mode/win_input.rs) | 输入控制 |
| [privacy_mode/win_mag.rs](file:///home/zy/repo/rustdesk/src/privacy_mode/win_mag.rs) | MAG 滤镜 |

#### 2.2.9 插件模块 (plugin/)

插件系统提供高度可扩展的架构，允许加载第三方插件扩展功能。

| 文件 | 功能描述 |
|------|----------|
| [plugin/mod.rs](file:///home/zy/repo/rustdesk/src/plugin/mod.rs) | 插件模块入口 |
| [plugin/manager.rs](file:///home/zy/repo/rustdesk/src/plugin/manager.rs) | 插件管理器 |
| [plugin/config.rs](file:///home/zy/repo/rustdesk/src/plugin/config.rs) | 插件配置 |
| [plugin/native.rs](file:///home/zy/repo/rustdesk/src/plugin/native.rs) | 原生插件接口 |
| [plugin/ipc.rs](file:///home/zy/repo/rustdesk/src/plugin/ipc.rs) | 插件 IPC |
| [plugin/native_handlers/](file:///home/zy/repo/rustdesk/src/plugin/native_handlers/) | 原生处理器 |

#### 2.2.10 其他功能模块

| 文件 | 功能描述 |
|------|----------|
| [keyboard.rs](file:///home/zy/repo/rustdesk/src/keyboard.rs) | 键盘映射和处理 |
| [clipboard.rs](file:///home/zy/repo/rustdesk/src/clipboard.rs) | 剪贴板管理 |
| [clipboard_file.rs](file:///home/zy/repo/rustdesk/src/clipboard_file.rs) | 剪贴板文件传输 |
| [auth_2fa.rs](file:///home/zy/repo/rustdesk/src/auth_2fa.rs) | 双因素认证 |
| [tray.rs](file:///home/zy/repo/rustdesk/src/tray.rs) | 系统托盘 |
| [service.rs](file:///home/zy/repo/rustdesk/src/service.rs) | 系统服务管理 |
| [naming.rs](file:///home/zy/repo/rustdesk/src/naming.rs) | ID 命名服务 |
| [updater.rs](file:///home/zy/repo/rustdesk/src/updater.rs) | 自动更新 |
| [lan.rs](file:///home/zy/repo/rustdesk/src/lan.rs) | 局域网发现 |
| [flutter.rs](file:///home/zy/repo/rustdesk/src/flutter.rs) | Flutter flutter_ffi.rs](file:///home集成 |
| [/zy/repo/rustdesk/src/flutter_ffi.rs) | Flutter FFI 桥接 |
| [lang.rs](file:///home/zy/repo/rustdesk/src/lang.rs) | 语言管理 |
| [lang/](file:///home/zy/repo/rustdesk/src/lang/) | 多语言资源（40+ 语言） |
| [virtual_display_manager.rs](file:///home/zy/repo/rustdesk/src/virtual_display_manager.rs) | 虚拟显示管理 |
| [custom_server.rs](file:///home/zy/repo/rustdesk/src/custom_server.rs) | 自定义服务器配置 |

### 2.3 公共库模块 (libs/)

#### 2.3.1 scrap - 屏幕捕获库

[scrap](file:///home/zy/repo/rustdesk/libs/scrap) 是一个跨平台屏幕捕获库，支持多种底层图形 API。

| 目录/文件 | 平台 | 功能描述 |
|-----------|------|----------|
| [src/lib.rs](file:///home/zy/repo/rustdesk/libs/scrap/src/lib.rs) | 通用 | 库入口和公共接口 |
| [src/common/](file:///home/zy/repo/rustdesk/libs/scrap/src/common/) | 通用 | 公共捕获逻辑 |
| [src/x11/](file:///home/zy/repo/rustdesk/libs/scrap/src/x11/) | Linux | X11 屏幕捕获 |
| [src/wayland/](file:///home/zy/repo/rustdesk/libs/scrap/src/wayland/) | Linux | Wayland 屏幕捕获 |
| [src/quartz/](file:///home/zy/repo/rustdesk/libs/scrap/src/quartz/) | macOS | Quartz 屏幕捕获 |
| [src/dxgi/](file:///home/zy/repo/rustdesk/libs/scrap/src/dxgi/) | Windows | DirectX GI 捕获 |
| [src/android/](file:///home/zy/repo/rustdesk/libs/scrap/src/android/) | Android | Android 屏幕捕获 |

**特性支持**：
- 硬件编解码（hwcodec）
- 虚拟显存（vram）
- 媒体编解码（mediacodec）
- 摄像头支持（通过 nokhwa）

#### 2.3.2 hbb_common - 公共功能库

hbb_common 是项目的公共基础库，提供通用的数据结构和工具函数。

**主要功能**：
- 配置管理
- 日志系统
- 加密工具
- 网络工具
- 平台抽象

#### 2.3.3 enigo - 输入模拟库

[enigo](file:///home/zy/repo/rustdesk/libs/enigo) 提供跨平台的鼠标和键盘输入模拟功能。

| 目录 | 平台 |
|------|------|
| [src/win/](file:///home/zy/repo/rustdesk/libs/enigo/src/win/) | Windows |
| [src/macos/](file:///home/zy/repo/rustdesk/libs/enigo/src/macos/) | macOS |
| [src/linux/](file:///home/zy/repo/rustdesk/libs/enigo/src/linux/) | Linux |

#### 2.3.4 clipboard - 剪贴板库

[clipboard](file:///home/zy/repo/rustdesk/libs/clipboard) 提供跨平台的剪贴板访问和文件传输支持。

| 目录 | 功能描述 |
|------|----------|
| [src/platform/windows.rs](file:///home/zy/repo/rustdesk/libs/clipboard/src/platform/windows.rs) | Windows 剪贴板 |
| [src/platform/unix/](file:///home/zy/repo/rustdesk/libs/clipboard/src/platform/unix/) | Unix 剪贴板（含 macOS） |

**特性**：
- 文本/图像/文件传输
- 跨平台剪贴板同步
- FUSE 文件系统支持（Linux）

#### 2.3.5 virtual_display - 虚拟显示库

[virtual_display](file:///home/zy/repo/rustdesk/libs/virtual_display) 为 Windows 提供虚拟显示功能，允许创建额外的虚拟显示器。

#### 2.3.6 remote_printer - 远程打印库

[remote_printer](file:///home/zy/repo/rustdesk/libs/remote_printer) 提供远程打印功能，支持将本地打印机共享到远程端。

## 3 技术栈详解

### 3.1 异步与并发

| 依赖 | 版本 | 用途 |
|------|------|------|
| async-trait | 0.1 | 异步 trait 语法支持 |
| parity-tokio-ipc | Git | 基于 Tokio 的 IPC |
| crossbeam-queue | 0.3 | 高性能并发队列 |
| tokio | 隐式 | 异步运行时（通过其他库使用） |

### 3.2 序列化与配置

| 依赖 | 版本 | 用途 |
|------|------|------|
| serde | 1.0 | 序列化框架 |
| serde_derive | 1.0 | 派生宏 |
| serde_json | 1.0 | JSON 序列化 |
| serde_repr | 0.1 | 枚举序列化 |
| lazy_static | 1.4 | 静态延迟初始化 |
| cfg-if | 1.0 | 条件编译 |

### 3.3 网络通信

| 依赖 | 版本 | 用途 |
|------|------|------|
| reqwest | 0.12 | HTTP 客户端（支持 SOCKS、JSON、TLS） |
| kcp-sys | Git | KCP 协议（可靠 UDP） |
| stunclient | 0.4 | STUN 客户端（NAT 穿透） |
| wol-rs | 1.0 | Wake-on-LAN |
| default-net | 0.14 | 网络接口信息 |
| url | 2.3 | URL 解析 |
| cidr-utils | 0.5 | CIDR 处理 |

### 3.4 加密与安全

| 依赖 | 版本 | 用途 |
|------|------|------|
| sha2 | 0.10 | SHA-256 等哈希算法 |
| uuid | 1.3 | UUID 生成 |
| hex | 0.4 | 十六进制编码 |
| ringbuf | 0.3 | 环形缓冲区 |
| errno | 0.3 | 错误码处理 |

### 3.5 音频处理

| 依赖 | 版本 | 用途 |
|------|------|------|
| magnum-opus | Git | Opus 音频编解码 |
| fon | 0.6 | 音频帧处理 |
| dasp | 0.11 | 数字信号处理（可选） |
| rubato | 0.12 | 音频重采样（可选） |
| samplerate | 0.2 | 采样率转换（可选） |

### 3.6 窗口与 UI 框架

| 依赖 | 版本 | 用途 |
|------|------|------|
| tao | Git | 窗口管理（TAO 派生） |
| tray-icon | Git | 系统托盘图标 |
| sciter-rs | Git | Sciter UI 框架 |
| winit | 0.30 | 窗口和事件循环（Linux） |
| gtk | 0.18 | GTK 界面（Linux） |
| image | 0.24 | 图像处理 |

### 3.7 图形与渲染

| 依赖 | 版本 | 用途 |
|------|------|------|
| tiny-skia | 0.11 | Skia 图形库 |
| piet | 0.6 | 2D 渲染库 |
| piet-coregraphics | 0.6 | macOS 渲染后端 |
| softbuffer | 0.4 | 软件缓冲 |
| fontdb | 0.23 | 字体数据库 |
| ttf-parser | 0.25 | TTF 字体解析 |
| bytemuck | 1.23 | 字节操作 |

### 3.8 平台特定依赖

#### Windows

| 依赖 | 版本 | 用途 |
|------|------|------|
| winapi | 0.3 | Windows API 绑定 |
| windows | 0.61 | Windows SDK 包装 |
| winreg | 0.11 | 注册表访问 |
| windows-service | 0.6 | Windows 服务 |
| virtual_display | 本地 | 虚拟显示 |
| remote_printer | 本地 | 远程打印 |
| impersonate-system | Git | 令牌模拟 |
| shared_memory | 0.12 | 共享内存 |
| runas | 1.2 | 以管理员运行 |
| virtual_display | 本地 | 虚拟显示驱动 |

#### macOS

| 依赖 | 版本 | 用途 |
|------|------|------|
| objc | 0.2 | Objective-C 运行时 |
| cocoa | 0.24 | Cocoa 框架 |
| core-foundation | 0.9 | Core Foundation |
| core-graphics | 0.22 | Core Graphics |
| dispatch | 0.2 | GCD 绑定 |
| fruitbasket | 0.10 | macOS 应用构建 |
| piet-coregraphics | 0.6 | macOS 渲染 |

#### Linux

| 依赖 | 版本 | 用途 |
|------|------|------|
| psimple | 2.27 | PulseAudio 简单客户端 |
| pulse | 2.27 | PulseAudio 绑定 |
| rust-pulsectl | Git | PulseAudio 控制 |
| dbus | 0.9 | D-Bus 绑定 |
| dbus-crossroads | 0.5 | D-Bus 路由器 |
| evdev | Git | 输入设备 |
| pam | Git | PAM 认证 |
| x11-clipboard | Git | X11 剪贴板 |
| x11rb | 0.12 | X11 绑定 |
| nix | 0.29 | Unix 系统调用 |
| termios | 0.3 | 终端控制 |
| terminfo | 0.8 | 终端信息 |
| gtk | 0.18 | GTK 界面 |
| winit | 0.30 | 窗口管理 |

#### 移动平台

| 依赖 | 版本 | 用途 |
|------|------|------|
| android_logger | 0.13 | Android 日志 |
| jni | 0.21 | JNI 绑定 |
| android-wakelock | Git | 唤醒锁 |
| cpal | Git | 音频处理 |

### 3.9 输入设备

| 依赖 | 版本 | 用途 |
|------|------|------|
| rdev | Git | 事件捕获 |
| enigo | 本地 | 输入模拟 |

### 3.10 系统集成

| 依赖 | 版本 | 用途 |
|------|------|------|
| ctrlc | 3.2 | Ctrl+C 处理 |
| system_shutdown | 4.0 | 系统关机 |
| mac_address | 1.1 | MAC 地址 |
| sys-locale | 0.3 | 系统语言 |
| qrcode-generator | 4.1 | 二维码生成 |
| portable-pty | Git | PTY 处理 |
| arboard | Git | 剪贴板（多平台） |
| clipboard-master | Git | 剪贴板扩展 |
| keepawake | Git | 阻止休眠 |

### 3.11 其他工具库

| 依赖 | 版本 | 用途 |
|------|------|------|
| chrono | 0.4 | 日期时间 |
| zip | 0.6 | ZIP 压缩 |
| totp-rs | 5.4 | TOTP 认证 |
| repng | 0.2 | PNG 处理 |
| webm | Git | WebM 封装 |
| libloading | 0.8 | 动态库加载 |
| clap | 4.2 | 命令行解析 |
| rpassword | 7.2 | 密码输入 |
| num_cpus | 1.15 | CPU 核心数 |
| fon | 0.6 | 音频帧 |
| shutdown_hooks | 0.1 | 关闭钩子 |

### 3.12 构建工具

| 依赖 | 版本 | 用途 |
|------|------|------|
| cc | 1.0 | C/C++ 编译器 |
| bindgen | 0.65 | C 绑定生成 |
| pkg-config | 0.3 | 包配置查询 |
| target_build_utils | 0.3 | 目标检测 |
| winres | 0.1 | Windows 资源 |
| hound | 3.5 | 音频样本 |
| docopt | 1.1 | 命令行文档 |

## 4 模块依赖关系

### 4.1 核心依赖图

```
main.rs/lib.rs
    ├── core_main.rs
    │   ├── common.rs
    │   ├── ui.rs
    │   │   ├── platform/ (跨平台 UI)
    │   │   └── ui_cm_interface.rs
    │   ├── server.rs
    │   │   └── server/ (各服务模块)
    │   └── client.rs
    │       └── client/ (客户端子模块)
    ├── hbb_common (公共库)
    │   ├── serde (序列化)
    │   ├── tokio (异步)
    │   └── 其他工具库
    └── scrap (屏幕捕获)
        ├── 平台特定实现
        └── hwcodec (硬件编解码)
```

### 4.2 平台抽象层

```
platform/mod.rs
    ├── Windows: platform/windows.rs
    ├── macOS: platform/macos.rs
    └── Linux: platform/linux.rs
```

### 4.3 服务间依赖

```
server.rs
    ├── connection.rs (连接管理)
    │   ├── input_service.rs (输入)
    │   ├── display_service.rs (显示)
    │   ├── audio_service.rs (音频)
    │   └── clipboard_service.rs (剪贴板)
    ├── video_service.rs (视频)
    │   └── scrap (屏幕捕获)
    └── printer_service.rs (打印)
        └── remote_printer (远程打印库)
```

## 5 构建特性 (Cargo Features)

### 5.1 通用特性

| 特性 | 默认值 | 描述 |
|------|--------|------|
| flutter | 否 | Flutter 支持 |
| cli | 否 | 命令行模式 |
| inline | 否 | 内联构建 |
| plugin_framework | 否 | 插件框架 |
| use_dasp | 是 | 使用 dasp 音频处理 |
| use_rubato | 否 | 使用 rubato 音频处理 |
| use_samplerate | 否 | 使用 samplerate 音频处理 |

### 5.2 平台相关特性

| 特性 | 平台 | 描述 |
|------|------|------|
| hwcodec | Windows | 硬件编解码支持 |
| vram | Windows | 虚拟显存 |
| mediacodec | Android | 媒体编解码 |
| screencapturekit | macOS | 屏幕捕获 |
| linux-pkg-config | Linux | Linux 包配置 |
| unix-file-copy-paste | Unix | Unix 文件复制粘贴 |

### 5.3 构建配置

```toml
[profile.release]
lto = true              # 链接时间优化
codegen-units = 1       # 编译单元
panic = 'abort'         # 中止 panic
strip = true            # 符号剥离
rpath = true            # 设置 RPATH
```

## 6 特色功能模块详解

### 6.1 隐私模式

隐私模式提供多层次的屏幕隐私保护：

1. **虚拟显示模式**：创建虚拟显示器，远程用户看到空白屏幕
2. **捕获排除模式**：特定窗口从远程视图排除
3. **最顶层窗口模式**：显示覆盖窗口阻止屏幕查看
4. **输入控制模式**：阻止远程输入

### 6.2 插件系统

RustDesk 提供完整的插件框架：

- **插件管理器** ([plugin/manager.rs](file:///home/zy/repo/rustdesk/src/plugin/manager.rs))：插件生命周期管理
- **原生处理器** ([plugin/native_handlers/](file:///home/zy/repo/rustdesk/src/plugin/native_handlers/))：原生功能暴露
- **IPC 通信** ([plugin/ipc.rs](file:///home/zy/repo/rustdesk/src/plugin/ipc.rs))：主程序与插件间通信

### 6.3 白板协作

实时协作标注功能：

- 跨平台绘图引擎
- 多种绘图工具（画笔、形状、文字等）
- 实时同步多端状态

### 6.4 屏幕捕获

多层次屏幕捕获架构：

| 层级 | 技术 | 平台 |
|------|------|------|
| 高级 API | scrap 统一接口 | 全部 |
| 显示协议 | X11/Wayland/Quartz/DXGI | 各平台 |
| 硬件加速 | hwcodec (NVENC/VAAPI) | Windows/Linux |
| 媒体编码 | libaom/mediacodec | 全部 |

### 6.5 虚拟显示

Windows 平台专用功能：

- 创建虚拟显示设备
- 驱动程序级别集成
- 支持隐私模式和安全显示

## 7 跨平台支持策略

### 7.1 平台检测

使用 Rust 条件编译和 `cfg-if` 实现：

```rust
#[cfg(target_os = "windows")]
// Windows 特定代码

#[cfg(target_os = "macos")]
// macOS 特定代码

#[cfg(target_os = "linux")]
// Linux 特定代码
```

### 7.2 平台抽象

通过 `platform/` 目录提供统一接口：

```
platform/mod.rs (统一接口)
    ├── get_active_display()     // 获取活动显示
    ├── get_displays()           // 获取所有显示
    ├── set_power_management()   // 电源管理
    └── ...
```

### 7.3 功能降级

根据平台特性启用/禁用功能：

- 硬件编解码：仅 Windows/Linux
- 虚拟显示：仅 Windows
- D-Bus：仅 Linux
- ScreenCaptureKit：仅 macOS

## 8 性能优化策略

### 8.1 发布构建优化

```toml
[profile.release]
lto = true              # 链接时间优化，减少二进制大小
codegen-units = 1       # 单编译单元，启用更多优化
panic = 'abort'         # 避免 panic 展开，减少大小
strip = true            # 剥离调试符号
rpath = true            # 正确设置库路径
```

### 8.2 异步架构

- 使用 Tokio 异步运行时
- 独立的 IO 事件循环
- 非阻塞网络操作

### 8.3 内存管理

- 使用 `lazy_static` 延迟初始化
- `ringbuf` 高效环形缓冲区
- 避免不必要的内存分配

## 9 总结

RustDesk 是一个功能完善的远程桌面解决方案，采用 Rust 语言实现，展示了以下技术特点：

1. **模块化设计**：清晰分离客户端、服务端、UI 和平台代码
2. **跨平台支持**：统一抽象层封装平台差异
3. **高性能**：Rust 的内存安全保证和异步架构
4. **可扩展性**：插件系统支持第三方扩展
5. **丰富的功能**：隐私模式、白板协作、远程打印等

项目的技术栈涵盖了从底层系统编程到高层 UI 框架的完整技术链条，是学习现代 Rust 桌面应用开发的优秀参考。
