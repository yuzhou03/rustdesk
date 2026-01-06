# Display Service 模块技术文档

## 1. 功能概述

`display_service.rs` 是 RustDesk 显示器服务模块，负责管理显示器信息、检测显示器变化、处理分辨率调整以及与客户端同步显示器状态。该模块是视频服务的基础，为远程桌面连接提供显示器信息支持。

### 核心功能

- **显示器信息管理**：获取、存储和同步显示器信息
- **显示器变化检测**：实时检测显示器插拔、分辨率变化等事件
- **分辨率管理**：记录和恢复显示器分辨率
- **虚拟显示器支持**：在 Windows 平台支持虚拟显示器（headless 模式）
- **跨平台适配**：支持 Windows、Linux、macOS 等多平台
- **隐私模式支持**：Windows 平台支持放大镜捕获器（Magnifier）隐私模式

## 2. 核心实现逻辑

### 2.1 显示器信息同步机制

模块使用 `SYNC_DISPLAYS` 静态变量来同步显示器信息，采用生产者-消费者模式：

```
生产者：check_update_displays() - 更新显示器信息
消费者：get_displays_msg() - 获取变化的消息
```

**同步流程**：
1. 定期调用 `check_update_displays()` 更新显示器信息
2. 比较新旧显示器信息，检测变化
3. 如果检测到变化，设置 `is_synced = false`
4. `get_displays_msg()` 检查 `is_synced` 标志
5. 如果未同步，生成显示器消息并发送给客户端

### 2.2 显示器变化检测

显示器变化检测基于以下条件：

- 显示器数量变化
- 显示器位置变化（x, y 坐标）
- 显示器分辨率变化（width, height）
- 显示器名称变化

**特殊处理**：
- Linux Wayland：不支持显示器切换，直接返回
- Windows：支持虚拟显示器自动创建
- 临时忽略：通过 `TEMP_IGNORE_DISPLAYS_CHANGED` 标志临时忽略变化

### 2.3 分辨率管理

分辨率管理使用 `CHANGED_RESOLUTIONS` 静态变量记录分辨率变化：

```rust
struct ChangedResolution {
    original: (i32, i32),  // 原始分辨率
    changed: (i32, i32),   // 修改后的分辨率
}
```

**管理流程**：
1. 用户修改分辨率时，记录原始和修改后的分辨率
2. 断开连接时，调用 `restore_resolutions()` 恢复原始分辨率
3. 清空分辨率记录

### 2.4 虚拟显示器支持（Windows）

Windows 平台支持虚拟显示器，用于 headless 场景：

**触发条件**：
- 平台已安装
- 虚拟显示器功能已启用
- 检测到无显示器或仅虚拟显示器

**实现逻辑**：
1. 获取当前显示器列表
2. 检查是否需要虚拟显示器
3. 如果需要，调用 `plug_in_headless()` 创建虚拟显示器
4. 重新获取显示器列表

## 3. 主要函数说明

### 3.1 服务启动函数

#### `new() -> GenericService`

**功能**：创建并启动显示器服务

**返回值**：`GenericService` 实例

**实现逻辑**：
1. 创建 `EmptyExtraFieldService` 实例
2. 启动服务运行循环
3. 返回服务实例

#### `run(sp: EmptyExtraFieldService) -> ResultType<()>`

**功能**：服务主循环，定期检测显示器变化

**参数**：
- `sp`: 服务实例

**实现逻辑**：
1. 每 300ms 检查一次
2. 检查是否有新订阅者
3. 检查显示器是否变化
4. 如果变化，发送显示器消息给客户端

### 3.2 显示器信息获取函数

#### `try_get_displays() -> ResultType<Vec<Display>>`

**功能**：获取所有显示器信息（非 Windows 平台）

**返回值**：显示器列表

**实现**：直接调用 `Display::all()`

#### `try_get_displays_() -> ResultType<Vec<Display>>` (Windows)

**功能**：获取所有显示器信息（Windows 平台）

**返回值**：显示器列表

**实现逻辑**：
1. 获取当前显示器列表
2. 检查是否需要虚拟显示器
3. 如果需要，创建虚拟显示器
4. 返回显示器列表

#### `try_get_displays_add_amyuni_headless() -> ResultType<Vec<Display>>` (Windows)

**功能**：获取显示器信息，必要时添加 amyuni 虚拟显示器

**返回值**：显示器列表

