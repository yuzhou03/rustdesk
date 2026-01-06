# CLI 模块文档

## 模块概述

`cli.rs` 模块提供了 RustDesk 命令行界面的核心功能，主要用于实现远程桌面连接的 CLI 模式，包括连接测试和端口转发功能。该模块基于异步编程模型，使用 `tokio` 运行时和 `async-trait` 来实现异步接口。

该模块的主要职责包括：
- 管理远程连接会话
- 处理登录认证流程
- 提供端口转发服务
- 处理各种消息和事件

## 主要功能点

### 1. 会话管理 (Session)

`Session` 结构体是 CLI 模式的核心组件，负责管理一个完整的远程连接会话。

**结构体定义：**
```rust
pub struct Session {
    id: String,                                    // 会话ID（远程设备ID）
    lc: Arc<RwLock<LoginConfigHandler>>,           // 登录配置处理器
    sender: mpsc::UnboundedSender<Data>,          // 数据发送通道
    password: String,                              // 连接密码
}
```

**主要特性：**
- 实现 `Clone` trait，支持会话的克隆
- 使用 `Arc<RwLock>` 实现线程安全的配置共享
- 使用无界通道进行异步消息传递

### 2. 连接测试 (connect_test)

提供连接测试功能，用于验证与远程设备的连接是否正常。

**功能特点：**
- 使用 `tokio` 异步运行时
- 支持密钥和令牌认证
- 自动处理超时和错误
- 测试完成后自动断开连接

### 3. 端口转发 (start_one_port_forward)

提供本地端口到远程端口的转发服务。

**功能特点：**
- 支持本地端口监听
- 支持远程主机和端口配置
- 自动测试中继服务器和NAT类型
- 完整的错误处理和日志记录

## 参数说明

### Session::new()

| 参数名 | 类型 | 说明 |
|--------|------|------|
| id | &str | 远程设备的唯一标识符 |
| sender | mpsc::UnboundedSender<Data> | 用于发送数据的无界通道发送端 |

**返回值：** 返回一个新创建的 `Session` 实例

**特殊行为：**
- 如果配置中未保存密码，会通过命令行提示用户输入密码
- 自动初始化登录配置处理器
- 连接类型默认为 `PORT_FORWARD`

### connect_test()

| 参数名 | 类型 | 说明 |
|--------|------|------|
| id | &str | 要连接的远程设备ID |
| key | String | 连接密钥 |
| token | String | 认证令牌 |

**返回值：** 无返回值（异步函数）

**执行流程：**
1. 创建会话和数据通道
2. 启动客户端连接
3. 等待接收 Hash 消息
4. 处理超时和错误
5. 连接测试完成后退出

### start_one_port_forward()

| 参数名 | 类型 | 说明 |
|--------|------|------|
| id | String | 远程设备ID |
| port | i32 | 本地监听端口 |
| remote_host | String | 远程主机地址 |
| remote_port | i32 | 远程目标端口 |
| key | String | 连接密钥 |
| token | String | 认证令牌 |

**返回值：** 无返回值（异步函数）

**执行流程：**
1. 测试中继服务器和NAT类型
2. 创建会话和数据通道
3. 启动端口转发监听服务
4. 处理错误和日志记录
5. 服务退出时记录日志

## 返回值解释

### Session::new()
- **成功：** 返回初始化完成的 `Session` 实例
- **失败：** 如果密码输入失败，会触发 panic

### connect_test()
- **无显式返回值**
- 通过日志输出连接状态：
  - 成功：输出 "direct: {true/false}" 表示是否直连
  - 失败：输出错误信息 "Failed to connect {id}: {error}"
  - 超时：输出 "Timeout"

### start_one_port_forward()
- **无显式返回值**
- 通过日志输出执行状态：
  - 成功启动：无输出
  - 失败：输出 "Failed to listen on {port}: {error}"
  - 退出：输出 "port forward (:{port}) exit"

## 关键代码说明

### 1. Session 初始化

