# mitmproxy 主调度循环（Master）设计与流量事件序列分析

## 1. 整体架构概览

mitmproxy 采用了层次化、事件驱动的架构设计，主要由以下核心组件构成：

```
┌─────────────────────────────────────────────────────────────────┐
│                         Master (主调度器)                         │
│  ┌─────────────┐  ┌───────────────┐  ┌─────────────────────┐  │
│  │  Options    │  │  Commands     │  │   AddonManager      │  │
│  │ (配置管理)  │  │  (命令管理)   │  │   (插件管理器)      │  │
│  └─────────────┘  └───────────────┘  └─────────────────────┘  │
└─────────────────────────────────────────────────────────────────┘
                              ▲
                              │ 事件/命令
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      Proxyserver Addon                            │
│  ┌─────────────────────────────────────────────────────────────┐ │
│  │                    ConnectionHandler                          │ │
│  │  ┌──────────┐  ┌──────────┐  ┌─────────────────────────┐ │ │
│  │  │  客户    │  │  服务器  │  │   Layer Stack (层栈)    │ │ │
│  │  │  连接    │  │  连接    │  │                         │ │ │
│  │  └──────────┘  └──────────┘  │  TLSLayer               │ │ │
│  │                                │  HttpLayer              │ │ │
│  │                                │  WebSocketLayer         │ │ │
│  │                                └─────────────────────────┘ │ │
│  └─────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

## 2. Master 主调度循环设计

### 2.1 Master 类的核心职责

Master 类定义在 `mitmproxy/master.py`，是 mitmproxy 的中央调度器，主要职责包括：

1. **事件循环管理**：管理 asyncio 事件循环
2. **组件协调**：整合 Options、AddonManager、CommandManager
3. **生命周期控制**：控制启动、运行、关闭全生命周期
4. **上下文设置**：设置全局上下文对象（`ctx`）

### 2.2 核心数据结构

```python
class Master:
    event_loop: asyncio.AbstractEventLoop  # asyncio 事件循环
    options: options.Options                # 配置选项
    commands: command.CommandManager         # 命令管理器
    addons: addonmanager.AddonManager       # 插件管理器
    should_exit: asyncio.Event               # 退出信号
```

### 2.3 主循环执行流程 (`run()` 方法)

Master 的 `run()` 方法实现了完整的启动和运行流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                        Master.run()                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 1. 安装异常处理器和 eager task factory                            │
│    - asyncio_utils.install_exception_handler()                   │
│    - asyncio_utils.set_eager_task_factory()                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 2. 错误检查 (errorcheck addon)                                    │
│    - 检查启动前是否有致命错误                                      │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 3. 启动代理服务器 (proxyserver addon)                             │
│    - 调用 ps.setup_servers() 启动监听                             │
│    - 同时监控 should_exit 事件                                    │
│    - asyncio.wait([setup_servers, should_exit.wait()])          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 4. 触发 running 事件                                               │
│    - await self.running()                                         │
│    - 触发 RunningHook() 通知所有插件                               │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 5. 等待退出信号                                                    │
│    - await self.should_exit.wait()                                │
│    - 阻塞直到 shutdown() 被调用                                   │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│ 6. 清理 (finally 块)                                               │
│    - await self.done()                                            │
│    - 触发 DoneHook() 通知所有插件                                  │
│    - 卸载日志处理器                                                │
└─────────────────────────────────────────────────────────────────┘
```

### 2.4 异步事件循环的关键特性

**Eager Task Factory（急切任务工厂）**

Master 使用 `asyncio_utils.set_eager_task_factory()` 来优化异步任务调度：
- 任务在创建时立即执行，而不是等待下一个事件循环迭代
- 减少任务调度延迟
- 注意：在 `server_event()` 中需要加锁防止重入问题

**异常处理机制**

```python
def _asyncio_exception_handler(self, loop, context) -> None:
    try:
        exc: Exception = context["exception"]
    except KeyError:
        logger.error(f"Unhandled asyncio error: {context}")
    else:
        # 特殊处理：忽略 Windows 上的特定 OSError
        if isinstance(exc, OSError) and exc.errno == 10038:
            return
        logger.error("Unhandled error in task.", exc_info=...)
```

