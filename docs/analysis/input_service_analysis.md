# input_service.rs 详细分析文档

## 一、功能概述

`input_service.rs` 是 RustDesk 远程桌面服务器的输入服务模块，负责处理和模拟远程控制端的鼠标和键盘输入事件。该模块实现了跨平台（Windows、Linux、macOS）的输入事件处理，包括：

- 鼠标移动、点击、滚轮等事件
- 键盘按键事件（包括特殊键和修饰键）
- 鼠标光标数据同步
- 鼠标位置跟踪
- 窗口焦点检测
- CapsLock 和 NumLock 状态同步
- 触摸事件支持（部分平台）

## 二、核心实现逻辑

### 2.1 架构设计

该模块采用服务模板模式，通过 `ServiceTmpl` 实现三个主要服务：

1. **MouseCursorService**: 鼠标光标数据服务，同步光标形状和样式
2. **MousePosService**: 鼠标位置服务，跟踪和同步鼠标位置
3. **WindowFocusService**: 窗口焦点服务，检测当前活动显示器

### 2.2 状态管理

使用多个状态结构来管理输入状态：

- `StateCursor`: 管理光标句柄和缓存的光标数据
- `StatePos`: 管理鼠标位置
- `StateWindowFocus`: 管理窗口焦点状态

### 2.3 输入模拟

使用 `enigo` 库进行跨平台的输入模拟，并在不同平台上使用特定的实现：

- **Windows/Linux**: 使用 `enigo` 库
- **macOS**: 使用 `rdev` 库的 `VirtualInput`，并配合 `enigo` 用于特定场景
- **Linux Wayland**: 支持 uinput 和 RDP 输入两种方式

### 2.4 修饰键处理

`LockModesHandler` 结构负责处理 CapsLock 和 NumLock 的状态同步，确保远程端和本地端的修饰键状态一致。

## 三、主要函数/方法说明

### 3.1 服务创建函数

#### `new_cursor()`
```rust
pub fn new_cursor() -> ServiceTmpl<MouseCursorSub>
```

**功能**: 创建鼠标光标服务

**返回值**: `ServiceTmpl<MouseCursorSub>` - 光标服务实例

**说明**: 
- 服务每 33ms 运行一次（约 30fps）
- 监控光标变化并同步到客户端
- 使用缓存机制减少数据传输

#### `new_pos()`
```rust
pub fn new_pos() -> GenericService
```

**功能**: 创建鼠标位置服务

**返回值**: `GenericService` - 位置服务实例

**说明**:
- 服务每 33ms 运行一次
- 跟踪系统鼠标位置
- 排除最近 300ms 内的远程输入以避免循环

#### `new_window_focus()`
```rust
pub fn new_window_focus() -> GenericService
```

**功能**: 创建窗口焦点服务

**返回值**: `GenericService` - 焦点服务实例

**说明**:
- 仅在多显示器环境下工作
- 检测当前活动显示器
- 支持跟随当前显示器功能

### 3.2 光标位置记录

#### `try_start_record_cursor_pos()`
```rust
pub fn try_start_record_cursor_pos() -> Option<thread::JoinHandle<()>>
```

**功能**: 启动光标位置记录线程

**返回值**: `Option<thread::JoinHandle<()>>` - 线程句柄，如果已运行则返回 None

**说明**:
- 每 33ms 采样一次系统光标位置
- 使用原子变量确保单实例运行
- 位置存储在 `LATEST_SYS_CURSOR_POS` 全局变量中

#### `try_stop_record_cursor_pos()`
```rust
pub fn try_stop_record_cursor_pos()
```

**功能**: 停止光标位置记录

**说明**:
- 仅在没有远程连接时停止
- 通过设置原子变量标志来停止线程

### 3.3 Linux 平台特定函数

#### `setup_uinput()`
```rust
#[cfg(target_os = "linux")]
pub async fn setup_uinput(minx: i32, maxx: i32, miny: i32, maxy: i32) -> ResultType<()>
```

**功能**: 设置 Linux uinput 设备用于输入模拟

