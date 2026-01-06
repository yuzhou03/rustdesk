# hbbs_http 模块技术文档

## 目录结构说明

```
src/hbbs_http/
├── account.rs          # 账户认证模块
├── http_client.rs      # HTTP客户端模块
├── sync.rs            # 同步服务模块
├── record_upload.rs   # 录制上传模块
└── downloader.rs      # 文件下载模块
```

## 模块概述

`hbbs_http` 模块是 RustDesk 远程桌面系统的 HTTP 通信层，负责与 HBBS (RustDesk Broker/ID Server) 服务器进行 HTTP 通信。该模块提供了账户认证、系统信息同步、录制文件上传、文件下载等核心功能，支持多种 TLS 配置、代理设置和自动降级机制。

## 核心功能模块

### 1. 账户认证模块 (account.rs)

#### 功能概述
实现基于 OIDC (OpenID Connect) 协议的用户认证流程，支持设备白名单、用户信息管理和多因素认证。

#### 核心数据结构

**OidcSession**
- `client: Option<Client>` - HTTP客户端
- `state_msg: &'static str` - 当前状态消息
- `failed_msg: String` - 失败消息
- `code_url: Option<OidcAuthUrl>` - 认证URL
- `auth_body: Option<AuthBody>` - 认证响应体
- `keep_querying: bool` - 是否持续查询
- `running: bool` - 是否正在运行
- `query_timeout: Duration` - 查询超时时间

**AuthBody**
- `access_token: String` - 访问令牌
- `type: String` - 认证类型
- `tfa_type: String` - 双因素认证类型
- `secret: String` - 密钥
- `user: UserPayload` - 用户信息

**UserPayload**
- `name: String` - 用户名
- `email: Option<String>` - 邮箱
- `note: Option<String>` - 备注
- `status: UserStatus` - 用户状态
- `info: UserInfo` - 用户详细信息
- `is_admin: bool` - 是否管理员
- `third_auth_type: Option<String>` - 第三方认证类型

**UserInfo**
- `settings: UserSettings` - 用户设置
- `login_device_whitelist: Vec<WhitelistItem>` - 登录设备白名单
- `other: HashMap<String, String>` - 其他信息

**WhitelistItem**
- `data: String` - IP地址或设备UUID
- `info: DeviceInfo` - 设备信息
- `exp: u64` - 过期时间

#### 主要接口

**account_auth**
```rust
pub fn account_auth(
    api_server: String,
    op: String,
    id: String,
    uuid: String,
    remember_me: bool,
)
```
- 参数说明：
  - `api_server`: API服务器地址
  - `op`: 操作类型
  - `id`: 用户ID
  - `uuid`: 设备UUID
  - `remember_me`: 是否记住登录
- 功能：启动OIDC认证流程
- 返回值：无返回值，结果通过 `get_result()` 获取

**auth_cancel**
```rust
pub fn auth_cancel()
```
- 功能：取消正在进行的认证
- 返回值：无

**get_result**
```rust
pub fn get_result() -> AuthResult
```
- 功能：获取认证结果
- 返回值：`AuthResult` 包含状态、错误信息、URL和认证体

#### 核心业务逻辑流程

1. **认证初始化**
   - 创建HTTP客户端
   - 发送认证请求到 `/api/oidc/auth` 端点
   - 获取认证URL和授权码

2. **轮询查询**
   - 每1秒查询一次 `/api/oidc/auth-query` 端点
   - 超时时间为180秒
   - 检查认证状态

3. **认证完成**
   - 获取访问令牌
   - 保存用户信息（如果选择记住登录）
   - 更新本地配置

4. **错误处理**
   - 网络错误
   - 超时错误
   - 服务器错误

#### 关键业务规则

- 认证超时时间为180秒
- 查询间隔为1秒
- 支持记住登录功能，将访问令牌和用户信息保存到本地配置
- 设备白名单用于限制登录设备

