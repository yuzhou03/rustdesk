# privacy_mode.rs 技术分析文档

## 文件概述

`privacy_mode.rs` 是 RustDesk 项目的隐私模式管理模块，负责实现和协调不同的隐私保护实现方式。该模块提供了统一的接口来管理隐私模式的开启、关闭和状态查询，支持多种实现策略，主要用于在远程控制会话中保护用户的隐私。

**文件路径**: `src/privacy_mode.rs`

## 核心功能

### 1. 隐私模式管理
- 开启和关闭隐私模式
- 状态查询和验证
- 连接 ID 管理

### 2. 多种实现策略
- **MAG（Magnifier）**：使用 Windows 放大镜实现
- **ExcludeFromCapture**：排除窗口捕获
- **VirtualDisplay**：使用虚拟显示器实现

### 3. 异步和同步模式
- 支持异步隐私模式（超时 7.5 秒）
- 支持同步隐私模式
- 自动选择合适的模式

### 4. 跨平台支持
- Windows 平台：完整实现
- 其他平台：空实现（返回 None）

## 主要数据结构

### 1. PrivacyModeState 枚举

```rust
#[derive(Debug, Serialize, Deserialize, Clone)]
#[serde(tag = "t", content = "c")]
pub enum PrivacyModeState {
    OffSucceeded,
    OffByPeer,
    OffUnknown,
}
```

**功能说明**：
- `OffSucceeded`：隐私模式成功关闭
- `OffByPeer`：隐私模式被对等端关闭
- `OffUnknown`：隐私模式关闭原因未知

**使用场景**：
- 记录隐私模式关闭状态
- 跨进程传递状态信息
- UI 显示隐私模式状态

### 2. PrivacyMode Trait

```rust
pub trait PrivacyMode: Sync + Send {
    fn is_async_privacy_mode(&self) -> bool;

    fn init(&self) -> ResultType<()>;
    fn clear(&mut self);
    fn turn_on_privacy(&mut self, conn_id: i32) -> ResultType<bool>;
    fn turn_off_privacy(&mut self, conn_id: i32, state: Option<PrivacyModeState>)
        -> ResultType<()>;

    fn pre_conn_id(&self) -> i32;

    fn get_impl_key(&self) -> &str;

    #[inline]
    fn check_on_conn_id(&self, conn_id: i32) -> ResultType<bool>;

    #[inline]
    fn check_off_conn_id(&self, conn_id: i32) -> ResultType<()>;
}
```

**功能说明**：
- `is_async_privacy_mode`: 是否为异步隐私模式
- `init`: 初始化隐私模式实现
- `clear`: 清除隐私模式状态
- `turn_on_privacy`: 开启隐私模式，返回是否已经开启
- `turn_off_privacy`: 关闭隐私模式
- `pre_conn_id`: 获取当前连接 ID
- `get_impl_key`: 获取实现键
- `check_on_conn_id`: 检查连接 ID 是否可以开启隐私模式
- `check_off_conn_id`: 检查连接 ID 是否可以关闭隐私模式

**实现要求**：
- 必须是 `Sync + Send`，支持多线程
- 提供完整的生命周期管理
- 处理连接 ID 冲突

## 主要函数说明

### 1. init 函数

```rust
#[inline]
pub fn init() -> Option<ResultType<()>>
```

**功能**：初始化当前隐私模式实现

**返回值**：`Option<ResultType<()>>` - 返回初始化结果，如果没有隐私模式实现则返回 None

**实现逻辑**：
1. 获取 PRIVACY_MODE 锁
2. 调用当前隐私模式实现的 `init` 方法
3. 返回结果

**使用场景**：
- 程序启动时初始化
- 切换隐私模式实现后初始化

### 2. clear 函数

```rust
#[inline]
pub fn clear() -> Option<()>
```

**功能**：清除当前隐私模式状态

**返回值**：`Option<()>` - 返回清除结果，如果没有隐私模式实现则返回 None