**参数**:
- `minx`, `maxx`: 鼠标 X 轴最小/最大坐标
- `miny`, `maxy`: 鼠标 Y 轴最小/最大坐标

**返回值**: `ResultType<()>` - 成功返回 Ok，失败返回错误

**说明**:
- 创建虚拟键盘和鼠标设备
- 必须在 tokio 运行时中调用
- 需要适当的权限

#### `setup_rdp_input()`
```rust
#[cfg(target_os = "linux")]
pub async fn setup_rdp_input() -> ResultType<(), Box<dyn std::error::Error>>
```

**功能**: 设置 RDP 会话输入

**返回值**: `ResultType<(), Box<dyn std::error::Error>>` - 成功返回 Ok，失败返回错误

**说明**:
- 用于 RDP 会话中的输入模拟
- 使用 RDP 协议的输入通道

#### `update_mouse_resolution()`
```rust
#[cfg(target_os = "linux")]
pub async fn update_mouse_resolution(minx: i32, maxx: i32, miny: i32, maxy: i32) -> ResultType<>
```

**功能**: 更新鼠标分辨率范围

**参数**: 同 `setup_uinput()`

**返回值**: `ResultType<()>` - 成功返回 Ok，失败返回错误

### 3.4 辅助函数

#### `is_left_up()`
```rust
pub fn is_left_up(evt: &MouseEvent) -> bool
```

**功能**: 检查鼠标事件是否为左键释放

**参数**:
- `evt`: 鼠标事件引用

**返回值**: `bool` - 如果是左键释放返回 true

**说明**:
- 通过解析事件掩码判断
- `mask >> 3` 获取按钮信息
- `mask & 0x7` 获取事件类型

#### `update_last_cursor_pos()`
```rust
fn update_last_cursor_pos(x: i32, y: i32)
```

**功能**: 更新最后记录的光标位置

**参数**:
- `x`, `y`: 光标坐标

**说明**:
- 更新全局变量 `LATEST_SYS_CURSOR_POS`
- 记录时间戳和位置

### 3.5 服务运行函数

#### `run_cursor()`
```rust
fn run_cursor(sp: MouseCursorService, state: &mut StateCursor) -> ResultType<>
```

**功能**: 运行光标服务的主循环

**参数**:
- `sp`: 鼠标光标服务实例
- `state`: 光标状态引用

**返回值**: `ResultType<()>` - 成功返回 Ok，失败返回错误

**流程**:
1. 获取当前系统光标句柄
2. 检查光标是否变化
3. 如果变化，获取光标数据并压缩
4. 发送光标数据到所有订阅者
5. 更新快照

#### `run_pos()`
```rust
fn run_pos(sp: EmptyExtraFieldService, state: &mut StatePos) -> ResultType<>
```

**功能**: 运行位置服务的主循环

**参数**:
- `sp`: 空服务实例
- `state`: 位置状态引用

**返回值**: `ResultType<()>` - 成功返回 Ok，失败返回错误

**流程**:
1. 获取最新系统光标位置
2. 检查位置是否移动
3. 如果移动，发送位置更新（排除最近输入的连接）
4. 更新状态和快照

#### `run_window_focus()`
```rust
fn run_window_focus(sp: EmptyExtraFieldService, state: &mut StateWindowFocus) -> ResultType<>
```

**功能**: 运行窗口焦点服务的主循环

**参数**:
- `sp`: 空服务实例
- `state`: 焦点状态引用

**返回值**: `ResultType<()>` - 成功返回 Ok，失败返回错误

**流程**:
1. 获取所有显示器信息
2. 检测当前活动显示器
3. 如果焦点变化，发送焦点更新消息

## 四、数据结构说明

### 4.1 StateCursor
```rust
struct StateCursor {
    hcursor: u64,
    cursor_data: Arc<Message>,
    cached_cursor_data: HashMap<u64, Arc<Message>>,
}
```

**字段说明**:
- `hcursor`: 当前光标句柄
- `cursor_data`: 当前光标数据（压缩后）
- `cached_cursor_data`: 光标数据缓存，避免重复传输

### 4.2 StatePos
```rust
struct StatePos {
    cursor_pos: (i32, i32),
}
```