### 2. HTTP客户端模块 (http_client.rs)

#### 功能概述
提供同步和异步HTTP客户端，支持多种TLS配置、代理设置、自动降级机制和连接缓存。

#### 核心数据结构

**TlsType** (枚举)
- `Plain` - 无TLS加密
- `NativeTls` - 使用系统原生TLS
- `Rustls` - 使用Rustls TLS库

#### 主要接口

**create_http_client**
```rust
pub fn create_http_client(tls_type: TlsType, danger_accept_invalid_cert: bool) -> SyncClient
```
- 参数说明：
  - `tls_type`: TLS类型
  - `danger_accept_invalid_cert`: 是否接受无效证书
- 功能：创建同步HTTP客户端
- 返回值：`SyncClient` 同步客户端实例

**create_http_client_async**
```rust
pub fn create_http_client_async(
    tls_type: TlsType,
    danger_accept_invalid_cert: bool,
) -> AsyncClient
```
- 参数说明：
  - `tls_type`: TLS类型
  - `danger_accept_invalid_cert`: 是否接受无效证书
- 功能：创建异步HTTP客户端
- 返回值：`AsyncClient` 异步客户端实例

**create_http_client_with_url**
```rust
pub fn create_http_client_with_url(url: &str) -> SyncClient
```
- 参数说明：
  - `url`: 目标URL
- 功能：根据URL自动选择TLS配置创建客户端
- 返回值：`SyncClient` 同步客户端实例

**create_http_client_async_with_url**
```rust
pub async fn create_http_client_async_with_url(url: &str) -> AsyncClient
```
- 参数说明：
  - `url`: 目标URL
- 功能：根据URL自动选择TLS配置创建异步客户端
- 返回值：`AsyncClient` 异步客户端实例

#### 核心业务逻辑流程

1. **客户端创建**
   - 根据TLS类型配置客户端
   - 配置代理（如果需要）
   - 配置证书验证

2. **TLS降级机制**
   - 首先尝试使用 Rustls
   - 失败后尝试接受无效证书
   - 再次失败后尝试 NativeTls
   - 最后尝试 NativeTls + 接受无效证书

3. **连接测试**
   - 发送 HEAD 请求测试连接
   - 根据结果缓存 TLS 配置
   - 提高后续连接速度

4. **代理支持**
   - 支持 HTTP 代理
   - 支持 HTTPS 代理
   - 支持 SOCKS5 代理
   - 支持代理认证

#### 关键业务规则

- 默认使用 Rustls TLS
- 自动降级顺序：Rustls → Rustls(accept invalid) → NativeTls → NativeTls(accept invalid)
- 缓存成功的 TLS 配置以提高性能
- 支持多种代理协议
- Android 和 iOS 平台使用自定义证书验证器

### 3. 同步服务模块 (sync.rs)

#### 功能概述
实现与 HBBS 服务器的同步机制，包括心跳、系统信息上传、策略配置同步和连接状态管理。

#### 核心数据结构

**InfoUploaded**
- `uploaded: bool` - 是否已上传
- `url: String` - 上传URL
- `last_uploaded: Option<Instant>` - 最后上传时间
- `id: String` - 设备ID
- `username: Option<String>` - 用户名

**StrategyOptions**
- `config_options: HashMap<String, String>` - 配置选项
- `extra: HashMap<String, String>` - 额外选项

#### 主要接口

**start**
```rust
pub fn start()
```
- 功能：启动同步服务
- 返回值：无

**signal_receiver**
```rust
pub fn signal_receiver() -> broadcast::Receiver<Vec<i32>>
```
- 功能：获取信号接收器
- 返回值：`broadcast::Receiver<Vec<i32>>` 接收器实例

**is_pro**
```rust
pub fn is_pro() -> bool
```
- 功能：检查是否为 Pro 版本
- 返回值：`bool` 是否为 Pro 版本

#### 核心业务逻辑流程

