# terminal_service.rs 详细分析文档

## 一、功能概述

`terminal_service.rs` 是 RustDesk 远程桌面服务器的终端服务模块，提供远程终端访问功能。该模块实现了：

- 多终端会话管理
- 持久化终端服务（可跨越连接断开）
- 终端输入输出处理
- 伪终端（PTY）管理
- 终端缓冲和压缩
- 终端进程生命周期管理
- 僵尸进程清理
- 跨平台 shell 支持（Windows PowerShell/CMD，Linux/macOS bash/zsh/sh）

## 二、核心实现逻辑

### 2.1 架构设计

该模块采用分层架构：

1. **服务层**:
   - `TerminalService`: 终端服务包装器
   - `PersistentTerminalService`: 持久化终端服务管理器

2. **会话层**:
   - `TerminalSession`: 单个终端会话
   - `OutputBuffer`: 终端输出缓冲区

3. **全局管理层**:
   - `TERMINAL_SERVICES`: 全局终端服务注册表
   - `TERMINAL_TASKS`: 僵尸进程列表
   - `CLEANUP_TASK`: 清理任务线程

### 2.2 持久化机制

终端服务支持持久化，即：
- 连接断开后终端会话仍然保持
- 重新连接时可以恢复终端状态
- 支持多个客户端共享同一终端服务

### 2.3 输入输出处理

- **输入**: 通过通道将客户端输入发送到 writer 线程
- **输出**: reader 线程从 PTY 读取输出，缓冲后发送给客户端
- **缓冲**: 使用 `OutputBuffer` 管理终端输出，限制大小和行数

### 2.4 进程管理

- 使用 `portable_pty` 创建伪终端
- 管理子进程生命周期
- 自动清理僵尸进程

## 三、主要函数/方法说明

### 3.1 服务管理函数

#### `new()`
```rust
pub fn new(
    service_id: String,
    is_persistent: bool,
    user_token: Option<UserToken>,
) -> GenericService
```

**功能**: 创建新的终端服务

**参数**:
- `service_id`: 服务唯一标识符
- `is_persistent`: 是否为持久化服务
- `user_token`: 用户令牌（可选）

**返回值**: `GenericService` - 通用服务实例

**说明**:
- 创建或获取持久化终端服务
- 启动服务运行循环
- 每 30ms 读取一次终端输出（约 33fps）

#### `get_service_name()`
```rust
pub fn get_service_name(source: VideoSource, idx: usize) -> String
```

**功能**: 生成服务名称

**参数**:
- `source`: 视频源类型
- `idx`: 索引

**返回值**: `String` - 服务名称

#### `generate_service_id()`
```rust
pub fn generate_service_id() -> String
```

**功能**: 生成新的持久化服务 ID

**返回值**: `String` - 格式为 "ts_{uuid}" 的服务 ID

#### `get_service()`
```rust
pub fn get_service(service_id: &str) -> Option<Arc<Mutex<PersistentTerminalService>>>
```

**功能**: 根据 ID 获取终端服务

**参数**:
- `service_id`: 服务 ID

**返回值**: `Option<Arc<Mutex<PersistentTerminalService>>>` - 服务实例或 None

#### `list_services()`
```rust
pub fn list_services() -> Vec<ServiceMetadata>
```

**功能**: 列出所有活动的终端服务

**返回值**: `Vec<ServiceMetadata>` - 服务元数据列表

#### `is_service_specified_user()`
```rust
pub fn is_service_specified_user(service_id: &str) -> Option<bool>
```

**功能**: 检查服务是否为指定用户

**参数**:
- `service_id`: 服务 ID

**返回值**: `Option<bool>` - 是否为指定用户

### 3.2 持久化服务方法

#### `PersistentTerminalService::new()`
```rust
pub fn new(service_id: String, is_persistent: bool, is_specified_user: bool) -> Self
```

**功能**: 创建新的持久化终端服务

**参数**:
- `service_id`: 服务 ID
- `is_persistent`: 是否持久化
- `is_specified_user`: 是否为指定用户

**返回值**: `PersistentTerminalService` - 服务实例

#### `list_terminals()`
```rust
pub fn list_terminals(&self) -> Vec<(i32, String, u32, Instant)>
```

**功能**: 列出服务中的所有终端