## 3. 流量事件序列分析

### 3.1 HTTP 请求的完整生命周期

一个 HTTP 请求从客户端进入到响应返回经历以下事件节点：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        HTTP 流量事件序列                                       │
└─────────────────────────────────────────────────────────────────────────────┘

客户端连接阶段
───────────────
 1. client_connected      - 客户端连接建立
                           (ClientConnectedHook)

协议协商阶段
───────────────
 2. next_layer            - 决定下一层协议 (NextLayerHook)
                           - 通常选择 TLSLayer 或 HttpLayer

 3. tls_start_client      - 开始客户端 TLS 握手
 4. tls_established_client - TLS 握手完成

服务器连接阶段
───────────────
 5. server_connect        - 准备连接上游服务器
                           (ServerConnectHook)

 6. server_connected      - 服务器连接建立
                           (ServerConnectedHook)

 7. tls_start_server      - 开始服务器端 TLS 握手
 8. tls_established_server - 服务器 TLS 完成

HTTP 请求阶段
───────────────
 9. requestheaders        - HTTP 请求头已读取
                           (HttpRequestHeadersHook)
                           - 此时 body 为空，可修改 headers

10. request               - 完整 HTTP 请求已读取
                           (HttpRequestHook)
                           - 包含完整 body（非流式时）

HTTP 响应阶段
───────────────
11. responseheaders       - HTTP 响应头已读取
                           (HttpResponseHeadersHook)
                           - 此时 body 为空

12. response              - 完整 HTTP 响应已读取
                           (HttpResponseHook)
                           - 包含完整 body（非流式时）

连接关闭阶段
───────────────
13. server_disconnected   - 服务器连接关闭
                           (ServerDisconnectedHook)

14. client_disconnected   - 客户端连接关闭
                           (ClientDisconnectedHook)
```

### 3.2 事件序列生成器 (`eventsequence.py`)

mitmproxy 提供了 `eventsequence.iterate()` 函数，用于从 Flow 对象生成标准事件序列：

```python
def iterate(f: flow.Flow) -> TEventGenerator:
    """根据 Flow 类型生成对应的事件序列"""
```

**HTTP 流的事件序列** (`_iterate_http`):

```python
def _iterate_http(f: http.HTTPFlow) -> TEventGenerator:
    if f.request:
        yield HttpRequestHeadersHook(f)
        yield HttpRequestHook(f)
    if f.response:
        yield HttpResponseHeadersHook(f)
        yield HttpResponseHook(f)
    if f.websocket:
        yield WebsocketStartHook(f)
        for m in message_queue:
            yield WebsocketMessageHook(f)
        yield WebsocketEndHook(f)
    elif f.error:
        yield HttpErrorHook(f)
```

**其他流类型的事件序列**:

| 流类型 | 事件序列 |
|--------|----------|
| TCPFlow | TcpStartHook → TcpMessageHook* → TcpEndHook/TcpErrorHook |
| UDPFlow | UdpStartHook → UdpMessageHook* → UdpEndHook/UdpErrorHook |
| DNSFlow | DnsRequestHook → DnsResponseHook/DnsErrorHook |

### 3.3 关键事件节点详解

**HttpRequestHeadersHook (`requestheaders`)**
- 文件位置: `mitmproxy/proxy/layers/http/_hooks.py:8`
- 触发时机: HTTP 请求头完全读取后，body 读取前
- 特点: 此时 `flow.request.body` 为空
- 用途: 修改请求头、决定是否拦截

**HttpRequestHook (`request`)**
- 文件位置: `mitmproxy/proxy/layers/http/_hooks.py:18`
- 触发时机: 完整 HTTP 请求读取后
- 特点: 
  - 非流式: 包含完整 body
  - 流式: body 已被发送到服务器
  - 警告: 启用流式时，`response` 可能在 `request` 之前触发（如服务器返回 413）
- 用途: 修改请求、生成自定义响应

**HttpResponseHeadersHook (`responseheaders`)**
- 文件位置: `mitmproxy/proxy/layers/http/_hooks.py:33`
- 触发时机: HTTP 响应头完全读取后
- 用途: 修改响应头

**HttpResponseHook (`response`)**
- 文件位置: `mitmproxy/proxy/layers/http/_hooks.py:43`
- 触发时机: 完整 HTTP 响应读取后
- 用途: 修改响应内容

**HttpErrorHook (`error`)**
- 文件位置: `mitmproxy/proxy/layers/http/_hooks.py:56`
- 触发时机: 发生 HTTP 错误（如无效响应、连接中断）
- 重要约束: 每个 flow 要么收到 `error`，要么收到 `response`，不会同时收到

### 3.4 连接生命周期事件

**服务器连接相关事件** (`server_hooks.py`):

```
server_connect (ServerConnectHook)
    │
    ▼