1. **心跳机制**
   - 每3秒发送一次心跳
   - 有活跃连接时发送心跳
   - 无连接且距离上次心跳小于15秒时跳过

2. **系统信息上传**
   - 收集系统信息（版本、ID、UUID等）
   - 检查是否需要上传（未上传、用户名变化、超时）
   - 计算信息哈希值
   - 如果哈希值未变化且版本相同则跳过上传
   - 上传超时时间为120秒

3. **策略配置同步**
   - 接收服务器下发的策略配置
   - 更新本地配置选项
   - 处理配置时间戳

4. **连接管理**
   - 获取活跃连接列表
   - 接收服务器断开连接指令
   - 通过广播通道通知其他模块

#### 关键业务规则

- 心跳间隔为3秒
- 系统信息上传超时为120秒
- 使用 SHA256 计算信息哈希
- 支持地址簿预设配置
- 支持设备分组配置
- Windows 平台需要特殊处理用户名获取逻辑

### 4. 录制上传模块 (record_upload.rs)

#### 功能概述
实现录制文件的上传功能，支持分片上传、断点续传和文件管理。

#### 核心数据结构

**RecordUploader**
- `client: Client` - HTTP客户端
- `api_server: String` - API服务器地址
- `filepath: String` - 文件路径
- `filename: String` - 文件名
- `upload_size: u64` - 已上传大小
- `running: bool` - 是否正在运行
- `last_send: Instant` - 最后发送时间

#### 主要接口

**run**
```rust
pub fn run(rx: Receiver<RecordState>)
```
- 参数说明：
  - `rx`: 接收录制状态的通道
- 功能：启动上传线程
- 返回值：无

**is_enable**
```rust
pub fn is_enable() -> bool
```
- 功能：检查上传是否启用
- 返回值：`bool` 是否启用

#### 核心业务逻辑流程

1. **新文件处理**
   - 接收新文件路径
   - 解析文件名
   - 初始化上传状态
   - 发送新文件通知

2. **帧数据上传**
   - 检查发送时间间隔（1秒）
   - 检查发送大小阈值（1MB）
   - 读取新增数据
   - 分片上传

3. **尾部处理**
   - 强制上传所有剩余数据
   - 读取文件头部（前1KB）
   - 发送尾部数据
   - 标记上传完成

4. **文件删除**
   - 发送删除通知
   - 清理本地状态

#### 关键业务规则

- 发送时间间隔为1秒
- 发送大小阈值为1MB
- 支持断点续传（记录已上传位置）
- 头部数据最大为1KB
- 使用通道接收录制状态

### 5. 文件下载模块 (downloader.rs)

#### 功能概述
实现文件下载功能，支持进度跟踪、取消下载、自动删除和内存/文件两种存储方式。

#### 核心数据结构

**Downloader**
- `data: Vec<u8>` - 下载的数据（内存存储）
- `path: Option<PathBuf>` - 文件路径（文件存储）
- `total_size: Option<u64>` - 总大小
- `downloaded_size: u64` - 已下载大小
- `error: Option<String>` - 错误信息
- `finished: bool` - 是否完成
- `tx_cancel: UnboundedSender<()>` - 取消通道

**DownloadData**
- `data: Vec<u8>` - 下载的数据
- `path: Option<PathBuf>` - 文件路径
- `total_size: Option<u64>` - 总大小
- `downloaded_size: u64` - 已下载大小
- `error: Option<String>` - 错误信息

#### 主要接口

**download_file**
```rust
pub fn download_file(
    url: String,
    path: Option<PathBuf>,
    auto_del_dur: Option<Duration>,
) -> ResultType<String>
```
- 参数说明：
  - `url`: 下载URL
  - `path`: 保存路径（None表示内存存储）
  - `auto_del_dur`: 自动删除延迟
- 功能：启动文件下载
- 返回值：`ResultType<String>` 下载任务ID