**实现逻辑**：
1. 获取 PRIVACY_MODE 锁
2. 调用当前隐私模式实现的 `clear` 方法
3. 返回结果

**使用场景**：
- 程序退出时清理
- 切换隐私模式实现前清理

### 3. switch 函数

```rust
#[inline]
pub fn switch(impl_key: &str)
```

**功能**：切换到指定的隐私模式实现

**参数**：
- `impl_key: &str` - 隐私模式实现的键

**实现逻辑**：
1. 获取 PRIVACY_MODE 锁
2. 检查当前实现是否已经是目标实现
3. 如果不是，从 PRIVACY_MODE_CREATOR 中获取创建器
4. 创建新的隐私模式实现
5. 替换当前实现

**使用场景**：
- 用户切换隐私模式实现
- 自动回退到默认实现

### 4. turn_on_privacy 函数

```rust
pub async fn turn_on_privacy(impl_key: &str, conn_id: i32) -> Option<ResultType<bool>>
```

**功能**：开启隐私模式

**参数**：
- `impl_key: &str` - 隐私模式实现的键
- `conn_id: i32` - 连接 ID

**返回值**：`Option<ResultType<bool>>` - 返回是否已经开启，如果没有隐私模式实现则返回 None

**实现逻辑**：
1. 检查是否为异步隐私模式
2. 如果是异步，调用 `turn_on_privacy_async`
3. 如果是同步，调用 `turn_on_privacy_sync`

**使用场景**：
- 远程控制会话开始时开启隐私模式
- 用户手动开启隐私模式

### 5. turn_on_privacy_async 函数

```rust
async fn turn_on_privacy_async(impl_key: String, conn_id: i32) -> Option<ResultType<bool>>
```

**功能**：异步开启隐私模式（超时 7.5 秒）

**参数**：
- `impl_key: String` - 隐私模式实现的键
- `conn_id: i32` - 连接 ID

**返回值**：`Option<ResultType<bool>>` - 返回是否已经开启

**实现逻辑**：
1. 创建 oneshot 通道
2. 在新线程中调用 `turn_on_privacy_sync`
3. 等待结果（最多 7.5 秒）
4. 返回结果或超时错误

**使用场景**：
- 需要长时间操作的隐私模式实现
- 避免阻塞主线程

**注意事项**：
- 超时时间设置为 7.5 秒，因为某些实现（如 Amyuni IDD）可能需要较长时间
- 某些笔记本电脑插入虚拟显示器也需要时间

### 6. turn_on_privacy_sync 函数

```rust
fn turn_on_privacy_sync(impl_key: &str, conn_id: i32) -> Option<ResultType<bool>>
```

**功能**：同步开启隐私模式

**参数**：
- `impl_key: &str` - 隐私模式实现的键
- `conn_id: i32` - 连接 ID

**返回值**：`Option<ResultType<bool>>` - 返回是否已经开启

**实现逻辑**：
1. 获取 PRIVACY_MODE 锁
2. 检查或切换隐私模式实现（`get_supported_impl`）
3. 检查连接 ID（`check_on_conn_id`）
4. 如果连接 ID 相同且实现相同，返回 true
5. 如果连接 ID 相同但实现不同，切换实现
6. 如果连接 ID 不同，检查是否被占用
7. 如果实现不同，切换实现
8. 调用 `turn_on_privacy` 开启隐私模式

**使用场景**：
- 同步开启隐私模式
- 处理连接 ID 冲突

### 7. turn_off_privacy 函数

```rust
#[inline]
pub fn turn_off_privacy(conn_id: i32, state: Option<PrivacyModeState>) -> Option<ResultType<()>>
```

**功能**：关闭隐私模式

**参数**：
- `conn_id: i32` - 连接 ID
- `state: Option<PrivacyModeState>` - 隐私模式状态

**返回值**：`Option<ResultType<()>>` - 返回关闭结果