**实现逻辑**：
1. 调用 `try_get_displays_(true)`
2. 传递 `add_amyuni_headless = true` 参数

#### `get_sync_displays() -> Vec<DisplayInfo>`

**功能**：获取同步的显示器信息

**返回值**：显示器信息列表

**实现**：从 `SYNC_DISPLAYS` 中读取

#### `get_display_info(idx: usize) -> Option<DisplayInfo>`

**功能**：获取指定索引的显示器信息

**参数**：
- `idx`: 显示器索引

**返回值**：显示器信息（如果存在）

### 3.3 显示器变化检测函数

#### `check_displays_changed() -> ResultType<()>`

**功能**：检查显示器是否变化

**返回值**：操作结果

**实现逻辑**：
1. 获取当前显示器列表
2. 调用 `check_update_displays()` 更新同步信息
3. 返回结果

#### `check_display_changed(ndisplay: usize, idx: usize, (x, y, w, h): (i32, i32, usize, usize)) -> Option<DisplayInfo>`

**功能**：检查指定显示器是否变化

**参数**：
- `ndisplay`: 当前显示器数量
- `idx`: 显示器索引
- `(x, y, w, h)`: 显示器位置和大小

**返回值**：
- `Some(display_info)`: 显示器变化
- `None`: 显示器未变化

**实现逻辑**：
1. 获取存储的显示器信息
2. 比较显示器数量
3. 比较显示器位置和大小
4. 如果不同，返回显示器信息

#### `check_update_displays(all: &Vec<Display>)`

**功能**：更新并检查显示器信息

**参数**：
- `all`: 显示器列表

**实现逻辑**：
1. 遍历显示器列表
2. 转换为 `DisplayInfo` 格式
3. 处理缩放比例（macOS 和 Linux Wayland）
4. 获取原始分辨率
5. 调用 `check_changed()` 更新同步信息

### 3.4 消息生成函数

#### `displays_to_msg(displays: Vec<DisplayInfo>) -> Message`

**功能**：将显示器列表转换为消息

**参数**：
- `displays`: 显示器信息列表

**返回值**：`Message` 实例

**实现逻辑**：
1. 创建 `PeerInfo` 实例
2. 设置显示器列表
3. Windows 平台：添加平台附加信息
4. 设置当前显示器为 0（兼容性）
5. 生成消息

#### `get_displays_msg() -> Option<Message>`

**功能**：获取显示器变化消息

**返回值**：
- `Some(msg)`: 有显示器变化
- `None`: 无变化

**实现逻辑**：
1. 获取更新的显示器信息
2. 调用 `displays_to_msg()` 生成消息

#### `check_get_displays_changed_msg() -> Option<Message>`

**功能**：检查并获取显示器变化消息

**返回值**：
- `Some(msg)`: 有显示器变化
- `None`: 无变化

**实现逻辑**：
1. Linux Wayland：直接返回显示器消息
2. 其他平台：检查显示器变化
3. 生成并返回消息

### 3.5 分辨率管理函数

#### `set_last_changed_resolution(display_name: &str, original: (i32, i32), changed: (i32, i32))`

**功能**：记录分辨率变化

**参数**：
- `display_name`: 显示器名称
- `original`: 原始分辨率
- `changed`: 修改后的分辨率

**实现逻辑**：
1. 获取分辨率记录锁
2. 更新或插入分辨率记录

#### `restore_resolutions()`

**功能**：恢复所有显示器的原始分辨率

**实现逻辑**：
1. 遍历所有分辨率记录
2. 调用 `change_resolution()` 恢复分辨率
3. 清空分辨率记录

#### `get_original_resolution(display_name: &str, w: usize, h: usize) -> MessageField<Resolution>`

**功能**：获取显示器的原始分辨率

**参数**：
- `display_name`: 显示器名称
- `w`: 当前宽度
- `h`: 当前高度

**返回值**：原始分辨率

**实现逻辑**：
1. 检查是否为 RustDesk 虚拟显示器
2. 如果是，返回空分辨率
3. 否则，从记录中获取原始分辨率
4. 如果没有记录，使用当前分辨率

### 3.6 平台特定函数

#### `is_capturer_mag_supported() -> bool`

**功能**：检查是否支持放大镜捕获器

**返回值**：是否支持

**实现**：
- Windows：调用 `scrap::CapturerMag::is_supported()`
- 其他平台：返回 false

#### `capture_cursor_embedded() -> bool`

**功能**：检查是否支持嵌入光标捕获

**返回值**：是否支持