**get_download_data**
```rust
pub fn get_download_data(id: &str) -> ResultType<DownloadData>
```
- 参数说明：
  - `id`: 下载任务ID
- 功能：获取下载状态和数据
- 返回值：`ResultType<DownloadData>` 下载数据

**cancel**
```rust
pub fn cancel(id: &str)
```
- 参数说明：
  - `id`: 下载任务ID
- 功能：取消下载
- 返回值：无

**remove**
```rust
pub fn remove(id: &str)
```
- 参数说明：
  - `id`: 下载任务ID
- 功能：移除下载任务
- 返回值：无

#### 核心业务逻辑流程

1. **下载初始化**
   - 检查任务是否已存在
   - 检查文件是否已存在
   - 创建目录（如果需要）
   - 创建下载任务

2. **获取文件大小**
   - 发送 HEAD 请求
   - 解析 Content-Length 头
   - 保存总大小

3. **下载数据**
   - 发送 GET 请求
   - 分块接收数据
   - 写入文件或内存
   - 更新下载进度

4. **取消处理**
   - 接收取消信号
   - 停止下载
   - 删除已下载文件

5. **自动清理**
   - 下载完成后等待指定时间
   - 自动移除下载任务

#### 关键业务规则

- 支持内存和文件两种存储方式
- 使用通道实现取消机制
- 支持自动删除下载任务
- 实时更新下载进度
- 下载失败时保存错误信息

## 数据流转路径

### 认证流程

```
用户请求 → account_auth() → auth_task()
    ↓
发送认证请求 → /api/oidc/auth
    ↓
获取认证URL → 轮询查询
    ↓
/api/oidc/auth-query → 获取访问令牌
    ↓
保存用户信息 → get_result() 返回结果
```

### 同步流程

```
启动服务 → start() → start_hbbs_sync_async()
    ↓
定时触发（3秒） → 收集系统信息
    ↓
检查上传条件 → 上传系统信息
    ↓
发送心跳 → /api/heartbeat
    ↓
接收响应 → 处理策略配置
    ↓
更新本地配置 → 广播断开信号
```

### 上传流程

```
录制开始 → run() → 接收 RecordState
    ↓
NewFile → handle_new_file() → 初始化上传
    ↓
NewFrame → handle_frame() → 分片上传
    ↓
WriteTail → handle_tail() → 完成上传
    ↓
RemoveFile → handle_remove() → 删除文件
```

### 下载流程

```
下载请求 → download_file() → 创建下载任务
    ↓
获取文件大小 → HEAD 请求
    ↓
开始下载 → GET 请求 → 分块接收
    ↓
写入文件/内存 → 更新进度
    ↓
下载完成 → 自动清理（可选）
    ↓
get_download_data() → 获取结果
```

## 核心算法逻辑

### TLS自动降级算法

```rust
fn create_http_client_with_url_(
    url: &str,
    tls_url: &str,
    tls_type: TlsType,
    is_tls_type_cached: bool,
    danger_accept_invalid_cert: Option<bool>,
    original_danger_accept_invalid_cert: Option<bool>,
) -> SyncClient {
    let mut client = create_http_client(tls_type, danger_accept_invalid_cert.unwrap_or(false));
    if is_tls_type_cached && original_danger_accept_invalid_cert.is_some() {
        return client;
    }
    if let Err(e) = client.head(url).send() {
        match (tls_type, is_tls_type_cached, danger_accept_invalid_cert) {
            (TlsType::Rustls, _, None) => {
                // 尝试接受无效证书
                client = create_http_client_with_url_(..., Some(true), ...);
            }
            (TlsType::Rustls, false, Some(_)) => {
                // 尝试 NativeTls
                client = create_http_client_with_url_(..., TlsType::NativeTls, ...);
            }
            (TlsType::NativeTls, _, None) => {
                // 尝试接受无效证书
                client = create_http_client_with_url_(..., Some(true), ...);
            }
            _ => {
                // 失败
            }
        }
    } else {
        // 成功，缓存配置
        upsert_tls_cache(tls_url, tls_type, danger_accept_invalid_cert.unwrap_or(false));
    }
    client
}
```