**实现逻辑**：
1. 获取 PRIVACY_MODE 锁
2. 调用当前隐私模式实现的 `turn_off_privacy` 方法
3. 返回结果

**使用场景**：
- 远程控制会话结束时关闭隐私模式
- 用户手动关闭隐私模式

### 8. check_on_conn_id 函数

```rust
#[inline]
pub fn check_on_conn_id(conn_id: i32) -> Option<ResultType<bool>>
```

**功能**：检查连接 ID 是否可以开启隐私模式

**参数**：
- `conn_id: i32` - 连接 ID

**返回值**：`Option<ResultType<bool>>` - 返回是否可以开启

**实现逻辑**：
1. 获取 PRIVACY_MODE 锁
2. 调用当前隐私模式实现的 `check_on_conn_id` 方法
3. 返回结果

**使用场景**：
- 开启隐私模式前检查
- 防止连接 ID 冲突

### 9. get_supported_privacy_mode_impl 函数

```rust
pub fn get_supported_privacy_mode_impl() -> Vec<(&'static str, &'static str)>
```

**功能**：获取支持的隐私模式实现列表

**返回值**：`Vec<(&'static str, &'static str)>` - 返回实现键和提示文本的列表

**实现逻辑**：
- **Windows 平台**：
  1. 如果 `win_exclude_from_capture::is_supported()`，添加 ExcludeFromCapture 实现
  2. 否则，如果 `display_service::is_privacy_mode_mag_supported()`，添加 MAG 实现
  3. 如果已安装且自服务正在运行，添加 VirtualDisplay 实现
- **其他平台**：返回空列表

**使用场景**：
- UI 显示可用的隐私模式实现
- 自动选择默认实现

### 10. is_privacy_mode_supported 函数

```rust
#[inline]
pub fn is_privacy_mode_supported() -> bool
```

**功能**：检查是否支持隐私模式

**返回值**：`bool` - 返回是否支持

**实现逻辑**：
- 检查 `DEFAULT_PRIVACY_MODE_IMPL` 是否为空

**使用场景**：
- UI 显示/隐藏隐私模式选项
- 功能开关控制

### 11. get_privacy_mode_conn_id 函数

```rust
#[inline]
pub fn get_privacy_mode_conn_id() -> Option<i32>
```

**功能**：获取当前隐私模式的连接 ID

**返回值**：`Option<i32>` - 返回连接 ID

**使用场景**：
- 查询当前连接
- 状态显示

### 12. is_in_privacy_mode 函数

```rust
#[inline]
pub fn is_in_privacy_mode() -> bool
```

**功能**：检查是否在隐私模式中

**返回值**：`bool` - 返回是否在隐私模式中

**实现逻辑**：
- 检查 `pre_conn_id()` 是否不等于 `INVALID_PRIVACY_MODE_CONN_ID`

**使用场景**：
- 状态查询
- UI 显示

## 常量定义

### 1. INVALID_PRIVACY_MODE_CONN_ID
```rust
pub const INVALID_PRIVACY_MODE_CONN_ID: i32 = 0;
```
无效的隐私模式连接 ID，表示没有隐私模式激活。

### 2. OCCUPIED
```rust
pub const OCCUPIED: &'static str = "Privacy occupied by another one.";
```
隐私模式被其他连接占用的错误消息。

### 3. TURN_OFF_OTHER_ID
```rust
pub const TURN_OFF_OTHER_ID: &'static str =
    "Failed to turn off privacy mode that belongs to someone else.";
```
无法关闭属于其他连接的隐私模式的错误消息。

### 4. NO_PHYSICAL_DISPLAYS
```rust
pub const NO_PHYSICAL_DISPLAYS: &'static str = "no_need_privacy_mode_no_physical_displays_tip";
```
没有物理显示器时不需要隐私模式的提示消息。