**实现**：调用 `scrap::is_cursor_embedded()`

#### `is_privacy_mode_mag_supported() -> bool` (Windows)

**功能**：检查是否支持隐私模式（放大镜）

**返回值**：是否支持

**实现逻辑**：
1. 检查是否支持放大镜捕获器
2. 检查版本是否 > 1.1.9

#### `no_displays(displays: &Vec<Display>) -> bool` (Windows)

**功能**：检查是否无真实显示器

**参数**：
- `displays`: 显示器列表

**返回值**：是否无真实显示器

**实现逻辑**：
1. 检查显示器数量
2. 如果为 0，返回 true
3. 如果为 1，检查是否为虚拟显示器
4. 检查是否有真实分辨率支持

### 3.7 主显示器函数

#### `get_primary() -> usize`

**功能**：获取主显示器索引

**返回值**：主显示器索引

**实现逻辑**：
1. Linux Wayland：调用 `wayland::get_primary()`
2. 其他平台：调用 `get_primary_2()`

#### `get_primary_2(all: &Vec<Display>) -> usize`

**功能**：从显示器列表中获取主显示器索引

**参数**：
- `all`: 显示器列表

**返回值**：主显示器索引

**实现**：查找 `is_primary()` 为 true 的显示器

### 3.8 临时忽略函数

#### `temp_ignore_displays_changed() -> SimpleCallOnReturn`

**功能**：临时忽略显示器变化

**返回值**：`SimpleCallOnReturn` 实例（RAII 模式）

**实现逻辑**：
1. 设置 `TEMP_IGNORE_DISPLAYS_CHANGED = true`
2. 创建 `SimpleCallOnReturn` 实例
3. 在析构时：
   - 等待 1 秒
   - 恢复 `TEMP_IGNORE_DISPLAYS_CHANGED = false`
   - 触发显示器变化检查

### 3.9 初始化和更新函数

#### `is_inited_msg() -> Option<Message>`

**功能**：获取初始化消息

**返回值**：
- `Some(msg)`: 有初始化消息
- `None`: 无初始化消息

**实现**：
- Linux Wayland：调用 `wayland::is_inited()`
- 其他平台：返回 None

#### `update_get_sync_displays_on_login() -> ResultType<Vec<DisplayInfo>>`

**功能**：登录时更新并获取同步显示器信息

**返回值**：显示器信息列表

**实现逻辑**：
1. Linux Wayland：调用 `wayland::get_displays()`
2. Windows：调用 `try_get_displays_add_amyuni_headless()`
3. 其他平台：调用 `try_get_displays()`
4. 更新同步信息
5. 返回显示器列表

## 4. 参数定义

### 4.1 常量定义

```rust
pub const NAME: &'static str = "display";  // 服务名称

#[cfg(windows)]
const DUMMY_DISPLAY_SIDE_MAX_SIZE: usize = 1024;  // 虚拟显示器最大尺寸
```

### 4.2 静态变量

```rust
static ref IS_CAPTURER_MAGNIFIER_SUPPORTED: bool;  // 是否支持放大镜捕获器
static ref CHANGED_RESOLUTIONS: Arc<RwLock<HashMap<String, ChangedResolution>>>;  // 分辨率变化记录
pub static ref PRIMARY_DISPLAY_IDX: usize;  // 主显示器索引
static ref SYNC_DISPLAYS: Arc<Mutex<SyncDisplaysInfo>>;  // 同步显示器信息
static TEMP_IGNORE_DISPLAYS_CHANGED: AtomicBool;  // 临时忽略显示器变化标志
```

### 4.3 数据结构

#### ChangedResolution

```rust
struct ChangedResolution {
    original: (i32, i32),  // 原始分辨率
    changed: (i32, i32),   // 修改后的分辨率
}
```

#### SyncDisplaysInfo

```rust
struct SyncDisplaysInfo {
    displays: Vec<DisplayInfo>,  // 显示器列表
    is_synced: bool,             // 是否已同步
}
```

## 5. 返回值类型

### 5.1 主要返回值类型

- `GenericService`: 服务实例
- `ResultType<()>`: 操作结果（成功或错误）
- `ResultType<Vec<Display>>`: 显示器列表或错误
- `ResultType<Vec<DisplayInfo>>`: 显示器信息列表或错误
- `Option<Message>`: 可选的消息
- `Option<DisplayInfo>`: 可选的显示器信息
- `MessageField<Resolution>`: Protobuf 消息字段（分辨率）
- `SimpleCallOnReturn`: RAII 风格的临时忽略对象
- `bool`: 布尔值（支持/不支持）