```rust
impl Session {
    pub fn new(id: &str, sender: mpsc::UnboundedSender<Data>) -> Self {
        let mut password = "".to_owned();
        if PeerConfig::load(id).password.is_empty() {
            password = rpassword::prompt_password("Enter password: ").unwrap();
        }
        let session = Self {
            id: id.to_owned(),
            sender,
            password,
            lc: Default::default(),
        };
        session.lc.write().unwrap().initialize(
            id.to_owned(),
            ConnType::PORT_FORWARD,
            None,
            false,
            None,
            None,
        );
        session
    }
}
```

**说明：**
- 首先检查是否已保存密码，如果没有则提示用户输入
- 创建 Session 实例并初始化所有字段
- 初始化登录配置处理器，设置连接类型为端口转发
- 使用 `rpassword` 库实现安全的密码输入

### 2. 消息处理 (msgbox)

```rust
fn msgbox(&self, msgtype: &str, title: &str, text: &str, link: &str) {
    match msgtype {
        "input-password" => {
            self.sender.send(Data::Login((self.password.clone(), true))).ok();
        }
        "re-input-password" => {
            log::error!("{}: {}", title, text);
            match rpassword::prompt_password("Enter password: ") {
                Ok(password) => {
                    let login_data = Data::Login((password, true));
                    self.sender.send(login_data).ok();
                }
                Err(e) => {
                    log::error!("reinput password failed, {:?}", e);
                }
            }
        }
        msg if msg.contains("error") => {
            log::error!("{}: {}: {}", msgtype, title, text);
        }
        _ => {
            log::info!("{}: {}: {}", msgtype, title, text);
        }
    }
}
```

**说明：**
- 处理不同类型的消息框事件
- "input-password"：使用保存的密码自动登录
- "re-input-password"：密码错误时重新提示输入
- 包含 "error" 的消息：记录错误日志
- 其他消息：记录信息日志

### 3. 连接测试实现

```rust
#[tokio::main(flavor = "current_thread")]
pub async fn connect_test(id: &str, key: String, token: String) {
    let (sender, mut receiver) = mpsc::unbounded_channel::<Data>();
    let handler = Session::new(&id, sender);
    match crate::client::Client::start(id, &key, &token, ConnType::PORT_FORWARD, handler).await {
        Err(err) => {
            log::error!("Failed to connect {}: {}", &id, err);
        }
        Ok((mut stream, direct)) => {
            log::info!("direct: {}", direct);
            loop {
                tokio::select! {
                    res = hbb_common::timeout(READ_TIMEOUT, stream.next()) => match res {
                        Err(_) => {
                            log::error!("Timeout");
                            break;
                        }
                        Ok(Some(Ok(bytes))) => {
                            if let Ok(msg_in) = Message::parse_from_bytes(&bytes) {
                                match msg_in.union {
                                    Some(message::Union::Hash(hash)) => {
                                        log::info!("Got hash");
                                        break;
                                    }
                                    _ => {}
                                }
                            }
                        }
                        _ => {}
                    }
                }
            }
        }
    }
}
```

**说明：**
- 使用 `tokio::main` 宏创建异步运行时
- 创建无界通道用于数据传输
- 启动客户端连接，使用 PORT_FORWARD 连接类型
- 使用 `tokio::select!` 宏实现超时控制
- 解析 protobuf 消息，等待 Hash 消息确认连接成功
- 超时或收到 Hash 消息后退出循环

### 4. 端口转发实现

```rust
#[tokio::main(flavor = "current_thread")]
pub async fn start_one_port_forward(
    id: String,
    port: i32,
    remote_host: String,
    remote_port: i32,
    key: String,
    token: String,
) {
    crate::common::test_rendezvous_server();
    crate::common::test_nat_type();
    let (sender, mut receiver) = mpsc::unbounded_channel::<Data>();
    let handler = Session::new(&id, sender);
    if let Err(err) = crate::port_forward::listen(
        handler.id.clone(),
        handler.password.clone(),
        port,
        handler.clone(),
        receiver,
        &key,
        &token,
        handler.lc.clone(),
        remote_host,
        remote_port,
    )
    .await
    {
        log::error!("Failed to listen on {}: {}", port, err);
    }
    log::info!("port forward (:{}) exit", port);
}
```

**说明：**
- 首先测试中继服务器和NAT类型
- 创建会话和数据通道
- 调用 `port_forward::listen` 启动端口转发服务
- 传递所有必要的参数（ID、密码、端口、会话处理器等）
- 处理错误并记录日志
- 服务退出时记录端口信息