### 系统信息哈希计算

```rust
use sha2::{Digest, Sha256};
let mut hasher = Sha256::new();
hasher.update(url.as_bytes());
hasher.update(&v.as_bytes());
let res = hasher.finalize();
let hash = hbb_common::base64::encode(&res[..]);
```

### 分片上传逻辑

```rust
fn handle_frame(&mut self, flush: bool) -> ResultType<()> {
    if !flush && self.last_send.elapsed() < SHOULD_SEND_TIME {
        return Ok(());
    }
    let len = m.len();
    if len <= self.upload_size {
        return Ok(());
    }
    if !flush && len - self.upload_size < SHOULD_SEND_SIZE {
        return Ok(());
    }
    // 读取新增数据
    file.seek(SeekFrom::Start(self.upload_size));
    file.read_to_end(&mut buf);
    // 上传分片
    self.send(&[
        ("type", "part"),
        ("file", &self.filename),
        ("offset", &self.upload_size.to_string()),
        ("length", &length.to_string()),
    ], buf)?;
    self.upload_size = len;
}
```

## 关键业务规则说明

### 认证相关

1. **超时规则**
   - 认证查询超时：180秒
   - 查询间隔：1秒

2. **记住登录**
   - 保存访问令牌到本地配置
   - 保存用户基本信息到本地配置

3. **设备白名单**
   - 支持IP地址和设备UUID
   - 支持设置过期时间
   - 记录设备信息（OS、类型、名称）

### TLS配置相关

1. **降级策略**
   - 优先使用 Rustls
   - 失败后尝试接受无效证书
   - 再次失败后尝试 NativeTls
   - 最后尝试 NativeTls + 接受无效证书

2. **缓存机制**
   - 缓存成功的 TLS 配置
   - 缓存是否接受无效证书
   - 提高后续连接速度

3. **平台差异**
   - Android 和 iOS 使用自定义证书验证器
   - 其他平台使用标准验证器

### 同步相关

1. **心跳规则**
   - 间隔：3秒
   - 有活跃连接时发送
   - 无连接且距离上次心跳小于15秒时跳过

2. **系统信息上传**
   - 超时：120秒
   - 检查哈希值避免重复上传
   - 用户名变化时重新上传
   - Windows 平台特殊处理用户名

3. **策略配置**
   - 服务器下发策略配置
   - 更新本地配置选项
   - 使用时间戳检测变更

### 上传相关

1. **分片规则**
   - 时间间隔：1秒
   - 大小阈值：1MB
   - 支持断点续传

2. **文件管理**
   - 新文件通知
   - 分片上传
   - 尾部处理（前1KB）
   - 删除通知

### 下载相关

1. **存储方式**
   - 内存存储：小文件或临时使用
   - 文件存储：大文件或持久化

2. **进度跟踪**
   - 实时更新已下载大小
   - 显示下载百分比
   - 支持查询下载状态

3. **取消和清理**
   - 支持取消下载
   - 删除已下载文件
   - 支持自动删除任务

## 使用示例

### 账户认证

```rust
use crate::hbbs_http::account;

// 启动认证
account::account_auth(
    "https://api.example.com".to_string(),
    "login".to_string(),
    "user123".to_string(),
    "device-uuid".to_string(),
    true, // 记住登录
);

// 获取认证结果
let result = account::get_result();
match result.state_msg.as_str() {
    "Login account auth" => {
        if let Some(auth_body) = result.auth_body {
            println!("登录成功: {}", auth_body.user.name);
        }
    }
    _ => {
        println!("认证失败: {}", result.failed_msg);
    }
}

// 取消认证
account::auth_cancel();
```

### HTTP客户端