**字段说明**:
- `cursor_pos`: 光标位置坐标 (x, y)

### 4.3 StateWindowFocus
```rust
struct StateWindowFocus {
    display_idx: i32,
}
```

**字段说明**:
- `display_idx`: 当前活动显示器索引

### 4.4 MouseCursorSub
```rust
pub struct MouseCursorSub {
    inner: ConnInner,
    cached: HashMap<u64, Arc<Message>>,
}
```

**字段说明**:
- `inner`: 内部连接对象
- `cached`: 客户端光标缓存

**说明**: 实现了 `Subscriber` trait，用于优化光标数据传输

### 4.5 LockModesHandler
```rust
#[cfg(any(target_os = "windows", target_os = "linux"))]
struct LockModesHandler {
    caps_lock_changed: bool,
    num_lock_changed: bool,
}
```

**字段说明**:
- `caps_lock_changed`: CapsLock 状态是否改变
- `num_lock_changed`: NumLock 状态是否改变

**说明**: macOS 版本为空结构体，使用不同的实现

## 五、全局变量和常量

### 5.1 常量
```rust
const INVALID_CURSOR_POS: i32 = i32::MIN;
const INVALID_DISPLAY_IDX: i32 = -1;
const KEY_CHAR_START: u64 = 9999;
const MOUSE_MOVE_PROTECTION_TIMEOUT: Duration = Duration::from_millis(1_000);
const MOUSE_ACTIVE_DISTANCE: i32 = 5;
```

### 5.2 全局变量
```rust
static ref ENIGO: Arc<Mutex<Enigo>> = Arc::new(Mutex::new(Enigo::new()));
static ref KEYS_DOWN: Arc<Mutex<HashMap<KeysDown, Instant>>> = Default::default();
static ref LATEST_PEER_INPUT_CURSOR: Arc<Mutex<Input>> = Default::default();
static ref LATEST_SYS_CURSOR_POS: Arc<Mutex<(Option<Instant>, (i32, i32))>> = Default::default();
static EXITING: AtomicBool = AtomicBool::new(false);
static RECORD_CURSOR_POS_RUNNING: AtomicBool = AtomicBool::new(false);
```

**说明**:
- `ENIGO`: 全局输入模拟器实例
- `KEYS_DOWN`: 记录按下的键和时间戳
- `LATEST_PEER_INPUT_CURSOR`: 记录最新的远程输入
- `LATEST_SYS_CURSOR_POS`: 记录最新的系统光标位置
- `EXITING`: 退出标志
- `RECORD_CURSOR_POS_RUNNING`: 光标记录线程运行标志

## 六、关键业务逻辑流程

### 6.1 鼠标输入处理流程

```
远程客户端发送鼠标事件
    ↓
解析事件类型（移动/点击/滚轮）
    ↓
检查是否需要排除（避免循环）
    ↓
使用 ENIGO 模拟输入
    ↓
更新 LATEST_PEER_INPUT_CURSOR
    ↓
记录输入时间和位置
```

### 6.2 键盘输入处理流程

```
远程客户端发送键盘事件
    ↓
解析按键和修饰键
    ↓
创建 LockModesHandler 处理修饰键状态
    ↓
根据平台选择输入模拟方式
    ↓
模拟按键按下/释放
    ↓
LockModesHandler 在 Drop 时恢复修饰键状态
```

### 6.3 光标同步流程

```
系统光标变化
    ↓
run_cursor() 检测到变化
    ↓
获取光标数据
    ↓
压缩光标颜色数据
    ↓
缓存光标数据
    ↓
发送到所有订阅者
    ↓
更新快照供新连接使用
```

### 6.4 位置同步流程

```
系统光标移动
    ↓
try_start_record_cursor_pos() 记录位置
    ↓
run_pos() 检测位置变化
    ↓
排除最近 300ms 的远程输入
    ↓
发送位置更新到其他客户端
    ↓
更新快照
```

## 七、使用示例

### 7.1 创建输入服务