┌──────────────┐
│  连接上游     │
│  服务器      │
└──────────────┘
    │
    ├──成功──► server_connected (ServerConnectedHook)
    │
    └──失败──► server_connect_error (ServerConnectErrorHook)
```

**注意事项**:
- 每个服务器连接要么收到 `server_connected`，要么收到 `server_connect_error`
- `server_connect` 中设置 `data.server.error` 可以终止连接

## 4. 事件异步分发机制

### 4.1 插件事件分发架构

mitmproxy 的事件分发由 `AddonManager` 负责，定义在 `mitmproxy/addonmanager.py`。

**核心分发流程**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    AddonManager.trigger_event()                  │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    ┌─────────────────┐
                    │  遍历插件链      │
                    │  self.chain     │
                    └─────────────────┘
                              │
                    ┌─────────────────┐
                    │  safecall()    │
                    │  异常保护       │
                    └─────────────────┘
                              │
                    ┌─────────────────┐
                    │ invoke_addon() │
                    └─────────────────┘
                              │
            ┌─────────────────┼─────────────────┐
            │                 │                 │
            ▼                 ▼                 ▼
    ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
    │  Addon A     │  │  Addon B     │  │  Addon C     │
    │ .request()   │  │ .request()   │  │ .request()   │
    └──────────────┘  └──────────────┘  └──────────────┘
```

### 4.2 同步与异步事件处理

AddonManager 支持两种调用方式：

**异步调用 (`invoke_addon`)**：
```python
async def invoke_addon(self, addon, event: hooks.Hook):
    for addon, func in self._iter_hooks(addon, event):
        res = func(*event.args())
        # 同时支持同步和异步钩子函数
        if res is not None and inspect.isawaitable(res):
            await res
```

**同步调用 (`invoke_addon_sync`)**：
```python
def invoke_addon_sync(self, addon, event: hooks.Hook):
    for addon, func in self._iter_hooks(addon, event):
        if inspect.iscoroutinefunction(func):
            raise exceptions.AddonManagerError(
                f"Async handler {event.name} cannot be called from sync context"
            )
        func(*event.args())
```

### 4.3 层（Layer）的事件与命令机制

mitmproxy 代理核心采用了 **Sans-IO** 设计模式，通过 `Layer` 层栈处理协议。

**核心概念**:
- **事件 (Events)**: 从外部传入层的通知（如 DataReceived、ConnectionClosed）
- **命令 (Commands)**: 层向外部发出的指令（如 SendData、OpenConnection、StartHook）

**Layer 基类** (`mitmproxy/proxy/layer.py`):

```python
class Layer:
    context: Context                    # 层上下文
    _paused: Paused | None              # 暂停状态（等待阻塞命令完成）
    _paused_event_queue: deque          # 暂停期间的事件队列
    
    @abstractmethod
    def _handle_event(self, event: events.Event) -> CommandGenerator[None]:
        """处理事件，生成命令"""
        yield from ()
```

**生成器模式模拟阻塞代码**:

Layer 使用 Python 生成器实现了类似协程的阻塞语义，但实际上是非阻塞的：

```python
def _handle_event(self, event):
    # 看起来像阻塞代码，实际通过 yield 实现非阻塞
    err = yield OpenConnection(server)  # 暂停等待连接完成
    if err:
        return
    
    # 连接建立后继续执行
    yield SendData(server, request_data)
```

**暂停/恢复机制**:

```
┌─────────────────────────────────────────────────────────────────┐
│                    Layer.handle_event() 执行流程                  │
└─────────────────────────────────────────────────────────────────┘

正常执行路径:
───────────────
1. 接收事件 (Event)
        │
        ▼
2. 调用 _handle_event(event)
        │
        ▼
3. 生成器产出命令 (yield Command)
        │
        ├── 非阻塞命令 ──► 立即返回，继续执行生成器
        │
        └── 阻塞命令 ──► 设置 _paused 状态
                               │
                               ▼
                        4. 等待 CommandCompleted 事件
                               │
                               ▼
                        5. __continue() 恢复执行
                               │
                               ▼
                        6. 处理 _paused_event_queue 中的缓冲事件
```

### 4.4 命令类型详解

**命令基类** (`mitmproxy/proxy/commands.py`):

```python
class Command:
    blocking: Union[bool, Layer] = False
    # True = 阻塞等待完成
    # False = 非阻塞
    # Layer = 已被某层处理，外层无需阻塞
```

**主要命令类型**:

| 命令类 | 阻塞 | 用途 |
|--------|------|------|
| OpenConnection | 是 | 打开新的网络连接 |
| StartHook | 是 | 触发插件事件钩子 |
| SendData | 否 | 发送数据到连接 |
| CloseConnection | 否 | 关闭连接 |
| RequestWakeup | 否 | 请求定时唤醒 |
| Log | 否 | 记录日志 |

### 4.5 ConnectionHandler 的事件循环

`ConnectionHandler` 定义在 `mitmproxy/proxy/server.py`，负责单个客户端连接的事件处理：

**核心方法 `server_event()`**:

```python
async def server_event(self, event: events.Event) -> None:
    async with self._server_event_lock:  # 防止重入
        self.timeout_watchdog.register_activity()
        
        # 将事件传给层栈处理，获取命令列表
        layer_commands = self.layer.handle_event(event)
        
        for command in layer_commands:
            if isinstance(command, commands.OpenConnection):
                # 创建异步任务打开连接
                asyncio_utils.create_task(self.open_connection(command), ...)
                
            elif isinstance(command, commands.SendData):
                # 立即写入数据
                writer.write(command.data)
                
            elif isinstance(command, commands.StartHook):
                # 创建异步任务处理钩子
                asyncio_utils.create_task(self.hook_task(command), ...)
                
            elif isinstance(command, commands.Log):
                self.log(command.message, command.level)
```

**事件处理的异步特性**:

1. **OpenConnection**: 创建独立的异步任务，不阻塞当前事件处理
2. **StartHook**: 创建独立的异步任务执行插件钩子
3. **SendData**: 同步写入（但实际写入是缓冲的）
4. **超时看门狗**: 独立的异步任务监控连接活动

### 4.6 阻塞钩子的处理机制

**Hook 任务流程** (`hook_task` 方法):

```python
async def hook_task(self, hook: commands.StartHook) -> None:
    # 1. 执行钩子（可能阻塞等待插件完成）
    await self.handle_hook(hook)
    
    # 2. 如果是阻塞钩子，发送 HookCompleted 事件恢复层执行
    if hook.blocking:
        await self.server_event(events.HookCompleted(hook))
```

这意味着：
- 插件可以在钩子函数中执行异步操作（async/await）
- 层的执行会暂停直到所有插件处理完成
- 这使得插件可以安全地修改 flow 对象

## 5. 组件间的协调机制

### 5.1 Master ↔ Proxyserver 的协调

**启动流程**:
1. Master.run() 调用 `proxyserver.setup_servers()`
2. Proxyserver 创建 `ServerInstance` 并开始监听
3. 新连接到来时，创建 `ProxyConnectionHandler`
4. ConnectionHandler 处理连接生命周期

**关闭流程**:
1. Master.shutdown() 调用 `should_exit.set()`（线程安全）
2. Master.run() 中的 `should_exit.wait()` 返回
3. 执行 `done()` 触发 `DoneHook`

### 5.2 事件流的完整路径