**返回值**: `Vec<(i32, String, u32, Instant)>` - 终端列表，包含 (ID, 标题, PID, 创建时间)

#### `get_terminal_buffer()`
```rust
pub fn get_terminal_buffer(&self, terminal_id: i32, max_bytes: usize) -> Option<Vec<u8>>
```

**功能**: 获取终端的缓冲输出

**参数**:
- `terminal_id`: 终端 ID
- `max_bytes`: 最大字节数

**返回值**: `Option<Vec<u8>>` - 缓冲数据或 None

#### `get_terminal_info()`
```rust
pub fn get_terminal_info(&self, terminal_id: i32) -> Option<(u16, u16, Vec<u8>)>
```

**功能**: 获取终端信息用于恢复

**参数**:
- `terminal_id`: 终端 ID

**返回值**: `Option<(u16, u16, Vec<u8>)>` - (行数, 列数, 缓冲数据)

#### `has_active_terminals()`
```rust
pub fn has_active_terminals(&self) -> bool
```

**功能**: 检查服务是否有活动终端

**返回值**: `bool` - 是否有活动终端

### 3.3 终端会话方法

#### `TerminalSession::new()`
```rust
fn new(terminal_id: i32, rows: u16, cols: u16) -> Self
```

**功能**: 创建新的终端会话

**参数**:
- `terminal_id`: 终端 ID
- `rows`: 行数
- `cols`: 列数

**返回值**: `TerminalSession` - 会话实例

#### `stop()`
```rust
fn stop(&mut self)
```

**功能**: 停止终端会话

**说明**:
- 设置退出标志
- 关闭输入通道
- 等待读写线程结束
- 终止子进程
- 将子进程添加到僵尸进程列表

### 3.4 输出缓冲方法

#### `OutputBuffer::new()`
```rust
fn new() -> Self
```

**功能**: 创建新的输出缓冲区

**返回值**: `OutputBuffer` - 缓冲区实例

#### `append()`
```rust
fn append(&mut self, data: &[u8])
```

**功能**: 追加数据到缓冲区

**参数**:
- `data`: 要追加的数据

**说明**:
- 处理不完整的行
- 按换行符分割数据
- 限制缓冲区大小（1MB）和行数（10000）
- 自动修剪旧数据

#### `get_recent()`
```rust
fn get_recent(&self, max_bytes: usize) -> Vec<u8>
```

**功能**: 获取最近的输出数据

**参数**:
- `max_bytes`: 最大字节数

**返回值**: `Vec<u8>` - 最近的输出数据

### 3.5 清理函数

#### `cleanup_inactive_services()`
```rust
pub fn cleanup_inactive_services()
```

**功能**: 清理不活动的服务

**说明**:
- 非持久化服务：1 小时无活动后清理
- 持久化服务：无终端且 2 小时无活动后清理

#### `check_zombie_terminals()`
```rust
fn check_zombie_terminals()
```

**功能**: 检查并清理僵尸终端进程

**说明**:
- 每 100ms 运行一次
- 检查子进程状态
- 移除已退出的进程

#### `ensure_cleanup_task()`
```rust
fn ensure_cleanup_task()
```

**功能**: 确保清理任务正在运行

**说明**:
- 如果清理任务未运行，则启动它
- 清理任务每 100ms 检查僵尸进程
- 每 5 分钟清理不活动的服务

### 3.6 辅助函数

#### `get_default_shell()`
```rust
fn get_default_shell() -> String
```

**功能**: 获取默认 shell 路径

**返回值**: `String` - shell 路径

**说明**:
- **Windows**: 优先 PowerShell Core，其次 Windows PowerShell，最后 cmd.exe
- **Linux/macOS**: 优先 SHELL 环境变量，其次 bash/zsh/sh

#### `get_terminal_session_count()`
```rust
#[cfg(target_os = "linux")]
pub fn get_terminal_session_count(include_zombie_tasks: bool) -> usize
```

**功能**: 获取终端会话数量

**参数**:
- `include_zombie_tasks`: 是否包含僵尸任务

**返回值**: `usize` - 会话数量

## 四、数据结构说明

### 4.1 ServiceMetadata
```rust
pub struct ServiceMetadata {
    pub service_id: String,
    pub created_at: Instant,
    pub terminal_count: usize,
    pub is_persistent: bool,
}
```