### 5. 隐私模式实现键
```rust
pub const PRIVACY_MODE_IMPL_WIN_MAG: &str = "privacy_mode_impl_mag";
pub const PRIVACY_MODE_IMPL_WIN_EXCLUDE_FROM_CAPTURE: &str =
    "privacy_mode_impl_exclude_from_capture";
pub const PRIVACY_MODE_IMPL_WIN_VIRTUAL_DISPLAY: &str = "privacy_mode_impl_virtual_display";
```
不同隐私模式实现的键。

## 静态变量

### 1. DEFAULT_PRIVACY_MODE_IMPL
```rust
lazy_static::lazy_static! {
    pub static ref DEFAULT_PRIVACY_MODE_IMPL: String = {
        #[cfg(windows)]
        {
            if win_exclude_from_capture::is_supported() {
                PRIVACY_MODE_IMPL_WIN_EXCLUDE_FROM_CAPTURE
            } else {
                if display_service::is_privacy_mode_mag_supported() {
                    PRIVACY_MODE_IMPL_WIN_MAG
                } else {
                    if is_installed() {
                        PRIVACY_MODE_IMPL_WIN_VIRTUAL_DISPLAY
                    } else {
                        ""
                    }
                }
            }.to_owned()
        }
        #[cfg(not(windows))]
        {
            "".to_owned()
        }
    };
}
```
默认隐私模式实现，根据平台和系统支持情况自动选择。

### 2. PRIVACY_MODE
```rust
lazy_static::lazy_static! {
    static ref PRIVACY_MODE: Arc<Mutex<Option<Box<dyn PrivacyMode>>>> = {
        let mut cur_impl = get_option("privacy-mode-impl-key".to_owned());
        if !get_supported_privacy_mode_impl().iter().any(|(k, _)| k == &cur_impl) {
            cur_impl = DEFAULT_PRIVACY_MODE_IMPL.to_owned();
        }

        let privacy_mode = match PRIVACY_MODE_CREATOR.lock().unwrap().get(&(&cur_impl as &str)) {
            Some(creator) => Some(creator(&cur_impl)),
            None => None,
        };
        Arc::new(Mutex::new(privacy_mode))
    };
}
```
当前隐私模式实现，支持动态切换。

### 3. PRIVACY_MODE_CREATOR
```rust
lazy_static::lazy_static! {
    static ref PRIVACY_MODE_CREATOR: Arc<Mutex<HashMap<&'static str, PrivacyModeCreator>>> = {
        #[cfg(not(windows))]
        let map: HashMap<&'static str, PrivacyModeCreator> = HashMap::new();
        #[cfg(windows)]
        let mut map: HashMap<&'static str, PrivacyModeCreator> = HashMap::new();
        #[cfg(windows)]
        {
            if win_exclude_from_capture::is_supported() {
                map.insert(win_exclude_from_capture::PRIVACY_MODE_IMPL, |impl_key: &str| {
                    Box::new(win_exclude_from_capture::PrivacyModeImpl::new(impl_key))
                });
            } else {
                map.insert(win_mag::PRIVACY_MODE_IMPL, |impl_key: &str| {
                    Box::new(win_mag::PrivacyModeImpl::new(impl_key))
                });
            }

            map.insert(win_virtual_display::PRIVACY_MODE_IMPL, |impl_key: &str| {
                    Box::new(win_virtual_display::PrivacyModeImpl::new(impl_key))
                });
        }
        Arc::new(Mutex::map))
    };
}
```
隐私模式实现创建器映射表，支持动态创建。

## 关键业务逻辑流程

### 1. 隐私模式初始化流程

```
程序启动
  ↓
init() 被调用
  ↓
获取 PRIVACY_MODE 锁
  ↓
调用当前实现的 init()
  ↓
初始化成功/失败
```

### 2. 开启隐私模式流程