```rust
use crate::hbbs_http::http_client::{create_http_client_with_url, TlsType};

// 自动选择TLS配置
let client = create_http_client_with_url("https://api.example.com");

// 手动指定TLS配置
let client = create_http_client(TlsType::Rustls, false);

// 发送请求
let response = client.get("https://api.example.com/data").send()?;
```

### 同步服务

```rust
use crate::hbbs_http::sync;

// 启动同步服务
sync::start();

// 获取信号接收器
let mut rx = sync::signal_receiver();

// 接收断开连接信号
tokio::spawn(async move {
    while let Ok(conns) = rx.recv().await {
        println!("断开连接: {:?}", conns);
    }
});

// 检查是否为 Pro 版本
if sync::is_pro() {
    println!("Pro 版本");
}
```

### 录制上传

```rust
use crate::hbbs_http::record_upload;
use scrap::record::RecordState;
use std::sync::mpsc;

// 创建通道
let (tx, rx) = mpsc::channel();

// 启动上传线程
record_upload::run(rx);

// 发送录制状态
tx.send(RecordState::NewFile("/path/to/recording.mp4".to_string()))?;
tx.send(RecordState::NewFrame)?;
tx.send(RecordState::WriteTail)?;
tx.send(RecordState::RemoveFile)?;
```

### 文件下载

```rust
use crate::hbbs_http::downloader;
use std::path::PathBuf;

// 下载到文件
let id = downloader::download_file(
    "https://example.com/file.zip".to_string(),
    Some(PathBuf::from("/path/to/file.zip")),
    None,
)?;

// 下载到内存
let id = downloader::download_file(
    "https://example.com/data.json".to_string(),
    None,
    None,
)?;

// 查询下载状态
let data = downloader::get_download_data(&id)?;
println!("下载进度: {}/{}", data.downloaded_size, data.total_size.unwrap_or(0));

// 取消下载
downloader::cancel(&id);

// 移除任务
downloader::remove(&id);
```

## 注意事项

### 安全性

1. **证书验证**
   - 默认验证服务器证书
   - 仅在必要时接受无效证书
   - 使用安全的 TLS 配置

2. **敏感信息**
   - 访问令牌存储在本地配置
   - 不在日志中输出敏感信息
   - 使用 HTTPS 传输

3. **代理配置**
   - 支持代理认证
   - 不在日志中输出代理密码

### 性能优化

1. **连接复用**
   - 缓存 TLS 配置
   - 复用 HTTP 客户端
   - 使用连接池

2. **批量操作**
   - 分片上传减少请求次数
   - 批量处理系统信息
   - 使用异步操作

3. **资源管理**
   - 及时释放资源
   - 限制并发下载数量
   - 自动清理过期任务

### 错误处理

1. **网络错误**
   - 自动重试（TLS降级）
   - 记录错误日志
   - 返回错误信息

2. **超时处理**
   - 设置合理的超时时间
   - 避免长时间阻塞
   - 提供取消机制

3. **数据验证**
   - 验证响应数据格式
   - 检查文件完整性
   - 处理边界情况

### 平台兼容性

1. **Windows**
   - 特殊处理用户名获取
   - 处理会话切换

2. **Android/iOS**
   - 使用自定义证书验证器
   - 适配移动平台特性

3. **Linux/macOS**
   - 标准处理流程
   - 支持原生 TLS

## 总结

`hbbs_http` 模块是 RustDesk 系统的核心通信层，提供了完整的 HTTP 通信功能。该模块具有以下特点：

1. **模块化设计**：各功能模块职责清晰，易于维护和扩展
2. **健壮性**：完善的错误处理和降级机制
3. **安全性**：支持多种 TLS 配置，保护数据传输安全
4. **性能优化**：连接复用、批量操作、资源管理
5. **跨平台**：支持 Windows、Linux、macOS、Android、iOS
6. **易用性**：简洁的 API 设计，方便调用

通过本文档，开发人员可以快速理解 `hbbs_http` 模块的架构和实现，便于后续的维护和开发工作。