**字段说明**:
- `service_id`: 服务 ID
- `created_at`: 创建时间
- `terminal_count`: 终端数量
- `is_persistent`: 是否持久化

### 4.2 TerminalService
```rust
pub struct TerminalService {
    sp: GenericService,
    user_token: Option<UserToken>,
}
```

**字段说明**:
- `sp`: 通用服务实例
- `user_token`: 用户令牌

### 4.3 TerminalSession
```rust
pub struct TerminalSession {
    pub created_at: Instant,
    last_activity: Instant,
    pty_pair: Option<portable_pty::PtyPair>,
    child: Option<Box<dyn Child + std::marker::Send + Sync>>,
    input_tx: Option<SyncSender<Vec<u8>>>,
    output_rx: Option<Receiver<Vec<u8>>>,
    exiting: Arc<AtomicBool>,
    reader_thread: Option<thread::JoinHandle<()>>,
    writer_thread: Option<thread::JoinHandle<()>>,
    output_buffer: OutputBuffer,
    title: String,
    pid: u32,
    rows: u16,
    cols: u16,
    closed_message_sent: bool,
    is_opened: bool,
}
```

**字段说明**:
- `created_at`: 创建时间
- `last_activity`: 最后活动时间
- `pty_pair`: 伪终端对
- `child`: 子进程
- `input_tx`: 输入通道发送端
- `output_rx`: 输出通道接收端
- `exiting`: 退出标志
- `reader_thread`: 读取线程句柄
- `writer_thread`: 写入线程句柄
- `output_buffer`: 输出缓冲区
- `title`: 终端标题
- `pid`: 进程 ID
- `rows`: 行数
- `cols`: 列数
- `closed_message_sent`: 是否已发送关闭消息
- `is_opened`: 是否已打开

### 4.4 PersistentTerminalService
```rust
pub struct PersistentTerminalService {
    service_id: String,
    sessions: HashMap<i32, Arc<Mutex<TerminalSession>>>,
    pub created_at: Instant,
    last_activity: Instant,
    pub is_persistent: bool,
    needs_session_sync: bool,
    is_specified_user: bool,
}
```

**字段说明**:
- `service_id`: 服务 ID
- `sessions`: 终端会话映射
- `created_at`: 创建时间
- `last_activity`: 最后活动时间
- `is_persistent`: 是否持久化
- `needs_session_sync`: 是否需要会话同步
- `is_specified_user`: 是否为指定用户

### 4.5 OutputBuffer
```rust
struct OutputBuffer {
    lines: VecDeque<Vec<u8>>,
    total_size: usize,
    last_line_incomplete: bool,
}
```

**字段说明**:
- `lines`: 行数据队列
- `total_size`: 总大小
- `last_line_incomplete`: 最后一行是否不完整

## 五、全局变量和常量

### 5.1 常量
```rust
const MAX_OUTPUT_BUFFER_SIZE: usize = 1024 * 1024; // 1MB per terminal
const MAX_BUFFER_LINES: usize = 10000;
const MAX_SERVICES: usize = 100;
const SERVICE_IDLE_TIMEOUT: Duration = Duration::from_secs(3600); // 1 hour
const CHANNEL_BUFFER_SIZE: usize = 100;
const COMPRESS_THRESHOLD: usize = 512;
```

### 5.2 全局变量
```rust
static ref TERMINAL_SERVICES: Arc<Mutex<HashMap<String, Arc<Mutex<PersistentTerminalService>>>>>
static ref CLEANUP_TASK: Arc<Mutex<Option<std::thread::JoinHandle<()>>>>
static ref TERMINAL_TASKS: Arc<Mutex<Vec<Box<dyn Child + Send + Sync>>>>
```

**说明**:
- `TERMINAL_SERVICES`: 全局终端服务注册表
- `CLEANUP_TASK`: 清理任务句柄
- `TERMINAL_TASKS`: 僵尸进程列表

## 六、关键业务逻辑流程

### 6.1 创建终端服务流程

```
调用 new(service_id, is_persistent, user_token)
    ↓
get_or_create_service() 获取或创建持久化服务
    ↓
创建 TerminalService 包装器
    ↓
启动服务运行循环
    ↓
run() 函数每 30ms 读取终端输出
    ↓
发送输出到客户端
```

### 6.2 创建终端会话流程