## 6. 关键算法和业务规则

### 6.1 显示器变化检测算法

**算法流程**：

```
1. 获取当前显示器列表
2. 比较显示器数量
   - 如果不同，标记为变化
3. 遍历每个显示器
   - 比较位置（x, y）
   - 比较大小（width, height）
   - 如果不同，标记为变化
4. 如果有变化，设置 is_synced = false
5. 生成变化消息
```

**时间复杂度**：O(n)，n 为显示器数量

### 6.2 分辨率恢复算法

**算法流程**：

```
1. 遍历所有分辨率记录
2. 对每个记录：
   a. 获取原始分辨率
   b. 调用 change_resolution() 恢复
   c. 记录日志
3. 清空分辨率记录
```

**注意事项**：
- 只在没有客户端连接时调用
- 恢复失败会记录错误日志
- 恢复完成后清空记录

### 6.3 虚拟显示器创建算法（Windows）

**触发条件**：

```
1. 平台已安装
2. 虚拟显示器功能已启用
3. 满足以下任一条件：
   - 无显示器
   - 仅有一个虚拟显示器（尺寸 <= 1024x1024）
   - 无真实分辨率支持
```

**创建流程**：

```
1. 获取当前显示器列表
2. 检查是否需要虚拟显示器
3. 如果需要：
   a. 调用 plug_in_headless()
   b. 重新获取显示器列表
4. 返回显示器列表
```

### 6.4 缩放比例处理算法

**macOS 平台**：

```
scale = d.scale()
```

**Linux 平台**：

```
if is_x11():
    scale = 1.0
else:  // Wayland
    if is_server() && displays.len() > 1:
        scale = d.scale()
    else:
        scale = 1.0
```

**Windows 平台**：

```
scale = 1.0
```

### 6.5 RTT 估算算法（如果有）

本模块不涉及 RTT 估算，相关功能在 `video_qos.rs` 中。

## 7. 与其他模块的交互

### 7.1 与 video_service.rs 的交互

- `video_service.rs` 调用 `check_display_changed()` 检测显示器变化
- `video_service.rs` 调用 `get_original_resolution()` 获取原始分辨率
- `video_service.rs` 调用 `get_sync_displays()` 获取同步显示器信息
- `video_service.rs` 调用 `temp_ignore_displays_changed()` 临时忽略变化

### 7.2 与 scrap 模块的交互

- 使用 `scrap::Display` 获取显示器信息
- 使用 `scrap::CapturerMag::is_supported()` 检查放大镜支持
- 使用 `scrap::is_cursor_embedded()` 检查光标嵌入支持

### 7.3 与平台模块的交互

**Windows 平台**：
- `crate::platform::is_installed()`: 检查是否已安装
- `crate::platform::change_resolution()`: 修改分辨率
- `crate::platform::resolutions()`: 获取支持的分辨率
- `crate::platform::desktop_changed()`: 检查桌面是否变化
- `crate::virtual_display_manager`: 虚拟显示器管理

**Linux 平台**：
- `crate::platform::linux::is_x11()`: 检查是否为 X11
- `crate::platform::change_resolution()`: 修改分辨率
- `super::wayland`: Wayland 相关功能

**macOS 平台**：
- `crate::platform::change_resolution()`: 修改分辨率

### 7.4 与网络模块的交互

- 通过 `GenericService` 发送显示器消息给客户端
- 使用 `Message` 和 `PeerInfo` Protobuf 消息格式

### 7.5 与配置系统的交互

- 读取配置文件中的虚拟显示器设置
- 检查版本号以确定功能支持

## 8. 使用示例

### 8.1 启动显示器服务

```rust
// 启动服务
let service = display_service::new();

// 服务会在后台运行，定期检测显示器变化
```

### 8.2 获取显示器信息

```rust
// 获取所有显示器信息
let displays = display_service::get_sync_displays();

// 获取指定显示器信息
if let Some(display) = display_service::get_display_info(0) {
    println!("Display: {}x{}", display.width, display.height);
}
```

### 8.3 检查显示器变化

```rust
// 检查显示器是否变化
if let Some(display) = display_service::check_display_changed(
    2,  // 显示器数量
    0,  // 显示器索引
    (0, 0, 1920, 1080)  // 位置和大小
) {
    println!("Display changed: {}", display.name);
}
```