```
┌──────────┐     ┌──────────────┐     ┌──────────────┐     ┌──────────┐
│  客户端   │────►│ Connection   │────►│   Layer      │────►│  Plugin  │
│          │     │  Handler     │     │   层栈       │     │  插件    │
└──────────┘     └──────────────┘     └──────────────┘     └──────────┘
      ▲                  │                    │                    │
      │                  │                    │                    │
      │                  ▼                    ▼                    ▼
      │            ┌──────────────┐     ┌──────────────┐     ┌──────────┐
      │            │   Events     │     │  Commands    │     │  Hooks   │
      │            │  (事件输入)  │     │  (命令输出)  │     │ (事件钩子)│
      │            └──────────────┘     └──────────────┘     └──────────┘
      │                  ▲                    │                    │
      │                  │                    ▼                    │
      │            ┌─────────────────────────────────┐             │
      └────────────│    server_event() 处理循环      │◄────────────┘
                   └─────────────────────────────────┘
```

### 5.3 上下文对象 (`ctx`)

mitmproxy 使用全局上下文对象简化组件间访问：

```python
# mitmproxy/ctx.py
from mitmproxy import types

class _Context:
    master: types.Master  # type: ignore
    log: types.Log        # type: ignore
    options: types.Options  # type: ignore

ctx = _Context()
```

**在 Master.__init__ 中设置**:
```python
mitmproxy_ctx.master = self
mitmproxy_ctx.log = self.log
mitmproxy_ctx.options = self.options
```

这样插件可以通过 `from mitmproxy import ctx` 访问全局资源。

## 6. 关键代码位置索引

| 功能模块 | 文件路径 | 关键类/函数 |
|----------|----------|-------------|
| 主调度器 | `mitmproxy/master.py` | `Master` 类, `run()` 方法 |
| 插件管理器 | `mitmproxy/addonmanager.py` | `AddonManager`, `trigger_event()` |
| 事件序列 | `mitmproxy/eventsequence.py` | `iterate()`, `_iterate_http()` |
| 钩子定义 | `mitmproxy/hooks.py` | `Hook` 基类 |
| HTTP 事件钩子 | `mitmproxy/proxy/layers/http/_hooks.py` | `HttpRequestHook`, `HttpResponseHook` |
| 服务器连接钩子 | `mitmproxy/proxy/server_hooks.py` | `ClientConnectedHook`, `ServerConnectHook` |
| 代理服务器 | `mitmproxy/proxy/server.py` | `ConnectionHandler`, `server_event()` |
| 层基类 | `mitmproxy/proxy/layer.py` | `Layer`, `NextLayer` |
| 命令定义 | `mitmproxy/proxy/commands.py` | `Command`, `StartHook`, `OpenConnection` |
| 代理服务插件 | `mitmproxy/addons/proxyserver.py` | `Proxyserver` |

## 7. 设计亮点与架构优势

### 7.1 Sans-IO 设计
- 层（Layer）本身不执行任何 IO
- 所有 IO 通过命令（Command）委派给 ConnectionHandler
- 便于测试和协议逻辑复用

### 7.2 生成器模拟协程
- 使用 Python 生成器实现类似 async/await 的语义
- 比原生协程更精细的控制（可暂停、队列事件）
- 单个连接内的事件串行化，避免竞争条件

### 7.3 插件系统的灵活性
- 支持同步和异步钩子函数
- 事件串行执行，插件可以安全地修改 flow
- 插件可以动态添加/移除

### 7.4 分层架构
- 协议层独立（TLS、HTTP、WebSocket、TCP、UDP、DNS）
- 通过 `next_layer` 钩子动态决定层栈
- 易于扩展新协议

## 8. 注意事项与边界情况

### 8.1 流式传输的事件顺序
启用 `stream_large_bodies` 时：
- `response` 可能在 `request` 之前触发（如服务器返回 413）
- Body 不会存储在内存中
- `store_streamed_bodies` 选项可以存储流式 body（增加内存消耗）

### 8.2 连接策略
- `eager`（默认）: 尽早建立上游连接
  - 可检测服务端问候协议
  - 准确镜像 TLS ALPN 协商
- `lazy`: 延迟建立连接
  - 支持离线服务器重放
  - `http_connected` 事件可能在上游连接建立前触发

### 8.3 异常处理
- 插件异常被 `safecall()` 捕获并记录
- 单个插件失败不会影响其他插件
- `AddonHalt` 异常可以提前终止事件传播

---

*分析基于 mitmproxy 代码库，文件路径均相对于项目根目录*