```
客户端请求创建终端
    ↓
创建 TerminalSession 实例
    ↓
使用 portable_pty 创建伪终端
    ↓
启动子进程（shell）
    ↓
创建输入输出通道
    ↓
启动 reader 线程（读取 PTY 输出）
    ↓
启动 writer 线程（写入客户端输入）
    ↓
会话添加到服务中
```

### 6.3 输入处理流程

```
客户端发送输入数据
    ↓
TerminalServiceProxy 接收输入
    ↓
通过 input_tx 通道发送到 writer 线程
    ↓
writer 线程写入 PTY
    ↓
子进程处理输入
    ↓
输出写入 PTY
```

### 6.4 输出处理流程

```
子进程输出到 PTY
    ↓
reader 线程从 PTY 读取
    ↓
通过 output_rx 通道发送
    ↓
OutputBuffer 缓冲输出
    ↓
压缩数据（如果超过阈值）
    ↓
发送到客户端
```

### 6.5 会话停止流程

```
调用 stop()
    ↓
设置 exiting 标志
    ↓
发送最后的换行符到输入通道
    ↓
关闭输入通道
    ↓
关闭 PTY（Windows/Linux）
    ↓
等待 reader 线程结束
    ↓
等待 writer 线程结束
    ↓
终止子进程
    ↓
添加子进程到僵尸列表
```

### 6.6 清理流程

```
清理任务运行（每 100ms）
    ↓
check_zombie_terminals()
    ↓
检查子进程状态
    ↓
移除已退出的进程
    ↓
每 5 分钟
    ↓
cleanup_inactive_services()
    ↓
检查服务活动状态
    ↓
移除不活动的服务
```

## 七、使用示例

### 7.1 创建终端服务

```rust
// 创建持久化终端服务
let service_id = generate_service_id();
let service = new(service_id.clone(), true, None);

// 创建非持久化服务
let temp_service = new("temp_terminal".to_string(), false, Some(user_token));
```

### 7.2 列出服务

```rust
// 列出所有活动服务
let services = list_services();
for svc in services {
    println!("Service: {}, Terminals: {}, Persistent: {}",
        svc.service_id, svc.terminal_count, svc.is_persistent);
}
```

### 7.3 获取服务信息

```rust
// 获取特定服务
if let Some(service) = get_service("service_id") {
    let svc = service.lock().unwrap();
    
    // 列出终端
    let terminals = svc.list_terminals();
    for (id, title, pid, created) in terminals {
        println!("Terminal {}: {} (PID: {})", id, title, pid);
    }
    
    // 获取终端缓冲
    if let Some(buffer) = svc.get_terminal_buffer(0, 4096) {
        println!("Buffer: {:?}", String::from_utf8_lossy(&buffer));
    }
}
```

### 7.4 检查服务状态

```rust
// 检查是否为指定用户服务
if let Some(is_specified) = is_service_specified_user("service_id") {
    println!("Is specified user: {}", is_specified);
}

// 检查是否有活动终端
if let Some(service) = get_service("service_id") {
    let svc = service.lock().unwrap();
    if svc.has_active_terminals() {
        println!("Service has active terminals");
    }
}
```

### 7.5 获取会话统计（Linux）

```rust
#[cfg(target_os = "linux")]
{
    let total_sessions = get_terminal_session_count(true);
    println!("Total terminal sessions: {}", total_sessions);
}
```

## 八、注意事项

### 8.1 平台兼容性

1. **Windows**:
   - 使用 ConPTY（Console Pseudo Terminal）
   - 支持 PowerShell 和 CMD
   - reader 线程可能在 pipe 读取时阻塞
   - 需要发送最后的 "\r\n" 来解除阻塞

2. **Linux**:
   - 使用 `libc::openpty`
   - 支持 bash、zsh、sh 等
   - reader 线程在关闭 PTY 后会收到 [13, 10]（\r\n）

3. **macOS**:
   - 使用 POSIX PTY
   - 目前未发现阻塞问题
   - 可能需要更多测试

### 8.2 资源管理

1. **缓冲区限制**:
   - 每个终端最大 1MB 缓冲
   - 最多 10000 行
   - 自动修剪旧数据

2. **服务限制**:
   - 最多 100 个持久化服务
   - 超过限制返回错误