### 8.4 修改分辨率

```rust
// 记录分辨率变化
display_service::set_last_changed_resolution(
    "DISPLAY1",
    (1920, 1080),  // 原始分辨率
    (1280, 720)    // 修改后的分辨率
);

// 修改分辨率
crate::platform::change_resolution("DISPLAY1", 1280, 720)?;
```

### 8.5 恢复分辨率

```rust
// 断开连接时恢复分辨率
display_service::restore_resolutions();
```

### 8.6 临时忽略显示器变化

```rust
// 临时忽略显示器变化
{
    let _guard = display_service::temp_ignore_displays_changed();
    // 在此代码块中，显示器变化会被忽略
    // 执行一些可能触发显示器变化的操作
}
// 代码块结束后，恢复检测
```

### 8.7 获取主显示器

```rust
// 获取主显示器索引
let primary_idx = display_service::get_primary();

// 获取主显示器信息
if let Some(display) = display_service::get_display_info(primary_idx) {
    println!("Primary display: {}", display.name);
}
```

## 9. 平台差异说明

### 9.1 Windows 平台

**特性**：
- 支持虚拟显示器（headless 模式）
- 支持放大镜捕获器（隐私模式）
- 支持 amyuni IDD 和 RustDesk IDD
- 自动创建虚拟显示器

**限制**：
- 需要平台安装
- 需要虚拟显示器功能启用

### 9.2 Linux 平台

**特性**：
- 支持 X11 和 Wayland
- Wayland 支持多显示器缩放
- 使用 DBus 调用（避免频繁调用）

**限制**：
- Wayland 不支持显示器切换
- Wayland 需要特殊处理（clear() 调用）

### 9.3 macOS 平台

**特性**：
- 支持显示器缩放
- 使用系统分辨率 API

**限制**：
- 无虚拟显示器支持

### 9.4 Android/iOS 平台

**特性**：
- 移动平台，功能受限

**限制**：
- 不支持分辨率恢复
- 功能简化

## 10. 性能优化要点

### 10.1 避免频繁检测

- 每 300ms 检测一次显示器变化
- 使用 `is_synced` 标志避免重复发送消息

### 10.2 减少系统调用

- 使用静态变量缓存显示器信息
- Linux 平台避免频繁 DBus 调用

### 10.3 懒加载

- 虚拟显示器按需创建
- 只在需要时获取显示器信息

### 10.4 读写锁优化

- `SYNC_DISPLAYS` 使用 `Mutex` 保护
- `CHANGED_RESOLUTIONS` 使用 `RwLock` 保护

## 11. 注意事项

1. **主显示器索引**：`PRIMARY_DISPLAY_IDX` 在初始化后不会更新，即使显示器变化
2. **Linux Wayland**：不支持显示器切换，需要特殊处理
3. **虚拟显示器**：Windows 平台需要平台安装和虚拟显示器功能启用
4. **分辨率恢复**：只在没有客户端连接时调用
5. **临时忽略**：使用 RAII 模式，确保恢复检测
6. **线程安全**：所有静态变量都使用适当的锁保护
7. **版本兼容**：某些功能需要特定版本支持（如隐私模式）

## 12. 故障排查

### 12.1 显示器变化未检测到

**可能原因**：
- `TEMP_IGNORE_DISPLAYS_CHANGED` 标志被设置
- Linux Wayland 平台
- 显示器信息未正确更新

**解决方法**：
- 检查临时忽略标志
- 检查平台类型
- 查看日志输出

### 12.2 虚拟显示器未创建

**可能原因**：
- 平台未安装
- 虚拟显示器功能未启用
- 已有真实显示器

**解决方法**：
- 检查平台安装状态
- 检查虚拟显示器配置
- 查看日志输出

### 12.3 分辨率恢复失败

**可能原因**：
- 显示器名称不匹配
- 分辨率不支持
- 权限不足

**解决方法**：
- 检查显示器名称
- 检查支持的分辨率列表
- 检查权限设置

## 13. 未来改进方向

1. **更智能的虚拟显示器管理**：自动检测和创建虚拟显示器
2. **显示器热插拔优化**：更快速地响应显示器插拔事件
3. **多显示器优化**：改进多显示器场景下的性能
4. **跨平台统一**：统一不同平台的显示器管理逻辑
5. **错误恢复机制**：增加更健壮的错误恢复机制
6. **性能监控**：增加性能监控和优化建议
