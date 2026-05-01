# Mitmproxy 协议层架构深度分析报告

## 目录
1. [架构概览](#架构概览)
2. [协议处理单元的嵌套机制](#协议处理单元的嵌套机制)
3. [网络事件传递与冒泡机制](#网络事件传递与冒泡机制)
4. [协议升级与无缝切换机制](#协议升级与无缝切换机制)
5. [命令系统与事件的双向通信](#命令系统与事件的双向通信)
6. [典型协议栈示例](#典型协议栈示例)

---

## 架构概览

### 核心设计理念

Mitmproxy 的协议层采用了 **分层嵌套架构**，通过一系列相互嵌套的 `Layer` 对象来处理不同层级的网络协议。这种设计实现了：

- **协议无关性**：每一层只负责处理特定协议，与其他层解耦
- **动态可扩展性**：可以在运行时动态添加/替换协议层
- **事件驱动**：所有 IO 操作都通过事件机制传递

### 主要组件关系

```
┌─────────────────────────────────────────────────────────────────┐
│                        ConnectionHandler                          │
│  (服务器事件循环，负责 IO 调度、命令执行、事件分发)                │
└───────────────────────────┬─────────────────────────────────────┘
                            │ 事件 (Events)
                            ▼
┌─────────────────────────────────────────────────────────────────┐
│                         Layer Stack                               │
│  ┌─────────────────────────────────────────────────────────┐   │
│  │  Mode Layer (ReverseProxy / HttpProxy / Socks5Proxy)   │   │
│  │  - 处理代理模式特定逻辑                                   │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                   │
│  ┌───────────────────────────▼─────────────────────────────┐   │
│  │  Security Layer (TLSLayer / QuicLayer)                 │   │
│  │  - 处理 TLS/DTLS/QUIC 加密握手                         │   │
│  │  - 解密后的数据传递给子层                                │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                   │
│  ┌───────────────────────────▼─────────────────────────────┐   │
│  │  Application Layer (HttpLayer / WebsocketLayer)         │   │
│  │  - 处理 HTTP/1.1, HTTP/2, HTTP/3, WebSocket           │   │
│  └───────────────────────────┬─────────────────────────────┘   │
│                              │                                   │
│  ┌───────────────────────────▼─────────────────────────────┐   │
│  │  Transport Layer (TCPLayer / UDPLayer / DNSLayer)      │   │
│  │  - 原始字节流转发                                         │   │
│  └─────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────┘
```

### 核心文件结构

| 文件路径 | 职责描述 |
|---------|---------|
| `mitmproxy/proxy/layer.py` | Layer 基类定义，事件处理核心机制 |
| `mitmproxy/proxy/events.py` | 事件类型定义 |
| `mitmproxy/proxy/commands.py` | 命令类型定义 |
| `mitmproxy/proxy/tunnel.py` | TunnelLayer 隧道协议基类 |
| `mitmproxy/proxy/server.py` | 连接处理器和事件循环 |
| `mitmproxy/addons/next_layer.py` | 协议层决策器 |
| `mitmproxy/proxy/layers/tls.py` | TLS/DTLS 协议层 |
| `mitmproxy/proxy/layers/http/` | HTTP 协议层实现 |
| `mitmproxy/proxy/layers/websocket.py` | WebSocket 协议层 |
| `mitmproxy/proxy/layers/tcp.py` | TCP 原始转发层 |

---

## 协议处理单元的嵌套机制

### Layer 基类设计

所有协议处理单元都继承自 `Layer` 基类，定义在 `layer.py:44`。

#### 核心属性

```python
class Layer:
    context: Context                    # 连接上下文，包含 client/server 连接对象
    _paused: Paused | None              # 暂停状态（等待命令回复时）
    _paused_event_queue: deque[Event]  # 暂停期间的事件缓冲队列
    debug: str | None                   # 调试日志前缀
```

#### 核心方法：`handle_event`

`handle_event` 方法 (`layer.py:131`) 是层与层之间通信的唯一入口，实现了类似协程的暂停/恢复机制：

```python
def handle_event(self, event: events.Event) -> CommandGenerator[None]:
    if self._paused:
        # 检查是否是我们等待的命令回复
        pause_finished = (
            isinstance(event, events.CommandCompleted)
            and event.command is self._paused.command
        )
        if pause_finished:
            yield from self.__continue(event)  # 恢复执行
        else:
            self._paused_event_queue.append(event)  # 缓冲其他事件
    else:
        # 正常处理事件
        command_generator = self._handle_event(event)
        # ... 处理生成器产出的命令
```

**关键点**：
1. **阻塞命令**：当层产出 `blocking=True` 的命令时，`handle_event` 会暂停该层的执行
2. **事件缓冲**：暂停期间收到的事件会被存入 `_paused_event_queue`
3. **恢复执行**：收到对应的 `CommandCompleted` 事件后，继续执行并重放缓冲的事件

### NextLayer：动态决策点

`NextLayer` 类 (`layer.py:248`) 是协议栈中的"决策点"，用于在运行时确定下一层协议：

```python
class NextLayer(Layer):
    layer: Layer | None           # 实际的下一层（由 addon 决定）
    events: list[Event]           # 决策前缓冲的事件
    _ask_on_start: bool           # 是否在 Start 时立即询问
```

#### 工作流程

1. **缓冲事件**：在 `_ask()` 被调用前，所有事件都被存入 `events` 列表
2. **触发 Hook**：产出 `NextLayerHook`，通知 addon 决策下一层
3. **设置子层**：addon 设置 `nextlayer.layer` 属性
4. **重放事件**：将缓冲的事件转发给新的子层
5. **透明代理**：将自身的 `handle_event` 直接指向子层的方法

```python
def _ask(self):
    yield NextLayerHook(self)  # 触发 Hook，addon 会设置 self.layer
    
    if self.layer:
        # 将缓冲的事件转发给子层
        for e in self.events:
            yield from self.layer.handle_event(e)
        self.events.clear()
        
        # 透明代理：后续事件直接转发
        self.handle_event = self.layer.handle_event
        self._handle_event = self.layer.handle_event
        self._handle = self.layer.handle_event
```

### NextLayer Addon：决策逻辑

`mitmproxy/addons/next_layer.py` 中的 `NextLayer` addon 实现了具体的协议决策逻辑：

#### 决策优先级（`_next_layer` 方法）

```
1. --ignore/--allow 主机过滤
   ↓
2. 代理模式特定处理 (ReverseProxy / HttpProxy)
   ↓
3. 安全协议检测 (TLS/DTLS/QUIC)
   ↓
4. --tcp/--udp 主机配置
   ↓
5. ALPN 协商结果检测
   ↓
6. DNS 端口检测 (53, 5353)
   ↓
7. UDP 原始转发
   ↓
8. rawtcp 模式检测
   ↓
9. 默认：HTTP 透明模式
```

#### 关键决策代码示例

```python
# TLS 检测
is_tls_or_dtls = (
    tcp_based and starts_like_tls_record(data_client)
    or udp_based and starts_like_dtls_record(data_client)
)
if is_tls_or_dtls:
    server_tls = ServerTLSLayer(context)
    server_tls.child_layer = ClientTLSLayer(context)  # 嵌套！
    return server_tls

# ALPN 协商
if context.client.alpn:
    if context.client.alpn in HTTP_ALPNS:
        return layers.HttpLayer(context, HTTPMode.transparent)
```

### TunnelLayer：隧道协议基类

`TunnelLayer` (`tunnel.py:21`) 是需要建立"隧道"的协议的基类，例如 TLS、SOCKS、HTTP CONNECT 等。

#### 核心概念

```python
class TunnelLayer(layer.Layer):
    child_layer: layer.Layer           # 内层协议层
    tunnel_connection: Connection      # 外层隧道连接
    conn: Connection                   # 内层逻辑连接
    tunnel_state: TunnelState          # 隧道状态机
```

#### 状态机设计

| 状态 | 含义 |
|-----|------|
| `INACTIVE` | 未激活 |
| `ESTABLISHING` | 握手进行中 |
| `OPEN` | 隧道已建立，数据透明传输 |
| `CLOSED` | 已关闭 |

#### 数据流向

```
┌──────────────────────────────────────────────────────────────┐
│                      TunnelLayer                               │
│                                                               │
│  隧道建立前 (ESTABLISHING):                                   │
│    DataReceived → receive_handshake_data() → 握手处理        │
│                                                               │
│  隧道建立后 (OPEN):                                           │
│    DataReceived                                                │
│         ↓                                                      │
│    receive_data()                                              │
│         ↓                                                      │
│    event_to_child(DataReceived(conn, plaintext))              │
│         ↓                                                      │
│    ┌──────────────────────────────────────────────────────┐   │
│    │  child_layer (如 HttpLayer)                          │   │
│    │  - 处理解密后的明文数据                                │   │
│    └──────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────┘
```

#### 双向数据转换

子层产出的命令会通过 `_handle_command` 方法转换：

```python
def _handle_command(self, command: commands.Command):
    if isinstance(command, commands.SendData) and command.connection == self.conn:
        # 子层要发送数据 → 调用 send_data() 进行加密/封装
        yield from self.send_data(command.data)
    elif isinstance(command, commands.CloseConnection):
        yield from self.send_close(command)
```

### LayerStack：优雅的层构建语法

`LayerStack` (`tunnel.py:185`) 提供了使用 `/` 运算符构建协议栈的语法糖：

```python
stack = tunnel.LayerStack()
stack /= ServerTLSLayer(context)      # 外层：服务端 TLS
stack /= ClientTLSLayer(context)      # 中层：客户端 TLS
stack /= HttpLayer(context, mode)     # 内层：HTTP 处理

return stack[0]  # 返回最外层
```

这实际上会自动设置各层的 `child_layer` 属性：

```python
def __truediv__(self, other: Layer | LayerStack) -> LayerStack:
    if isinstance(other, Layer):
        if self._stack:
            self._stack[-1].child_layer = other  # 自动嵌套！
        self._stack.append(other)
    return self
```

### 典型嵌套结构示例

#### HTTPS 请求的协议栈

```
NextLayer (初始决策点)
  ↓ 设置 layer
ServerTLSLayer (与客户端的 TLS 握手)
  ├── child_layer: ClientTLSLayer (与服务端的 TLS 握手)
  │                    ├── child_layer: NextLayer (再次决策)
  │                    │                 ↓
  │                    │            HttpLayer (HTTP 处理)
  │                    │                 ├── connections: {
  │                    │                 │     client_conn: Http1Server,
  │                    │                 │     server_conn: Http1Client
  │                    │                 │   }
  │                    │                 └── streams: {
  │                    │                       1: HttpStream (请求/响应处理)
  │                    │                     }
  │                    └── tunnel_connection: server_conn
  └── tunnel_connection: client_conn
```

---

## 网络事件传递与冒泡机制

### 事件类型体系

所有事件定义在 `events.py`，继承自 `Event` 基类：

```
Event (基类)
│
├── Start                    # 层初始化事件
│
├── ConnectionEvent          # 连接相关事件（dataclass）
│   ├── DataReceived         # 收到数据：connection + data
│   └── ConnectionClosed     # 连接关闭
│
├── CommandCompleted         # 命令完成回复
│   ├── OpenConnectionCompleted
│   ├── HookCompleted
│   └── Wakeup
│
└── MessageInjected[T]      # 用户注入消息（泛型）
    ├── TcpMessageInjected
    └── WebSocketMessageInjected
```

#### HTTP 层内部事件

HTTP 层定义了自己的事件类型 `HttpEvent` (`http/_base.py:16`)，用于在 `HttpLayer` 内部传递：

```python
@dataclass
class HttpEvent(events.Event):
    stream_id: StreamId  # 每个事件都携带流 ID，避免竞态条件
```

具体事件类型：
- `RequestHeaders` / `ResponseHeaders`：请求/响应头
- `RequestData` / `ResponseData`：请求/响应体数据
- `RequestTrailers` / `ResponseTrailers`：尾部头（HTTP/2+）
- `RequestEndOfMessage` / `ResponseEndOfMessage`：消息结束
- `RequestProtocolError` / `ResponseProtocolError`：协议错误

### 事件传递方向

事件在协议栈中有两个传递方向：

#### 1. 向下传递（Top-Down）

从最外层流向最内层，通常由 `ConnectionHandler` 触发：

```
ConnectionHandler.server_event(event)
         ↓
layer.handle_event(event)  # 最外层
         ↓
子层.handle_event(event)
         ↓
... 继续向内 ...
```

#### 2. 命令向上传递（Bottom-Up）

子层产出的命令会被父层捕获和处理：

```
子层.handle_event() 产出 Command
         ↓
父层.event_to_child() 中的循环捕获
         ↓
父层._handle_command() 处理或转换
         ↓
（可能）继续向上传递
         ↓
ConnectionHandler 执行
```

### 事件传递机制详解

#### 基础传递：`event_to_child`

`TunnelLayer.event_to_child` (`tunnel.py:146`) 是标准的事件转发实现：

```python
def event_to_child(self, event: events.Event) -> CommandGenerator[None]:
    # 建立隧道期间缓冲事件
    if self.tunnel_state is TunnelState.ESTABLISHING and not self.command_to_reply_to:
        self._event_queue.append(event)
        return
    
    # 转发给子层
    for command in self.child_layer.handle_event(event):
        yield from self._handle_command(command)  # 捕获子层的命令
```

#### HTTP 层的事件路由

`HttpLayer` 内部有更复杂的事件路由逻辑 (`http/__init__.py:951`)：

```python
def _handle_event(self, event: events.Event):
    if isinstance(event, events.Start):
        # 根据 ALPN 选择 HTTP 版本
        if is_h3_alpn(self.context.client.alpn):
            http_conn = Http3Server(...)
        elif self.context.client.alpn == b"h2":
            http_conn = Http2Server(...)
        else:
            http_conn = Http1Server(...)
        
        self.connections[self.context.client] = http_conn
        yield from self.event_to_child(http_conn, event)
    
    elif isinstance(event, events.ConnectionEvent):
        # 根据连接找到对应的处理层
        handler = self.connections[event.connection]
        yield from self.event_to_child(handler, event)
```

#### `HttpLayer.event_to_child`：命令转换

`HttpLayer` 的 `event_to_child` 方法 (`http/__init__.py:1021`) 实现了关键的协议转换：

```
┌──────────────────────────────────────────────────────────────────┐
│                         HttpLayer                                 │
│                                                                  │
│  ConnectionEvent (原始字节)                                       │
│       ↓                                                          │
│  connections[conn].handle_event()                                │
│       ↓                                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ Http1Server / Http2Server / Http3Server                  │  │
│  │  - 解析字节为 HTTP 消息                                    │  │
│  │  - 产出 ReceiveHttp(HttpEvent)                           │  │
│  └──────────────────────────────────────────────────────────┘  │
│       ↓                                                          │
│  ReceiveHttp 被捕获                                              │
│       ↓                                                          │
│  streams[stream_id].handle_event(HttpEvent)                     │
│       ↓                                                          │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │ HttpStream                                                 │  │
│  │  - 处理请求/响应逻辑                                       │  │
│  │  - 产出 SendHttp(HttpEvent)                               │  │
│  └──────────────────────────────────────────────────────────┘  │
│       ↓                                                          │
│  SendHttp 被捕获                                                 │
│       ↓                                                          │
│  connections[conn].handle_event(HttpEvent)                       │
│       ↓                                                          │
│  产出 SendData(原始字节)                                          │
└──────────────────────────────────────────────────────────────────┘
```

关键代码：

```python
def event_to_child(self, child: Layer | HttpStream, event: Event):
    for command in child.handle_event(event):
        if isinstance(command, ReceiveHttp):
            # HTTP 连接层产出的 ReceiveHttp → 转发给对应 Stream
            if isinstance(command.event, RequestHeaders):
                yield from self.make_stream(command.event.stream_id)
            stream = self.streams.get(command.event.stream_id)
            if stream:
                yield from self.event_to_child(stream, command.event)
        
        elif isinstance(command, SendHttp):
            # Stream 产出的 SendHttp → 转发给对应连接层
            conn = self.connections[command.connection]
            yield from self.event_to_child(conn, command.event)
        
        elif isinstance(command, commands.Command):
            # 其他命令直接向上传递
            yield command
```

### 事件冒泡：子层命令的处理

命令的"冒泡"是通过父层的 `_handle_command` 或 `event_to_child` 中的捕获逻辑实现的。

#### 示例：TLS 层的加密/解密

```
┌─────────────────────────────────────────────────────────────┐
│                      TLSLayer                                │
│                                                              │
│  接收 DataReceived(client_conn, encrypted_data)             │
│       ↓                                                      │
│  self.tls.bio_write(encrypted_data)                         │
│  self.tls.recv() → plaintext                                │
│       ↓                                                      │
│  event_to_child(                                            │
│      DataReceived(conn, plaintext)  # 解密后的数据         │
│  )                                                          │
│       ↓                                                      │
│  ┌─────────────────────────────────────────────────────┐   │
│  │  child_layer (如 HttpLayer)                          │   │
│  │       ↓                                               │   │
│  │  产出 SendData(conn, response_plaintext)             │   │
│  └─────────────────────────────────────────────────────┘   │
│       ↓                                                      │
│  _handle_command 捕获 SendData                               │
│       ↓                                                      │
│  self.tls.sendall(response_plaintext)                       │
│  self.tls.bio_read() → encrypted_response                   │
│       ↓                                                      │
│  产出 SendData(tunnel_connection, encrypted_response)       │
└─────────────────────────────────────────────────────────────┘
```

关键代码 (`tls.py:408-458`)：

```python
def receive_data(self, data: bytes) -> CommandGenerator[None]:
    # 解密
    if data:
        self.tls.bio_write(data)
    
    plaintext = bytearray()
    while True:
        try:
            plaintext.extend(self.tls.recv(65535))
        except SSL.WantReadError:
            break
    
    if plaintext:
        # 解密后的数据传递给子层
        yield from self.event_to_child(
            events.DataReceived(self.conn, bytes(plaintext))
        )

def send_data(self, data: bytes) -> CommandGenerator[None]:
    # 加密
    self.tls.sendall(data)
    yield from self.tls_interact()  # 产出 SendData
```

---

## 协议升级与无缝切换机制

Mitmproxy 支持多种协议升级场景，实现了在同一连接上从一种协议无缝切换到另一种协议。

### 协议升级类型

| 升级类型 | 触发条件 | 实现位置 |
|---------|---------|---------|
| HTTP CONNECT 隧道 | `CONNECT` 请求 + 200 响应 | `HttpStream.handle_connect()` |
| WebSocket | 101 响应 + `Upgrade: websocket` | `HttpStream.flow_done()` |
| 原始 TCP 隧道 | 101 响应 + rawtcp 选项启用 | `HttpStream.flow_done()` |
| HTTP 版本协商 | TLS ALPN 扩展 | `HttpLayer._handle_event()` |

### HTTP CONNECT 隧道机制

#### 工作流程

```
1. 客户端发送 CONNECT 请求
   CONNECT example.com:443 HTTP/1.1
   Host: example.com:443

2. HttpStream 处理 CONNECT
   state_wait_for_request_headers
         ↓
   handle_connect()

3. 建立隧道
   - 设置 server.address
   - 创建 NextLayer 作为子层
   - 切换到 passthrough 模式

4. 返回 200 响应
   HTTP/1.1 200 Connection established

5. 透明转发后续数据
   客户端数据 → TLS 握手 → 服务端
```

#### 关键代码 (`http/__init__.py:773`)

```python
def handle_connect(self) -> CommandGenerator[None]:
    self.client_state = self.state_done
    yield HttpConnectHook(self.flow)
    
    # 设置目标地址
    self.context.server.address = (self.flow.request.host, self.flow.request.port)
    
    # 创建 NextLayer，让它决定下一层协议（通常是 TLS）
    self.child_layer = layer.NextLayer(self.context)
    
    # 发送 200 响应
    self.flow.response = http.Response(..., 200, b"Connection established", ...)
    
    # 切换到 passthrough 模式
    self._handle_event = self.passthrough
    yield from self.child_layer.handle_event(events.Start())
```

#### `passthrough` 模式：协议透明转换

`passthrough` 方法 (`http/__init__.py:848`) 实现了 HTTP 事件与原始连接事件的双向转换。**关键在于：CONNECT 隧道（200 响应）与协议升级（101 响应）在服务端方向的处理逻辑完全不同**。

##### 核心差异分析

代码中的关键条件是 `self.flow.response.status_code == 101`：

```python
def passthrough(self, event: events.Event) -> CommandGenerator[None]:
    assert self.flow.response
    assert self.child_layer
    
    # ==========================================
    # 第一部分：HTTP 事件 → 原始连接事件
    # （这部分对 CONNECT 和 WebSocket 是相同的）
    # ==========================================
    if isinstance(event, RequestData):
        event = events.DataReceived(self.context.client, event.data)
    elif isinstance(event, ResponseData):
        event = events.DataReceived(self.context.server, event.data)
    elif isinstance(event, RequestEndOfMessage):
        event = events.ConnectionClosed(self.context.client)
    elif isinstance(event, ResponseEndOfMessage):
        event = events.ConnectionClosed(self.context.server)

    # ==========================================
    # 第二部分：原始连接命令 → HTTP 事件（或直接透传）
    # （这部分对 CONNECT 和 WebSocket 有本质区别）
    # ==========================================
    for command in self.child_layer.handle_event(event):
        if isinstance(command, commands.SendData):
            # 客户端方向：总是封装为 HTTP 事件
            if command.connection == self.context.client:
                yield SendHttp(
                    ResponseData(self.stream_id, command.data), self.context.client
                )
            # 服务端方向：检查是否为协议升级（101）
            elif (
                command.connection == self.context.server
                and self.flow.response.status_code == 101  # 关键条件！
            ):
                # WebSocket 等协议升级：服务端连接也是 HTTP 层管理的
                # 需要封装为 HTTP 事件
                yield SendHttp(
                    RequestData(self.stream_id, command.data), self.context.server
                )
            else:
                # CONNECT 隧道（200 响应）：服务端连接不是 HTTP 层管理的
                # 直接透传命令，不封装为 HTTP 事件
                yield command
        
        elif isinstance(command, commands.CloseConnection):
            # 客户端方向：总是封装为 HTTP 事件
            if command.connection == self.context.client:
                yield SendHttp(
                    ResponseProtocolError(
                        self.stream_id, "EOF", ErrorCode.PASSTHROUGH_CLOSE
                    ),
                    self.context.client,
                )
            # 服务端方向：检查是否为协议升级（101）
            elif (
                command.connection == self.context.server
                and self.flow.response.status_code == 101  # 同样的关键条件！
            ):
                # WebSocket 等协议升级：封装为 HTTP 事件
                yield SendHttp(
                    RequestProtocolError(
                        self.stream_id, "EOF", ErrorCode.PASSTHROUGH_CLOSE
                    ),
                    self.context.server,
                )
            else:
                # CONNECT 隧道：直接透传
                if isinstance(command, commands.CloseTcpConnection):
                    command = commands.CloseConnection(command.connection)
                yield command
        
        else:
            yield command
```

##### 差异原因（代码注释说明）

```python
# http/__init__.py:872-873
# there only is a HTTP server connection if we have switched protocols,
# not if a connection is established via CONNECT.
```

**解释**：
- **协议升级（WebSocket 等，101 响应）**：服务端连接是通过 `HttpLayer` 的 `HttpClient` 机制管理的（`Http1Client`/`Http2Client`），所以子层产出的命令需要封装为 `SendHttp` 事件，由 HTTP 连接层处理。
- **CONNECT 隧道（200 响应）**：服务端连接是通过 `NextLayer` 全新建立的独立连接（如 `TLSLayer` 包裹的原始 TCP 连接），不经过 HTTP 层管理，所以命令直接透传给上层 `ConnectionHandler` 执行。

##### 数据流对比

| 方向 | WebSocket 升级 (101) | CONNECT 隧道 (200) |
|-----|---------------------|---------------------|
| **客户端数据流入** | `RequestData` → `DataReceived(client)` | 相同 |
| **服务端数据流入** | `ResponseData` → `DataReceived(server)` | 相同 |
| **客户端数据流出** | `SendData(client)` → `SendHttp(ResponseData)` | 相同 |
| **服务端数据流出** | `SendData(server)` → `SendHttp(RequestData)` | `SendData(server)` → **直接透传** |
| **客户端关闭** | `CloseConnection(client)` → `SendHttp(ResponseProtocolError)` | 相同 |
| **服务端关闭** | `CloseConnection(server)` → `SendHttp(RequestProtocolError)` | `CloseConnection(server)` → **直接透传** |

### WebSocket 升级机制

#### 触发条件检查 (`http/__init__.py:499`)

```python
is_websocket = (
    self.flow.response.status_code == 101
    and self.flow.response.headers.get("upgrade", "").lower() == "websocket"
    and self.flow.request.headers.get("Sec-WebSocket-Version", "").encode()
        == wsproto.handshake.WEBSOCKET_VERSION
    and self.context.options.websocket
)
```

#### 升级流程

```
1. 客户端发送升级请求
   GET /ws HTTP/1.1
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Key: ...
   Sec-WebSocket-Version: 13

2. 服务端返回 101 响应
   HTTP/1.1 101 Switching Protocols
   Upgrade: websocket
   Connection: Upgrade
   Sec-WebSocket-Accept: ...

3. HttpStream.flow_done() 检测到 101
         ↓
   创建 WebsocketLayer
         ↓
   切换到 passthrough 模式
         ↓
   发送 Start 事件给 WebsocketLayer

4. WebsocketLayer 接管
   - 解析 WebSocket 帧
   - 拦截/修改消息
   - 双向转发
```

#### 关键代码 (`http/__init__.py:537`)

```python
def flow_done(self) -> CommandGenerator[None]:
    if self.flow.response.status_code == 101:
        if self.flow.websocket:
            # 创建 WebSocket 层
            self.child_layer = websocket.WebsocketLayer(self.context, self.flow)
        elif self.context.options.rawtcp:
            # 回退到原始 TCP
            self.child_layer = tcp.TCPLayer(self.context)
        else:
            # 无协议可升级
            yield commands.Log(
                f"Sent HTTP 101 response, but no protocol is enabled to upgrade to.",
                WARNING
            )
            return
        
        # 切换到 passthrough 模式
        self._handle_event = self.passthrough
        yield from self.child_layer.handle_event(events.Start())
```

### HTTP 版本协商（ALPN）

#### 基于 ALPN 的版本选择

`HttpLayer` 在收到 `Start` 事件时根据 ALPN 选择 HTTP 版本 (`http/__init__.py:951`)：

```python
def _handle_event(self, event: events.Event):
    if isinstance(event, events.Start):
        if is_h3_alpn(self.context.client.alpn):
            # HTTP/3 over QUIC
            http_conn = Http3Server(self.context.fork())
        elif self.context.client.alpn == b"h2":
            # HTTP/2
            http_conn = Http2Server(self.context.fork())
        else:
            # 默认 HTTP/1.1
            http_conn = Http1Server(self.context.fork())
        
        self.connections.setdefault(self.context.client, http_conn)
        yield from self.event_to_child(self.connections[self.context.client], event)
```

#### ALPN 协商发生位置

ALPN 协商发生在 TLS 握手期间，由 `TLSLayer` 完成：

```python
# tls.py:380
self.conn.alpn = self.tls.get_alpn_proto_negotiated()
```

然后 `NextLayer` addon 会检查这个 `alpn` 属性：

```python
# next_layer.py:168
if context.client.alpn:
    if context.client.alpn in HTTP_ALPNS:  # h3, h2, http/1.1, 等
        return layers.HttpLayer(context, HTTPMode.transparent)
```

#### 服务端连接的 HTTP 版本选择

建立服务端连接时也会根据 ALPN 选择客户端实现 (`http/__init__.py:1202`)：

```python
class HttpClient(layer.Layer):
    def _handle_event(self, event: events.Event):
        # ... 建立连接后
        if is_h3_alpn(self.context.server.alpn):
            self.child_layer = Http3Client(self.context)
        elif self.context.server.alpn == b"h2":
            self.child_layer = Http2Client(self.context)
        else:
            self.child_layer = Http1Client(self.context)
```

### HTTP/1.x 与 HTTP/2 的自动转换

`Http1Server`/`Http1Client` 在 `send()` 方法中自动处理 HTTP 版本转换：

#### HTTP/2 → HTTP/1 响应转换 (`_http1.py:235`)

```python
def send(self, event: HttpEvent):
    if isinstance(event, ResponseHeaders):
        self.response = response = event.response
        
        if response.is_http2 or response.is_http3:
            response = response.copy()
            response.http_version = "HTTP/1.1"
            response.reason = status_codes.RESPONSES.get(response.status_code, "")
        
        raw = http1.assemble_response_head(response)
        yield commands.SendData(self.conn, raw)
```

#### HTTP/2 → HTTP/1 请求转换 (`_http1.py:349`)

```python
def send(self, event: HttpEvent):
    if isinstance(event, RequestHeaders):
        request = event.request
        if request.is_http2 or request.is_http3:
            request = request.copy()
            request.http_version = "HTTP/1.1"
            if "Host" not in request.headers and request.authority:
                request.headers.insert(0, "Host", request.authority)
            request.authority = ""
        
        raw = http1.assemble_request_head(request)
        yield commands.SendData(self.conn, raw)
```

---

## 命令系统与事件的双向通信

### 设计理念

事件和命令是 mitmproxy 协议层的两个核心概念，形成双向通信：

```
┌─────────────────────────────────────────────────────────────┐
│                      ConnectionHandler                        │
│  (运行在 asyncio 事件循环中)                                  │
│                                                              │
│  IO 发生                                                       │
│    │                                                          │
│    ▼                                                          │
│  包装为 Event ──────────────────────────────────┐            │
│                                                  │            │
│                                                  ▼            │
│                                        ┌─────────────────┐   │
│                                        │   Layer Stack   │   │
│                                        │  handle_event() │   │
│                                        └────────┬────────┘   │
│                                                 │             │
│                                                 ▼             │
│                                        yield Command ◄────────┘
│                                           │
│                                           ▼
│                                    执行 Command
│                                    - OpenConnection
│                                    - SendData
│                                    - StartHook (触发 addon)
│                                    - RequestWakeup
│                                           │
│                                           ▼
│                              完成后包装为 CommandCompleted
│                                           │
│                                           ▼
│                              再次调用 layer.handle_event()
```

### 命令类型体系

定义在 `commands.py`：

```
Command (基类)
│  ┌─ blocking: bool | Layer  # 是否阻塞
│  │                           # True = 阻塞等待完成
│  │                           # Layer 引用 = 已被某层处理，外层不阻塞
│
├── RequestWakeup        # 请求延时唤醒
│
├── ConnectionCommand     # 连接相关命令
│   ├── SendData          # 发送数据
│   ├── OpenConnection    # 打开连接（阻塞）
│   ├── CloseConnection   # 关闭连接
│   └── CloseTcpConnection # 半关闭
│
├── StartHook             # 触发 Hook（阻塞）
│   ├── TlsStartClientHook
│   ├── HttpRequestHook
│   └── ...
│
└── Log                   # 日志记录
```

### 阻塞命令的处理机制

#### 核心逻辑 (`layer.py:131`)

当层产出 `blocking=True` 的命令时：

```python
def handle_event(self, event: events.Event):
    # ...
    command_generator = self._handle_event(event)
    
    while True:
        if command.blocking is True:
            # 将 blocking 标记为当前层，通知外层"此命令已被处理"
            command.blocking = self
            
            # 保存状态，暂停执行
            self._paused = Paused(command, command_generator)
            
            yield command
            return  # 暂时返回，等待 CommandCompleted
```

#### 阻塞标记的巧妙设计

`blocking` 属性可以是 `True` 或 `Layer` 引用：

```python
# 层内：设置为 True 表示需要阻塞
yield GetHttpConnection(...)  # blocking = True

# handle_event 中：转换为层引用
if command.blocking is True:
    command.blocking = self  # 现在 blocking 是一个 Layer

# 外层检查：只关心 `is True`
# 如果是 Layer 引用，表示已被内层处理，外层不阻塞
```

这使得 HTTP/2 等多路复用协议可以只阻塞单个流而不阻塞整个连接：

```python
# 注释中的例子 (layer.py:165)
# HTTP/2 连接中，如果我们拦截一个特定请求，
# 我们不希望连接中的所有其他请求也被阻塞。
# 通过将 blocking 设置为层引用，外层知道此命令已被处理。
```

#### 恢复执行 (`layer.py:224`)

收到 `CommandCompleted` 事件后：

```python
def __continue(self, event: events.CommandCompleted):
    # 恢复生成器
    command_generator = self._paused.generator
    self._paused = None
    
    # send() 方法将 reply 传回生成器
    yield from self.__process(command_generator, event.reply)
    
    # 处理暂停期间缓冲的事件
    while not self._paused and self._paused_event_queue:
        ev = self._paused_event_queue.popleft()
        yield from self._handle_event(ev)
```

### ConnectionHandler 中的命令执行

`server.py` 中的 `ConnectionHandler` 是命令的最终执行者：

```python
async def server_event(self, event: events.Event) -> None:
    layer_commands = self.layer.handle_event(event)
    
    for command in layer_commands:
        if isinstance(command, commands.OpenConnection):
            # 异步打开连接
            handler = asyncio_utils.create_task(
                self.open_connection(command), ...
            )
        
        elif isinstance(command, commands.SendData):
            # 直接写入 socket
            writer = self.transports[command.connection].writer
            writer.write(command.data)
        
        elif isinstance(command, commands.CloseConnection):
            self.close_connection(command.connection)
        
        elif isinstance(command, commands.StartHook):
            # 异步触发 addon hook
            asyncio_utils.create_task(self.hook_task(command), ...)
        
        elif isinstance(command, commands.Log):
            self.log(command.message, command.level)
        
        elif isinstance(command, commands.RequestWakeup):
            # 创建延时任务
            task = asyncio_utils.create_task(self.wakeup(command), ...)
            self.wakeup_timer.add(task)
```

### 典型命令-事件交互流程

#### 示例：建立服务端连接

```
1. HttpStream 需要建立连接
   def make_server_connection():
       connection, err = yield GetHttpConnection(...)  # blocking=True

2. handle_event 暂停 HttpStream
   self._paused = Paused(GetHttpConnection, generator)
   command.blocking = HttpStream 实例
   yield command

3. HttpLayer 捕获（不阻塞，因为 blocking is not True）
   def event_to_child(child, event):
       for command in child.handle_event(event):
           if command.blocking or isinstance(command, RequestWakeup):
               self.command_sources[command] = child  # 记录来源
           
           if isinstance(command, GetHttpConnection):
               yield from self.get_connection(command)

4. HttpLayer.get_connection() 实际建立连接
   - 可能创建新的 LayerStack (TLSLayer + HttpClient)
   - yield commands.OpenConnection (真正的阻塞命令)

5. ConnectionHandler 执行 OpenConnection
   - asyncio.open_connection()
   - 完成后触发 OpenConnectionCompleted

6. 原路返回
   - HttpClient 收到 OpenConnectionCompleted
   - HttpClient 选择 HTTP 版本 (Http1/2/3Client)
   - 产出 RegisterHttpConnection

7. HttpLayer 收到 RegisterHttpConnection
   def register_connection(command):
       for cmd in waiting:
           stream = self.command_sources.pop(cmd)
           yield from self.event_to_child(
               stream, 
               GetHttpConnectionCompleted(cmd, (connection, None))
           )

8. HttpStream 恢复执行
   __continue(event)
   connection, err = event.reply  # (connection, None)
   # 继续执行...
```

---

## 典型协议栈示例

### 场景 1：HTTPS 反向代理

#### 配置
```bash
mitmproxy --mode reverse:https://example.com
```

#### 协议栈构建流程

```
1. 客户端连接，ReverseProxy 作为第一层
   context.layers = [ReverseProxy]

2. ReverseProxy._handle_event()
   spec.scheme = "https"
   self.context.server.address = ("example.com", 443)
   self.child_layer = NextLayer(context)
   yield from self.child_layer.handle_event(Start())

3. NextLayerHook 触发，NextLayer addon 决策
   _next_layer() 检测：
   - stack_match([ReverseProxy]) → True
   - 调用 _setup_reverse_proxy()

4. _setup_reverse_proxy() 构建协议栈
   spec.scheme = "https"
   stack = LayerStack()
   stack /= ServerTLSLayer(context)      # 与客户端的 TLS
   stack /= ClientTLSLayer(context)      # 与服务端的 TLS
   stack /= HttpLayer(context, transparent)
   return stack[0]  # ServerTLSLayer

5. 此时的层嵌套
   ReverseProxy.child_layer = ServerTLSLayer
   ServerTLSLayer.child_layer = ClientTLSLayer
   ClientTLSLayer.child_layer = NextLayer  (内部的，由 TunnelLayer 创建)

6. ClientTLSLayer 握手完成后
   self.conn.alpn = b"h2"  # 假设协商为 HTTP/2
   然后会再次触发 NextLayerHook...

7. 最终的完整栈
   ReverseProxy
     └── ServerTLSLayer (client_conn, 已握手)
           └── ClientTLSLayer (server_conn, 已握手)
                 └── HttpLayer (mode=transparent)
                       ├── connections: {
                       │     client_conn: Http2Server,
                       │     server_conn: Http2Client
                       │   }
                       └── streams: {
                             1: HttpStream,
                             3: HttpStream,
                             ...
                           }
```

### 场景 2：HTTP CONNECT + TLS + WebSocket

#### 数据流

```
客户端                              mitmproxy                              服务端
  │                                    │                                    │
  │ 1. CONNECT example.com:443        │                                    │
  │───────────────────────────────────>│                                    │
  │                                    │ 2. HttpStream.handle_connect()      │
  │                                    │    - 创建 NextLayer                 │
  │                                    │    - 检测到将使用 TLS               │
  │                                    │                                    │
  │ 3. HTTP/1.1 200 Established       │                                    │
  │<───────────────────────────────────│                                    │
  │                                    │                                    │
  │ 4. TLS ClientHello                 │                                    │
  │───────────────────────────────────>│ 5. ClientTLSLayer 握手             │
  │                                    │    - 解析 SNI=example.com          │
  │                                    │    - 伪造证书                        │
  │                                    │                                    │
  │ 6. TLS ServerHello + Cert          │                                    │
  │<───────────────────────────────────│                                    │
  │    ... 握手完成 ...                 │                                    │
  │                                    │ 7. 再次 NextLayerHook               │
  │                                    │    alpn = b"http/1.1"               │
  │                                    │    → HttpLayer                      │
  │                                    │                                    │
  │ 8. GET /ws HTTP/1.1                │                                    │
  │    Upgrade: websocket              │                                    │
  │    (TLS 加密)                       │                                    │
  │───────────────────────────────────>│ 9. ServerTLSLayer 解密              │
  │                                    │    → Http1Server 解析               │
  │                                    │    → HttpStream 处理                │
  │                                    │                                    │
  │                                    │ 10. 建立与服务端的连接               │
  │                                    │     - ServerTLSLayer 握手           │
  │                                    │     - 发送升级请求                   │
  │                                    │────────────────────────────────────>│
  │                                    │                                    │
  │                                    │ 11. 收到 101 响应                   │
  │                                    │<────────────────────────────────────│
  │                                    │                                    │
  │                                    │ 12. HttpStream.flow_done()          │
  │                                    │     - 检测 101 + websocket          │
  │                                    │     - 创建 WebsocketLayer           │
  │                                    │     - 切换到 passthrough            │
  │                                    │                                    │
  │ 13. 101 Switching Protocols        │                                    │
  │<───────────────────────────────────│                                    │
  │                                    │                                    │
  │ 14. WebSocket 帧                   │                                    │
  │───────────────────────────────────>│ 15. WebsocketLayer 处理             │
  │                                    │     - 解析帧                        │
  │                                    │     - 触发 WebsocketMessageHook     │
  │                                    │     - 转发给服务端                   │
  │                                    │────────────────────────────────────>│
```

---

## 总结

### 核心设计亮点

1. **分层嵌套架构**：通过 `Layer.child_layer` 实现任意深度的协议嵌套，每层只负责单一协议

2. **生成器驱动的异步模型**：使用 `yield Command` 模拟阻塞代码，实际是非阻塞的，类似协程

3. **事件-命令双向通信**：
   - 事件（Event）：从外层向内层传递，表示"发生了什么"
   - 命令（Command）：从内层向外层产出，表示"需要做什么"

4. **动态协议决策**：`NextLayer` + Hook 机制允许在运行时根据数据特征动态选择协议层

5. **无缝协议升级**：通过 `passthrough` 模式实现 HTTP 事件与原始连接事件的透明转换

### 关键数据结构

| 组件 | 职责 | 关键方法/属性 |
|-----|------|-------------|
| `Layer` | 协议层基类 | `handle_event()`, `_handle_event()`, `_paused` |
| `NextLayer` | 决策点占位符 | `_ask()`, `layer`, `events` |
| `TunnelLayer` | 隧道协议基类 | `event_to_child()`, `_handle_command()`, `tunnel_state` |
| `LayerStack` | 协议栈构建器 | `__truediv__()` (使用 `/` 运算符) |
| `HttpLayer` | HTTP 多路复用层 | `connections`, `streams`, `event_to_child()` |
| `HttpStream` | 单个 HTTP 流 | `passthrough()`, `flow_done()` |

### 扩展点

mitmproxy 的协议层架构设计了多个扩展点：

1. **`NextLayerHook`**：通过 addon 完全自定义协议层决策
2. **各种 `Hook` 命令**：在协议处理的关键节点注入自定义逻辑
3. **继承 `Layer` 或 `TunnelLayer`**：实现新的协议处理层
4. **`rawtcp` 选项**：允许非 HTTP 协议的透明转发

这种设计使得 mitmproxy 不仅是一个 HTTP 代理，更是一个通用的协议分析和修改框架。