```rust
// 创建光标服务
let cursor_service = new_cursor();

// 创建位置服务
let pos_service = new_pos();

// 创建焦点服务
let focus_service = new_window_focus();

// 启动光标位置记录
let handle = try_start_record_cursor_pos();

// ... 服务运行 ...

// 停止记录
try_stop_record_cursor_pos();
```

### 7.2 Linux uinput 设置

```rust
// 设置 uinput 设备
setup_uinput(0, 1920, 0, 1080).await?;

// 更新鼠标分辨率
update_mouse_resolution(0, 2560, 0, 1440).await?;
```

### 7.3 检测鼠标事件

```rust
fn handle_mouse_event(evt: &MouseEvent) {
    if is_left_up(evt) {
        println!("Left button released");
    }
    
    // 检查事件掩码
    let buttons = evt.mask >> 3;
    let evt_type = evt.mask & 0x7;
    
    match evt_type {
        1 => println!("Mouse down"),
        2 => println!("Mouse up"),
        _ => println!("Other event"),
    }
}
```

## 八、注意事项

### 8.1 平台兼容性

1. **Windows**:
   - 使用 `enigo` 库进行输入模拟
   - 特殊处理 NumLock 状态
   - 支持 legacy 模式

2. **Linux**:
   - 支持 X11 和 Wayland
   - Wayland 下使用 uinput 或 RDP 输入
   - 需要适当的权限访问 `/dev/uinput`

3. **macOS**:
   - 必须在主线程运行键盘输入（>= 10.15）
   - 使用 `rdev` 的 `VirtualInput`
   - 特殊处理 CapsLock 状态
   - legacy 模式有特殊处理（issue #9729）

### 8.2 性能优化

1. **光标缓存**:
   - 使用 `cached_cursor_data` 避免重复传输
   - 客户端也需要实现缓存机制

2. **位置更新频率**:
   - 服务每 33ms 运行一次（约 30fps）
   - 避免过于频繁的更新

3. **压缩**:
   - 光标颜色数据使用压缩算法
   - 减少网络传输量

### 8.3 并发安全

1. **全局变量**:
   - 所有全局变量使用 `Arc<Mutex<>>` 保护
   - 注意避免死锁

2. **线程安全**:
   - 光标记录在独立线程中运行
   - 使用原子变量控制线程生命周期

### 8.4 错误处理

1. **输入模拟失败**:
   - 使用 `allow_err!` 宏忽略非关键错误
   - 记录错误日志

2. **权限问题**:
   - Linux uinput 需要适当的权限
   - macOS 可能需要辅助功能权限

### 8.5 已知问题和限制

1. **macOS legacy 模式**:
   - 物理鼠标移动可能影响虚拟输入
   - 需要特殊处理（issue #9729）

2. **Linux Wayland**:
   - uinput 模式下需要延迟确保锁定状态应用
   - 可能影响用户体验

3. **多显示器**:
   - 窗口焦点检测仅在有多个显示器时工作
   - 需要平台特定的实现

### 8.6 安全考虑

1. **输入验证**:
   - 验证输入事件的合法性
   - 防止恶意输入

2. **权限控制**:
   - 确保只有授权连接可以发送输入
   - 使用 `AUTHED_CONNS` 验证

### 8.7 调试建议

1. **日志记录**:
   - 使用 `log::trace!` 记录详细事件
   - 使用 `log::error!` 记录错误

2. **状态监控**:
   - 监控 `KEYS_DOWN` 状态
   - 检查 `LATEST_PEER_INPUT_CURSOR` 时间戳

3. **性能分析**:
   - 监控服务运行时间
   - 检查缓存命中率

## 九、依赖项

- `enigo`: 跨平台输入模拟
- `rdev`: macOS 输入事件处理
- `hbb_common`: RustDesk 公共库
- `scrap`: 屏幕捕获（Linux Wayland）
- `portable-pty`: 伪终端支持

## 十、相关文件

- `src/server/service.rs`: 服务模板定义
- `src/server/display_service.rs`: 显示器服务
- `src/input.rs`: 输入事件定义
- `src/whiteboard.rs`: 白板功能（非移动平台）
- `src/platform/`: 平台特定实现