## 使用示例

### 示例 1：连接测试

```rust
use rustdesk::cli;

#[tokio::main]
async fn main() {
    let remote_id = "123456789";
    let key = "your-connection-key".to_string();
    let token = "your-auth-token".to_string();
    
    cli::connect_test(remote_id, key, token).await;
}
```

**说明：**
- 提供远程设备ID、密钥和令牌
- 函数会自动连接并测试连接状态
- 通过日志输出连接结果

### 示例 2：端口转发

```rust
use rustdesk::cli;

#[tokio::main]
async fn main() {
    let remote_id = "123456789".to_string();
    let local_port = 8080;
    let remote_host = "192.168.1.100".to_string();
    let remote_port = 3389;
    let key = "your-connection-key".to_string();
    let token = "your-auth-token".to_string();
    
    cli::start_one_port_forward(
        remote_id,
        local_port,
        remote_host,
        remote_port,
        key,
        token,
    ).await;
}
```

**说明：**
- 配置本地端口 8080 转发到远程主机的 3389 端口
- 服务会持续运行直到手动终止
- 所有连接数据会自动转发

### 示例 3：创建自定义会话

```rust
use rustdesk::cli::Session;
use tokio::sync::mpsc;

#[tokio::main]
async fn main() {
    let remote_id = "123456789";
    let (sender, mut receiver) = mpsc::unbounded_channel();
    
    let session = Session::new(remote_id, sender);
    
    // 使用会话进行其他操作
    // ...
}
```

## 注意事项

### 1. 密码安全
- 密码输入使用 `rpassword` 库，不会在终端显示明文
- 密码仅在内存中保存，不会写入日志
- 建议使用强密码并定期更换

### 2. 异步运行时
- 所有公共函数都使用 `#[tokio::main(flavor = "current_thread")]` 宏
- 使用单线程运行时，适合 CLI 应用
- 不要在异步函数中使用阻塞操作

### 3. 错误处理
- 大部分错误通过日志输出，不会抛出异常
- 使用 `.ok()` 忽略发送错误，避免通道关闭导致 panic
- 关键操作（如密码输入）失败会触发 panic

### 4. 资源管理
- 使用 `Arc<RwLock>` 实现线程安全的配置共享
- 无界通道可能导致内存压力，注意控制消息速率
- 连接超时使用 `READ_TIMEOUT` 常量配置

### 5. 连接类型
- 默认使用 `ConnType::PORT_FORWARD` 连接类型
- 如需其他连接类型，需要修改 `Session::new` 中的初始化代码
- 不同连接类型可能有不同的行为和限制

### 6. 日志记录
- 使用 `log` crate 进行日志记录
- 错误日志使用 `log::error!`
- 信息日志使用 `log::info!`
- 建议配置适当的日志级别

### 7. 依赖项
- 需要 `tokio` 异步运行时
- 需要 `async-trait` 支持 trait 的异步方法
- 需要 `hbb_common` 提供公共功能
- 需要 `rpassword` 用于安全密码输入

### 8. 线程安全
- `Session` 实现了 `Clone`，可以在线程间共享
- 使用 `Arc<RwLock>` 确保配置的线程安全
- 发送通道是无界的，需要注意背压问题

### 9. 性能考虑
- 端口转发使用异步 I/O，性能较好
- 连接测试会等待 Hash 消息，可能有一定延迟
- 避免在循环中创建大量会话

### 10. 兼容性
- 需要支持异步的 Rust 版本（1.39+）
- 依赖 protobuf 进行消息序列化
- 需要网络连接和适当的防火墙配置

## 相关模块

- `crate::client` - 客户端核心功能
- `crate::port_forward` - 端口转发实现
- `crate::common` - 公共工具函数
- `hbb_common` - 公共库，提供配置、消息、网络等功能

## 总结

`cli.rs` 模块为 RustDesk 提供了完整的命令行界面支持，包括连接测试和端口转发两大核心功能。该模块设计简洁，基于异步编程模型，具有良好的性能和可扩展性。开发者可以通过该模块轻松实现 CLI 模式的远程桌面连接和端口转发服务。