```
turn_on_privacy(impl_key, conn_id)
  ↓
检查是否为异步模式
  ↓
如果是异步
  ↓
创建 oneshot 通道
  ↓
在新线程中调用 turn_on_privacy_sync
  ↓
等待结果（最多 7.5 秒）
  ↓
返回结果
  ↓
如果是同步
  ↓
直接调用 turn_on_privacy_sync
  ↓
检查或切换实现
  ↓
检查连接 ID
  ↓
调用 turn_on_privacy
  ↓
返回结果
```

### 3. 关闭隐私模式流程

```
turn_off_privacy(conn_id, state)
  ↓
获取 PRIVACY_MODE 锁
  ↓
调用当前实现的 turn_off_privacy
  ↓
检查连接 ID
  ↓
关闭隐私模式
  ↓
返回结果
```

### 4. 切换隐私模式实现流程

```
switch(impl_key)
  ↓
获取 PRIVACY_MODE 锁
  ↓
检查当前实现是否已经是目标实现
  ↓
如果不是
  ↓
从 PRIVACY_MODE_CREATOR 获取创建器
  ↓
创建新的隐私模式实现
  ↓
替换当前实现
```

## 使用场景

### 1. 远程控制会话
- 会话开始时开启隐私模式
- 会话结束时关闭隐私模式
- 防止远程端看到敏感信息

### 2. 隐私保护
- 屏幕内容保护
- 窗口捕获排除
- 虚拟显示器隔离

### 3. 多连接管理
- 连接 ID 管理
- 防止连接冲突
- 状态同步

### 4. 自动回退
- 实现不支持时自动回退
- 根据系统环境选择最佳实现
- 用户体验优化

## 注意事项

### 1. 平台限制
- 仅 Windows 平台有完整实现
- 其他平台返回 None
- 需要平台特定的依赖

### 2. 连接 ID 管理
- 使用 INVALID_PRIVACY_MODE_CONN_ID 表示无激活连接
- 防止连接 ID 冲突
- 正确处理连接切换

### 3. 异步模式
- 超时时间设置为 7.5 秒
- 某些实现可能需要较长时间
- 需要正确处理超时

### 4. 实现选择
- 根据系统支持情况自动选择
- ExcludeFromCapture 优先级最高
- MAG 和 VirtualDisplay 作为备选

### 5. 错误处理
- 使用 ResultType 处理错误
- 提供清晰的错误消息
- 支持自动回退

### 6. 线程安全
- 使用 Arc<Mutex<>> 保护共享状态
- PrivacyMode trait 要求 Sync + Send
- 正确处理并发访问

## 依赖关系

### 外部依赖
- `serde`: 序列化/反序列化
- `lazy_static`: 静态变量延迟初始化
- `hbb_common`: 通用工具库
  - `anyhow`: 错误处理
  - `tokio`: 异步运行时

### 内部依赖
- `crate::ui_interface::get_option`: 获取配置选项
- `crate::display_service`: 显示服务
- `crate::platform`: 平台相关功能
- `crate::ipc`: IPC 通信
- `crate::video_service`: 视频服务

### Windows 特定模块
- `win_exclude_from_capture`: 排除窗口捕获
- `win_mag`: 放大镜实现
- `win_virtual_display`: 虚拟显示器
- `win_input`: 输入处理
- `win_topmost_window`: 置顶窗口

## 总结

`privacy_mode.rs` 是一个功能完善的隐私模式管理模块，提供了统一的接口来管理不同的隐私保护实现。该模块的设计遵循了以下原则：

1. **抽象化**：通过 PrivacyMode trait 提供统一接口
2. **可扩展**：支持多种实现方式，易于添加新实现
3. **平台适配**：根据平台和系统环境自动选择最佳实现
4. **线程安全**：使用 Arc<Mutex<>> 保护共享状态
5. **异步支持**：支持异步和同步模式，避免阻塞
6. **错误处理**：完善的错误处理和自动回退机制

该模块是 RustDesk 隐私保护功能的核心，为用户提供了灵活的隐私保护解决方案，确保在远程控制会话中保护用户的敏感信息。