3. **进程管理**:
   - 自动清理僵尸进程
   - 子进程在会话停止时终止
   - 使用 `portable_pty` 管理 PTY

### 8.3 并发安全

1. **全局变量**:
   - 所有全局变量使用 `Arc<Mutex<>>` 保护
   - 注意避免死锁

2. **线程安全**:
   - reader 和 writer 线程独立运行
   - 使用通道进行线程间通信
   - 使用原子变量控制退出

3. **会话同步**:
   - 每个会话有自己的互斥锁
   - 避免长时间持有锁

### 8.4 性能优化

1. **输出频率**:
   - 每 30ms 读取一次输出（约 33fps）
   - 平衡响应性和性能

2. **数据压缩**:
   - 超过 512 字节的数据进行压缩
   - 减少网络传输量

3. **缓冲策略**:
   - 使用 `VecDeque` 高效管理行数据
   - 自动修剪旧数据

### 8.5 错误处理

1. **PTY 创建失败**:
   - 使用 `allow_err!` 宏忽略非关键错误
   - 记录错误日志

2. **进程终止**:
   - 使用 `child.kill()` 强制终止
   - 添加到僵尸列表等待清理

3. **通道错误**:
   - 通道关闭时优雅处理
   - 记录警告日志

### 8.6 持久化注意事项

1. **会话恢复**:
   - 重新连接时可以恢复终端状态
   - 使用 `get_terminal_info()` 获取会话信息

2. **状态同步**:
   - `needs_session_sync` 标志指示需要同步
   - `is_opened` 标志跟踪会话状态

3. **清理策略**:
   - 非持久化服务：1 小时无活动后清理
   - 持久化服务：无终端且 2 小时无活动后清理

### 8.7 安全考虑

1. **用户令牌**:
   - 使用 `user_token` 验证用户身份
   - `is_specified_user` 标志跟踪用户类型

2. **进程隔离**:
   - 每个终端在独立的 PTY 中运行
   - 限制缓冲区大小防止内存耗尽

3. **权限控制**:
   - 确保只有授权用户可以访问终端
   - 验证服务 ID 和用户令牌

### 8.8 调试建议

1. **日志记录**:
   - 使用 `log::info!` 记录重要事件
   - 使用 `log::warn!` 记录警告
   - 使用 `log::error!` 记录错误

2. **状态监控**:
   - 监控终端会话数量
   - 检查缓冲区大小
   - 跟踪进程状态

3. **性能分析**:
   - 监控读写线程性能
   - 检查通道缓冲区使用情况
   - 分析压缩效果

### 8.9 已知问题和限制

1. **Windows PTY**:
   - reader 线程可能在 pipe 读取时阻塞
   - 需要特殊处理来解除阻塞

2. **Linux PTY**:
   - 关闭 PTY 后可能收到额外的数据
   - 需要正确处理

3. **macOS PTY**:
   - 可能存在未发现的阻塞问题
   - 需要更多测试

4. **缓冲区限制**:
   - 大量输出可能导致数据丢失
   - 客户端需要处理这种情况

### 8.10 最佳实践

1. **资源清理**:
   - 始终在适当的时候停止会话
   - 依赖 Drop trait 自动清理

2. **错误恢复**:
   - 实现重连机制
   - 使用持久化服务保持会话

3. **性能监控**:
   - 定期检查缓冲区使用情况
   - 监控进程资源使用

4. **日志记录**:
   - 记录关键事件和错误
   - 使用适当的日志级别

## 九、依赖项

- `portable-pty`: 跨平台伪终端支持
- `hbb_common`: RustDesk 公共库
- `uuid`: UUID 生成
- `anyhow`: 错误处理

## 十、相关文件

- `src/server/service.rs`: 服务模板定义
- `src/video_service.rs`: 视频服务
- `src/server/clipboard_service.rs`: 剪贴板服务
- `src/lib.rs`: 库入口

## 十一、扩展建议

1. **功能增强**:
   - 支持终端颜色和样式
   - 实现终端大小调整
   - 添加终端搜索功能

2. **性能优化**:
   - 使用更高效的压缩算法
   - 实现增量更新
   - 优化缓冲区管理

3. **安全增强**:
   - 添加命令白名单
   - 实现审计日志
   - 支持用户权限管理

4. **用户体验**:
   - 添加终端历史记录
   - 支持多标签页
   - 实现终端会话恢复
