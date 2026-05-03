# mitmproxy 多入口流量分发架构分析

## 概述

mitmproxy 采用了一套精心设计的多入口流量分发架构，支持多种代理模式（regular、透明代理、SOCKS5、反向代理、WireGuard、DNS 等），同时保持了统一的核心处理流程。本文档详细分析这些模式的流量入口路径以及它们如何汇入统一的核心处理流程。

---

## 一、架构总览

### 1.1 核心模块层次结构

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           用户配置层 (Options)                                │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐         │
│  │  mode    │ │listen_host││listen_port││  server  │ │  ...     │         │
│  └──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                         模式解析层 (mode_specs.py)                            │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  ProxyMode.parse("regular@8080") → RegularMode                          │ │
│  │  ProxyMode.parse("socks5@1080") → Socks5Mode                            │ │
│  │  ProxyMode.parse("reverse:https://example.com") → ReverseMode           │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        服务器实例层 (mode_servers.py)                          │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │RegularInstance││Socks5Instance││TransparentInst││WireGuardInst │           │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘           │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────┐ ┌─────────────┐           │
│  │ReverseInst  ││UpstreamInst ││  DnsInstance ││  TunInstance │           │
│  └─────────────┘ └─────────────┘ └─────────────┘ └─────────────┘           │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                      连接处理层 (server.py: ConnectionHandler)                 │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │                    ProxyConnectionHandler.handle_client()                 │ │
│  │  ┌─────────┐   ┌─────────────────┐   ┌─────────────────────────────┐    │ │
│  │  │ClientConn│ → │  事件/命令循环   │ ← │  ServerConnection (上游)   │    │ │
│  │  └─────────┘   └─────────────────┘   └─────────────────────────────┘    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                          层协议栈 (layers/)                                    │
│  ┌─────────────────────────────────────────────────────────────────────────┐ │
│  │  顶层模式层: HttpProxy, Socks5Proxy, TransparentProxy, ReverseProxy...  │ │
│  │                              ↓                                            │ │
│  │  NextLayer (决策点) → next_layer addon 决定下一层                         │ │
│  │                              ↓                                            │ │
│  │  核心协议层: HttpLayer, TLSLayer, TCPLayer, UDPLayer, DNSLayer...       │ │
│  │                              ↓                                            │ │
│  │  应用层: HttpStream, WebsocketLayer 等                                    │ │
│  └─────────────────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 1.2 关键设计模式

| 设计模式 | 应用场景 | 代码位置 |
|---------|---------|---------|
| **工厂模式** | `ServerInstance.make()` 根据模式类型创建对应服务器实例 | `mode_servers.py:110-122` |
| **责任链模式** | 层协议栈通过 `handle_event` 传递事件和命令 | `layer.py` |
| **策略模式** | `next_layer` addon 根据上下文动态决定下一层 | `addons/next_layer.py` |
| **状态机模式** | 各层内部使用状态机管理协议处理流程 | `layers/modes.py:Socks5Proxy` |

---

## 二、代理模式类型详解

### 2.1 模式定义 (mode_specs.py)

所有代理模式都继承自 `ProxyMode` 基类，通过 `__init_subclass__` 自动注册：

```python
# mode_specs.py:68-71
def __init_subclass__(cls, **kwargs):
    cls.type_name = cls.__name__.removesuffix("Mode").lower()
    assert cls.type_name not in ProxyMode.__types
    ProxyMode.__types[cls.type_name] = cls
```

### 2.2 完整模式列表

| 模式类型 | 类名 | 描述 | 默认端口 | 传输协议 |
|---------|------|------|---------|---------|
| **regular** | `RegularMode` | 标准 HTTP(S) 代理，使用 CONNECT 或绝对 URL | 8080 | TCP |
| **transparent** | `TransparentMode` | 透明代理，无需客户端配置 | 8080 | TCP |
| **socks5** | `Socks5Mode` | SOCKS v5 代理 | 1080 | TCP |
| **reverse** | `ReverseMode` | 反向代理，转发到固定目标 | 8080/53 | TCP/UDP/Both |
| **upstream** | `UpstreamMode` | 链式代理，转发到另一个 HTTP 代理 | 8080 | TCP |
| **dns** | `DnsMode` | DNS 服务器/代理 | 53 | Both |
| **wireguard** | `WireGuardMode` | WireGuard VPN 模式 | 51820 | UDP |
| **local** | `LocalMode` | 操作系统级透明重定向 | None | Both |
| **tun** | `TunMode` | TUN 虚拟网络接口 | None | Both |

### 2.3 模式规格解析语法

```
模式 [: 模式配置] [@ [监听地址:]监听端口]
```

**示例：**
- `regular` - 标准代理，默认端口 8080
- `socks5@1080` - SOCKS5 代理，端口 1080
- `reverse:https://example.com@127.0.0.1:443` - 反向代理到 example.com，监听 localhost:443

---

## 三、各模式流量入口路径分析

### 3.1 服务器实例层次结构

```
ServerInstance (抽象基类)
├── AsyncioServerInstance (异步服务器基类)
│   ├── RegularInstance      → 标准 HTTP 代理
│   ├── UpstreamInstance     → 上游代理模式
│   ├── TransparentInstance  → 透明代理
│   ├── ReverseInstance      → 反向代理
│   ├── Socks5Instance       → SOCKS5 代理
│   ├── DnsInstance          → DNS 服务器
│   └── WireGuardServerInstance → WireGuard 模式
├── LocalRedirectorInstance  → 本地重定向
└── TunInstance              → TUN 接口
```

### 3.2 各模式入口详解

#### 3.2.1 Regular 模式（标准 HTTP 代理）

**入口路径：**

```
1. 用户配置: --mode regular 或 --mode regular@8080
         ↓
2. 模式解析: ProxyMode.parse("regular") → RegularMode
         ↓
3. 服务器创建: ServerInstance.make() → RegularInstance
         ↓
4. 监听端口: asyncio.start_server(handle_stream, host, port)
         ↓
5. 连接到达: 调用 handle_stream(reader, writer)
         ↓
6. 创建处理器: ProxyConnectionHandler(master, r, w, options, mode)
         ↓
7. 顶层 Layer: make_top_layer() → layers.modes.HttpProxy(context)
         ↓
8. 开始处理: handler.handle_client()
```

**关键代码：**

```python
# mode_servers.py:473-475
class RegularInstance(AsyncioServerInstance[mode_specs.RegularMode]):
    def make_top_layer(self, context: Context) -> Layer:
        return layers.modes.HttpProxy(context)
```

**HttpProxy 层实现：**

```python
# layers/modes.py:24-29
class HttpProxy(layer.Layer):
    @expect(events.Start)
    def _handle_event(self, event: events.Event) -> layer.CommandGenerator[None]:
        child_layer = layer.NextLayer(self.context)
        self._handle_event = child_layer.handle_event
        yield from child_layer.handle_event(event)
```

**特点：**
- 直接创建 `NextLayer`，由 `next_layer` addon 决定后续层
- 客户端发送 HTTP CONNECT 请求或绝对 URL 形式的请求

---

#### 3.2.2 SOCKS5 模式

**入口路径：**

```
1. 用户配置: --mode socks5 或 --mode socks5@1080
         ↓
2. 模式解析: ProxyMode.parse("socks5") → Socks5Mode
         ↓
3. 服务器创建: ServerInstance.make() → Socks5Instance
         ↓
4. 监听端口: asyncio.start_server(handle_stream, host, port)
         ↓
5. 连接到达: handle_stream() → ProxyConnectionHandler
         ↓
6. 顶层 Layer: make_top_layer() → layers.modes.Socks5Proxy(context)
         ↓
7. SOCKS5 握手: 处理版本协商、认证、CONNECT 请求
         ↓
8. 解析目标地址: 从 SOCKS5 协议中提取 host:port
         ↓
9. 创建 NextLayer: child_layer = layer.NextLayer(self.context)
         ↓
10. 进入统一处理流程
```

**关键代码：**

```python
# layers/modes.py:133-302
class Socks5Proxy(DestinationKnown):
    buf: bytes = b""
    
    # 状态机处理 SOCKS5 协议
    state: Callable[..., layer.CommandGenerator[None]] = state_greet
    
    def state_greet(self) -> layer.CommandGenerator[None]:
        # 处理版本协商: 0x05 (SOCKS5)
        if self.buf[0] != SOCKS5_VERSION:
            yield from self.socks_err("Invalid SOCKS version...")
            return
        
        # 选择认证方法
        if "proxyauth" in self.context.options:
            method = SOCKS5_METHOD_USER_PASSWORD_AUTHENTICATION
            self.state = self.state_auth
        else:
            method = SOCKS5_METHOD_NO_AUTHENTICATION_REQUIRED
            self.state = self.state_connect
        
        yield commands.SendData(self.context.client, bytes([SOCKS5_VERSION, method]))
    
    def state_connect(self) -> layer.CommandGenerator[None]:
        # 解析 CONNECT 请求中的目标地址
        # 支持 IPv4、IPv6、域名三种地址类型
        atyp = self.buf[3]
        if atyp == SOCKS5_ATYP_IPV4_ADDRESS:
            host = socket.inet_ntop(socket.AF_INET, msg[4:-2])
        elif atyp == SOCKS5_ATYP_DOMAINNAME:
            host_bytes = msg[5:-2]
            host = host_bytes.decode("ascii", "replace")
        
        (port,) = struct.unpack("!H", msg[-2:])
        
        # 设置目标地址，创建 NextLayer
        self.context.server.address = (host, port)
        self.child_layer = layer.NextLayer(self.context)
        
        # 发送成功响应
        yield commands.SendData(
            self.context.client, 
            b"\x05\x00\x00\x01\x00\x00\x00\x00\x00\x00"
        )
        
        # 进入统一处理流程
        err = yield from self.finish_start()
```

**SOCKS5 协议状态机：**

```
┌──────────┐    版本协商    ┌──────────┐
│  Start   │ ────────────→ │ state_   │
│          │               │  greet   │
└──────────┘               └────┬─────┘
                               │
              ┌────────────────┼────────────────┐
              ↓ 无认证          ↓ 用户名密码认证
         ┌──────────┐     ┌──────────┐
         │ state_   │     │ state_   │
         │ connect  │ ←── │   auth   │
         └────┬─────┘     └──────────┘
              │
              ↓ 解析目标地址
         ┌────────────────────────────────┐
         │ NextLayer → 统一核心处理流程    │
         └────────────────────────────────┘
```

---

#### 3.2.3 透明代理模式

**入口路径：**

```
1. 用户配置: --mode transparent
         ↓
2. 模式解析: ProxyMode.parse("transparent") → TransparentMode
         ↓
3. 服务器创建: ServerInstance.make() → TransparentInstance
         ↓
4. 监听端口: asyncio.start_server(handle_stream, host, port)
         ↓
5. 连接到达: handle_stream()
         ↓
6. 特殊处理: 获取原始目标地址 (SO_ORIGINAL_DST)
         ↓
7. 设置 context: 
     - handler.layer.context.client.sockname = original_dst
     - handler.layer.context.server.address = original_dst
         ↓
8. 顶层 Layer: make_top_layer() → layers.modes.TransparentProxy(context)
         ↓
9. 进入统一处理流程
```

**关键代码 - 获取原始目标地址：**

```python
# mode_servers.py:197-209
if isinstance(self.mode, mode_specs.TransparentMode):
    assert isinstance(writer, asyncio.StreamWriter)
    s = cast(socket.socket, writer.get_extra_info("socket"))
    try:
        assert platform.original_addr
        original_dst = platform.original_addr(s)
    except Exception as e:
        logger.error(f"Transparent mode failure: {e!r}")
        writer.close()
        return
    else:
        handler.layer.context.client.sockname = original_dst
        handler.layer.context.server.address = original_dst
```

**平台相关实现：**

```python
# platform/linux.py
def original_addr(sock: socket.socket) -> Address:
    # Linux: 使用 SO_ORIGINAL_DST 选项
    dst = sock.getsockopt(socket.SOL_IP, SO_ORIGINAL_DST, 16)
    port, raw_ip = struct.unpack_from("!2xH4s", dst)
    ip = socket.inet_ntop(socket.AF_INET, raw_ip)
    return (ip, port)

# platform/osx.py (pf 防火墙)
def original_addr(sock: socket.socket) -> Address:
    # macOS: 使用 SO_ORIGINAL_DST 或 ioctl
    ...
```

**TransparentProxy 层实现：**

```python
# layers/modes.py:89-96
class TransparentProxy(DestinationKnown):
    @expect(events.Start)
    def _handle_event(self, event: events.Event) -> layer.CommandGenerator[None]:
        assert self.context.server.address, "No server address set."
        self.child_layer = layer.NextLayer(self.context)
        err = yield from self.finish_start()
        if err:
            yield commands.CloseConnection(self.context.client)
```

**特点：**
- 依赖操作系统防火墙规则（iptables、pf 等）将流量重定向
- 通过 `SO_ORIGINAL_DST` 套接字选项获取原始目标地址
- 目标地址在连接建立时就已确定

---

#### 3.2.4 反向代理模式

**入口路径：**

```
1. 用户配置: --mode reverse:https://example.com
         ↓
2. 模式解析: ProxyMode.parse() → ReverseMode
   - scheme: https
   - address: ('example.com', 443)
         ↓
3. 服务器创建: ServerInstance.make() → ReverseInstance
         ↓
4. 监听端口: 根据 scheme 决定协议
   - http/https/tcp/tls/dns → TCP 或 Both
   - http3/quic/udp/dtls → UDP
         ↓
5. 连接到达: handle_stream() → ProxyConnectionHandler
         ↓
6. 顶层 Layer: make_top_layer() → layers.modes.ReverseProxy(context)
         ↓
7. 设置目标地址: self.context.server.address = spec.address
         ↓
8. 可选设置 SNI: self.context.server.sni = spec.address[0]
         ↓
9. 创建 NextLayer，进入统一处理流程
```

**关键代码：**

```python
# layers/modes.py:65-87
class ReverseProxy(DestinationKnown):
    @expect(events.Start)
    def _handle_event(self, event: events.Event) -> layer.CommandGenerator[None]:
        spec = self.context.client.proxy_mode
        assert isinstance(spec, ReverseMode)
        self.context.server.address = spec.address
        
        self.child_layer = layer.NextLayer(self.context)
        
        # 根据协议设置 SNI
        match spec.scheme:
            case "http3" | "quic" | "https" | "tls" | "dtls":
                if not self.context.options.keep_host_header:
                    self.context.server.sni = spec.address[0]
            case "tcp" | "http" | "udp" | "dns":
                pass
        
        err = yield from self.finish_start()
        if err:
            yield commands.CloseConnection(self.context.client)
```

**ReverseMode 支持的协议：**

| scheme | 传输协议 | 说明 |
|--------|---------|------|
| `http` | TCP | 普通 HTTP 反向代理 |
| `https` | Both | HTTPS 反向代理（默认） |
| `http3` | UDP | HTTP/3 (QUIC) 反向代理 |
| `tcp` | TCP | 纯 TCP 反向代理 |
| `tls` | TCP | TLS 终止反向代理 |
| `udp` | UDP | 纯 UDP 反向代理 |
| `dtls` | UDP | DTLS 反向代理 |
| `dns` | Both | DNS 反向代理 |
| `quic` | UDP | 原始 QUIC 反向代理 |

---

#### 3.2.5 Upstream 模式（链式代理）

**入口路径：**

```
1. 用户配置: --mode upstream:http://proxy.example.com:8080
         ↓
2. 模式解析: ProxyMode.parse() → UpstreamMode
   - scheme: http
   - address: ('proxy.example.com', 8080)
         ↓
3. 服务器创建: ServerInstance.make() → UpstreamInstance
         ↓
4. 监听端口: asyncio.start_server(handle_stream, host, port)
         ↓
5. 连接到达: handle_stream() → ProxyConnectionHandler
         ↓
6. 顶层 Layer: make_top_layer() → layers.modes.HttpUpstreamProxy(context)
         ↓
7. HttpUpstreamProxy 创建 NextLayer
         ↓
8. next_layer addon 识别为上游模式，创建 HttpLayer(HTTPMode.upstream)
         ↓
9. HttpLayer 在连接上游时发送 CONNECT 请求
```

**关键代码：**

```python
# mode_servers.py:478-480
class UpstreamInstance(AsyncioServerInstance[mode_specs.UpstreamMode]):
    def make_top_layer(self, context: Context) -> Layer:
        return layers.modes.HttpUpstreamProxy(context)

# layers/modes.py:32-37
class HttpUpstreamProxy(layer.Layer):
    @expect(events.Start)
    def _handle_event(self, event: events.Event) -> layer.CommandGenerator[None]:
        child_layer = layer.NextLayer(self.context)
        self._handle_event = child_layer.handle_event
        yield from child_layer.handle_event(event)
```

**上游代理连接逻辑：**

```python
# layers/http/__init__.py:964-967
if self.mode is HTTPMode.upstream:
    proxy_mode = self.context.client.proxy_mode
    assert isinstance(proxy_mode, UpstreamMode)
    self.context.server.via = (proxy_mode.scheme, proxy_mode.address)
```

---

#### 3.2.6 WireGuard 模式

**入口路径：**

```
1. 用户配置: --mode wireguard 或 --mode wireguard@51820
         ↓
2. 模式解析: ProxyMode.parse("wireguard") → WireGuardMode
         ↓
3. 服务器创建: ServerInstance.make() → WireGuardServerInstance
         ↓
4. 密钥管理: 加载或生成密钥对 (server_key, client_key)
         ↓
5. 启动服务器: mitmproxy_rs.wireguard.start_wireguard_server()
   - Rust 实现的 WireGuard 协议栈
   - 处理 UDP 数据包
         ↓
6. 虚拟连接到达: handle_stream()
   - WireGuard 解密后的数据流
         ↓
7. 特殊处理: 设置目标地址
   handler.layer.context.server.address = writer.get_extra_info(
       "remote_endpoint", handler.layer.context.client.sockname
   )
         ↓
8. 顶层 Layer: make_top_layer() → layers.modes.TransparentProxy(context)
         ↓
9. 进入透明代理统一处理流程
```

**关键代码：**

```python
# mode_servers.py:336-417
class WireGuardServerInstance(AsyncioServerInstance[mode_specs.WireGuardMode]):
    server_key: str
    client_key: str
    pubkey: str
    
    def make_top_layer(self, context: Context) -> Layer:
        # WireGuard 模式使用透明代理层
        return layers.modes.TransparentProxy(context)
    
    async def _start(self) -> None:
        # 加载或生成密钥配置
        conf_path = Path(self.mode.data) if self.mode.data else \
                    Path(ctx.options.confdir) / "wireguard.conf"
        
        if not conf_path.exists():
            conf_path.write_text(json.dumps({
                "server_key": mitmproxy_rs.wireguard.genkey(),
                "client_key": mitmproxy_rs.wireguard.genkey(),
            }, indent=4))
        
        # 启动 WireGuard 服务器 (Rust 实现)
        await super()._start()
    
    async def start_udp_based_server(self, host, port):
        return await mitmproxy_rs.wireguard.start_wireguard_server(
            host,
            port,
            self.server_key,
            [self.pubkey],
            self.handle_stream,  # TCP 流处理
            self.handle_stream,  # UDP 数据报处理
        )
```

**WireGuard 客户端配置示例：**

```ini
[Interface]
PrivateKey = <client_key>
Address = 10.0.0.1/32
DNS = 10.0.0.53

[Peer]
PublicKey = <server_pubkey>
AllowedIPs = 0.0.0.0/0
Endpoint = <server_host>:51820
```

---

#### 3.2.7 Local 模式（本地重定向）

**入口路径：**

```
1. 用户配置: --mode local 或 --mode local:chrome.exe
         ↓
2. 模式解析: ProxyMode.parse("local") → LocalMode
         ↓
3. 服务器创建: ServerInstance.make() → LocalRedirectorInstance
         ↓
4. 启动重定向器: mitmproxy_rs.local.start_local_redirector()
   - Windows: 可能使用 WFP (Windows Filtering Platform)
   - macOS: 使用 pf 防火墙
   - Linux: 使用 iptables/nftables
         ↓
5. 设置拦截规则: cls._server.set_intercept(spec)
   spec = f"{self.mode.data},!{os.getpid()}"  # 排除自身进程
         ↓
6. 被拦截的连接到达: redirector_handle_stream()
         ↓
7. 转发到 handle_stream() → ProxyConnectionHandler
         ↓
8. 顶层 Layer: make_top_layer() → layers.modes.TransparentProxy(context)
         ↓
9. 进入透明代理统一处理流程
```

**关键代码：**

```python
# mode_servers.py:420-470
class LocalRedirectorInstance(ServerInstance[mode_specs.LocalMode]):
    _server: ClassVar[mitmproxy_rs.local.LocalRedirector | None] = None
    _instance: ClassVar[LocalRedirectorInstance | None] = None
    
    def make_top_layer(self, context: Context) -> Layer:
        return layers.modes.TransparentProxy(context)
    
    @classmethod
    async def redirector_handle_stream(cls, stream: mitmproxy_rs.Stream):
        if cls._instance is not None:
            await cls._instance.handle_stream(stream)
    
    async def _start(self) -> None:
        # 构建拦截规则
        if self.mode.data:
            spec = f"{self.mode.data},!{os.getpid()}"
        else:
            spec = f"!{os.getpid()}"  # 默认拦截所有进程，排除自身
        
        # 启动重定向器守护进程（单例）
        if cls._server is None:
            cls._server = await mitmproxy_rs.local.start_local_redirector(
                cls.redirector_handle_stream,
                cls.redirector_handle_stream,
            )
        
        # 设置拦截规则
        cls._server.set_intercept(spec)
```

**特点：**
- 操作系统级别的流量拦截
- 无需修改客户端配置
- 可按进程名过滤（如 `--mode local:chrome.exe`）
- 自动排除 mitmproxy 自身进程防止死循环

---

#### 3.2.8 TUN 模式

**入口路径：**

```
1. 用户配置: --mode tun 或 --mode tun:utun3
         ↓
2. 模式解析: ProxyMode.parse("tun") → TunMode
         ↓
3. 服务器创建: ServerInstance.make() → TunInstance
         ↓
4. 创建 TUN 接口: mitmproxy_rs.tun.create_tun_interface()
         ↓
5. 操作系统路由配置: 用户需配置路由表指向 TUN 接口
         ↓
6. IP 数据包到达: TUN 接口读取 IP 包
         ↓
7. 协议解析: Rust 层解析 TCP/UDP，提取流
         ↓
8. 虚拟连接到达: handle_stream()
         ↓
9. 特殊处理: 设置目标地址
   handler.layer.context.server.address = writer.get_extra_info(
       "remote_endpoint", handler.layer.context.client.sockname
   )
         ↓
10. 顶层 Layer: make_top_layer() → layers.modes.TransparentProxy(context)
         ↓
11. 进入透明代理统一处理流程
```

**关键代码：**

```python
# mode_servers.py:503-541
class TunInstance(ServerInstance[mode_specs.TunMode]):
    _server: mitmproxy_rs.tun.TunInterface | None = None
    listen_addrs = ()
    
    def make_top_layer(self, context: Context) -> Layer:
        return layers.modes.TransparentProxy(context)
    
    async def _start(self) -> None:
        assert self._server is None
        self._server = await mitmproxy_rs.tun.create_tun_interface(
            self.handle_stream,    # TCP 流处理
            self.handle_stream,    # UDP 数据报处理
            tun_name=self.mode.data or None,  # 可选指定接口名
        )
        logger.info(f"TUN interface created: {self._server.tun_name()}")
```

**TUN 模式网络配置示例：**

```bash
# 创建 TUN 接口后，需要配置 IP 和路由
sudo ip addr add 10.0.0.1/24 dev tun0
sudo ip link set tun0 up

# 路由特定流量到 TUN 接口
sudo ip route add 192.168.1.0/24 dev tun0

# 或者路由所有流量（需要配置 NAT）
sudo ip route add default via 10.0.0.1 dev tun0
```

---

#### 3.2.9 DNS 模式

**入口路径：**

```
1. 用户配置: --mode dns 或 --mode dns@5353
         ↓
2. 模式解析: ProxyMode.parse("dns") → DnsMode
         ↓
3. 服务器创建: ServerInstance.make() → DnsInstance
         ↓
4. 监听端口: TCP + UDP (transport_protocol = BOTH)
   - asyncio.start_server() for TCP
   - mitmproxy_rs.udp.start_udp_server() for UDP
         ↓
5. DNS 查询到达:
   - UDP: 53 端口 UDP 数据报
   - TCP: 53 端口 TCP 流（DNS over TCP）
         ↓
6. 顶层 Layer: make_top_layer() → layers.DNSLayer(context)
         ↓
7. DNSLayer 处理 DNS 协议
         ↓
8. 触发 dns_request、dns_response hooks
```

**关键代码：**

```python
# mode_servers.py:498-500
class DnsInstance(AsyncioServerInstance[mode_specs.DnsMode]):
    def make_top_layer(self, context: Context) -> Layer:
        return layers.DNSLayer(context)
```

---

## 四、统一核心处理流程

### 4.1 流程总览

无论哪种代理模式，最终都汇入以下统一的核心处理流程：

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        连接建立阶段                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. ProxyConnectionHandler 初始化                                             │
│     ├── 创建 Client 连接对象                                                  │
│     ├── 创建 Context (包含 client, server, options, layers)                  │
│     └── 创建 NextLayer 作为初始层                                             │
│                                                                              │
│  2. handle_client() 开始执行                                                  │
│     ├── 启动超时看门狗 (TimeoutWatchdog)                                      │
│     ├── 触发 ClientConnectedHook                                              │
│     └── 发送 Start 事件到层栈                                                 │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        层决策阶段 (NextLayer)                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 顶层模式层发送 Start 事件                                                 │
│     ├── HttpProxy / Socks5Proxy / TransparentProxy / ...                   │
│     └── 创建 NextLayer，触发 NextLayerHook                                   │
│                                                                              │
│  2. next_layer addon 决策                                                    │
│     ├── 检查代理模式                                                          │
│     ├── 检查数据特征 (TLS ClientHello, HTTP 报文等)                          │
│     ├── 检查配置 (--tcp-hosts, --udp-hosts, --ignore-hosts)                │
│     └── 创建相应的下一层                                                      │
│                                                                              │
│  3. 典型决策路径：                                                            │
│     ├── 反向代理 → 根据 scheme 构建层栈                                       │
│     ├── 显式 HTTP 代理 → 检查是否 TLS → HttpLayer                            │
│     ├── 透明模式 → 检查数据 → TLSLayer / HttpLayer / TCPLayer               │
│     └── 特殊配置 → TCPLayer / UDPLayer (忽略或直通)                          │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        事件/命令循环阶段                                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ConnectionHandler.server_event() 是核心调度函数：                           │
│                                                                              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  def server_event(self, event: events.Event):                        │   │
│  │      # 1. 注册活动，重置超时                                          │   │
│  │      self.timeout_watchdog.register_activity()                        │   │
│  │                                                                       │   │
│  │      # 2. 调用当前层的 handle_event，获取命令列表                      │   │
│  │      layer_commands = self.layer.handle_event(event)                 │   │
│  │                                                                       │   │
│  │      # 3. 处理每个命令                                                │   │
│  │      for command in layer_commands:                                  │   │
│  │          if isinstance(command, commands.OpenConnection):            │   │
│  │              # 打开上游连接                                           │   │
│  │          elif isinstance(command, commands.SendData):                │   │
│  │              # 发送数据到连接                                         │   │
│  │          elif isinstance(command, commands.StartHook):               │   │
│  │              # 触发 addon hook                                        │   │
│  │          elif isinstance(command, commands.CloseConnection):         │   │
│  │              # 关闭连接                                                │   │
│  │          # ... 更多命令类型                                            │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        数据处理阶段                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 连接 IO 处理                                                             │
│     ├── handle_connection() 循环读取数据                                     │
│     ├── 收到数据 → DataReceived 事件 → 层栈处理                              │
│     └── 连接关闭 → ConnectionClosed 事件                                     │
│                                                                              │
│  2. 上游连接管理                                                             │
│     ├── OpenConnection 命令 → asyncio.open_connection()                    │
│     ├── ServerConnectedHook / ServerConnectErrorHook                        │
│     └── 连接池和复用 (HTTP/2, HTTP/3)                                       │
│                                                                              │
│  3. 应用层处理                                                               │
│     ├── HttpLayer → HttpStream → request/response hooks                     │
│     ├── TLSLayer → 证书伪造、TLS 终止/转发                                   │
│     ├── TCPLayer → 原始 TCP 数据转发                                         │
│     ├── WebsocketLayer → WebSocket 消息处理                                 │
│     └── DNSLayer → DNS 查询/响应处理                                         │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      ↓
┌─────────────────────────────────────────────────────────────────────────────┐
│                        连接关闭阶段                                             │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  1. 超时或主动关闭                                                           │
│     ├── TimeoutWatchdog 超时 → handler.cancel()                             │
│     └── 命令 CloseConnection → writer.close()                                │
│                                                                              │
│  2. 清理资源                                                                 │
│     ├── 取消所有活跃任务                                                      │
│     ├── 关闭所有传输连接                                                      │
│     └── 从 connections 字典移除                                              │
│                                                                              │
│  3. 触发钩子                                                                 │
│     └── ClientDisconnectedHook                                               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 4.2 核心类详解

#### 4.2.1 ProxyConnectionHandler

```python
# mode_servers.py:61-76
class ProxyConnectionHandler(server.LiveConnectionHandler):
    master: Master
    
    def __init__(self, master, r, w, options, mode):
        self.master = master
        super().__init__(r, w, options, mode)
        self.log_prefix = f"{human.format_address(self.client.peername)}: "
    
    async def handle_hook(self, hook: commands.StartHook) -> None:
        with self.timeout_watchdog.disarm():
            (data,) = hook.args()
            await self.master.addons.handle_lifecycle(hook)
            if isinstance(data, flow.Flow):
                await data.wait_for_resume()
```

**职责：**
- 桥接底层连接和 addon 系统
- 处理生命周期钩子，支持 flow 暂停/恢复
- 继承自 `LiveConnectionHandler`，提供完整的连接处理能力

#### 4.2.2 LiveConnectionHandler

```python
# server.py:465-485
class LiveConnectionHandler(ConnectionHandler, metaclass=abc.ABCMeta):
    def __init__(
        self,
        reader: asyncio.StreamReader | mitmproxy_rs.Stream,
        writer: asyncio.StreamWriter | mitmproxy_rs.Stream,
        options: moptions.Options,
        mode: mode_specs.ProxyMode,
    ) -> None:
        # 创建 Client 连接对象
        client = Client(
            transport_protocol=writer.get_extra_info("transport_protocol", "tcp"),
            peername=writer.get_extra_info("peername"),
            sockname=writer.get_extra_info("sockname"),
            timestamp_start=time.time(),
            proxy_mode=mode,
            state=ConnectionState.OPEN,
        )
        # 创建 Context
        context = Context(client, options)
        super().__init__(context)
        # 注册传输 IO
        self.transports[client] = ConnectionIO(
            handler=None, reader=reader, writer=writer
        )
```

#### 4.2.3 ConnectionHandler (核心调度器)

```python
# server.py:98-463
class ConnectionHandler(metaclass=abc.ABCMeta):
    transports: MutableMapping[Connection, ConnectionIO]
    timeout_watchdog: TimeoutWatchdog
    client: Client
    layer: "layer.Layer"
    
    async def handle_client(self) -> None:
        # 1. 设置任务调试信息
        asyncio_utils.set_current_task_debug_info(...)
        
        # 2. 启动超时看门狗
        watch = asyncio_utils.create_task(self.timeout_watchdog.watch(), ...)
        
        # 3. 客户端连接钩子
        self.log("client connect")
        await self.handle_hook(server_hooks.ClientConnectedHook(self.client))
        
        # 4. 发送 Start 事件启动层栈
        await self.server_event(events.Start())
        
        # 5. 启动客户端连接处理器
        handler = asyncio_utils.create_task(
            self.handle_connection(self.client), ...
        )
        self.transports[self.client].handler = handler
        
        # 6. 等待连接处理完成
        await asyncio.wait([handler])
        
        # 7. 清理
        watch.cancel()
        # ... 取消定时器等
        
        # 8. 客户端断开钩子
        self.log("client disconnect")
        self.client.timestamp_end = time.time()
        await self.handle_hook(server_hooks.ClientDisconnectedHook(self.client))
```

**核心调度循环 - server_event：**

```python
# server.py:377-437
async def server_event(self, event: events.Event) -> None:
    async with self._server_event_lock:
        self.timeout_watchdog.register_activity()
        
        # 调用层处理事件，获取命令生成器
        layer_commands = self.layer.handle_event(event)
        
        for command in layer_commands:
            # 根据命令类型分发处理
            
            if isinstance(command, commands.OpenConnection):
                # 打开上游连接
                handler = asyncio_utils.create_task(
                    self.open_connection(command), ...
                )
                self.transports[command.connection] = ConnectionIO(handler=handler)
            
            elif isinstance(command, commands.SendData):
                # 发送数据
                writer = self.transports[command.connection].writer
                if not writer.is_closing():
                    writer.write(command.data)
            
            elif isinstance(command, commands.StartHook):
                # 触发钩子
                asyncio_utils.create_task(
                    self.hook_task(command), ...
                )
            
            elif isinstance(command, commands.Log):
                # 日志记录
                self.log(command.message, command.level)
            
            elif isinstance(command, commands.RequestWakeup):
                # 定时唤醒
                task = asyncio_utils.create_task(self.wakeup(command), ...)
                self.wakeup_timer.add(task)
            
            elif isinstance(command, commands.CloseConnection):
                # 关闭连接
                self.close_connection(command.connection, False)
            
            # ... 更多命令类型
```

### 4.3 层 (Layer) 机制

#### 4.3.1 Layer 基类

```python
# layer.py:44-241
class Layer:
    context: Context
    _paused: Paused | None  # 用于阻塞命令的暂停状态
    _paused_event_queue: collections.deque[events.Event]
    
    def __init__(self, context: Context) -> None:
        self.context = context
        self.context.layers.append(self)  # 注册到层栈
        self._paused = None
        self._paused_event_queue = collections.deque()
    
    def handle_event(self, event: events.Event) -> CommandGenerator[None]:
        if self._paused:
            # 处理暂停状态
            pause_finished = (
                isinstance(event, events.CommandCompleted)
                and event.command is self._paused.command
            )
            if pause_finished:
                # 恢复执行
                yield from self.__continue(event)
            else:
                # 缓冲事件，稍后重放
                self._paused_event_queue.append(event)
        else:
            # 正常处理
            command_generator = self._handle_event(event)
            # 处理命令生成器，支持阻塞命令
            # ...
    
    @abstractmethod
    def _handle_event(self, event: events.Event) -> CommandGenerator[None]:
        """子类实现具体的事件处理逻辑"""
        yield from ()
```

#### 4.3.2 阻塞命令机制

Layer 支持类似阻塞代码的写法，但实际是非阻塞的：

```python
# 示例：在层中"阻塞"等待连接建立
def _handle_event(self, event):
    # 这行代码看起来是阻塞的
    # 但实际上通过 yield 实现异步等待
    err = yield commands.OpenConnection(self.context.server)
    
    if err:
        yield commands.Log(f"Connection failed: {err}")
        return
    
    # 连接建立后继续执行
    yield commands.SendData(self.context.server, b"GET / HTTP/1.1\r\n...")
```

**实现原理：**
1. `yield commands.OpenConnection(...)` 暂停生成器
2. `ConnectionHandler.server_event()` 收到命令，开始异步建立连接
3. 连接建立完成后，发送 `CommandCompleted` 事件
4. `Layer.handle_event()` 检测到对应的完成事件，恢复生成器执行
5. 通过 `generator.send(reply)` 将结果返回给层

### 4.4 NextLayer 决策机制

#### 4.4.1 NextLayer 类

```python
# layer.py:248-341
class NextLayer(Layer):
    layer: Layer | None  # 由 addon 设置
    events: list[events.Event]  # 决策前缓冲的事件
    
    def _handle_event(self, event: events.Event):
        self.events.append(event)
        
        # 收到 Start 事件或数据时触发决策
        if self._ask_on_start and isinstance(event, events.Start):
            yield from self._ask()
        elif isinstance(event, events.DataReceived):
            yield from self._ask()
    
    def _ask(self):
        # 触发 NextLayerHook
        yield NextLayerHook(self)
        
        # 如果 addon 已设置下一层
        if self.layer:
            # 将缓冲的事件转发给新层
            for e in self.events:
                yield from self.layer.handle_event(e)
            self.events.clear()
            
            # 替换自身为新层
            self.handle_event = self.layer.handle_event
            self._handle_event = self.layer.handle_event
            self._handle = self.layer.handle_event
```

#### 4.4.2 next_layer addon 决策逻辑

```python
# addons/next_layer.py:115-196
class NextLayer:
    def _next_layer(self, context: Context, data_client: bytes, data_server: bytes):
        # 决策顺序：
        
        # 1. 检查忽略/允许列表
        if self._ignore_connection(context, data_client, data_server):
            return TCPLayer(...)  # 或 UDPLayer
        
        # 2. 处理有明确定义的代理模式
        # 2a) 反向代理：根据 scheme 构建层栈
        if stack_match(context, [modes.ReverseProxy]):
            return self._setup_reverse_proxy(context, data_client)
        
        # 2b) 显式 HTTP 代理
        if stack_match(context, [modes.HttpProxy, modes.HttpUpstreamProxy]):
            return self._setup_explicit_http_proxy(context, data_client)
        
        # 3. 检查安全协议
        # 3a) TLS/DTLS
        is_tls_or_dtls = (
            tcp_based and starts_like_tls_record(data_client)
            or udp_based and starts_like_dtls_record(data_client)
        )
        if is_tls_or_dtls:
            # ServerTLSLayer 包装 ClientTLSLayer
            server_tls = ServerTLSLayer(context)
            server_tls.child_layer = ClientTLSLayer(context)
            return server_tls
        
        # 3b) QUIC
        if udp_based and _starts_like_quic(data_client, ...):
            server_quic = ServerQuicLayer(context)
            server_quic.child_layer = ClientQuicLayer(context)
            return server_quic
        
        # 4. 检查 --tcp-hosts / --udp-hosts 配置
        if tcp_based and self._is_destination_in_hosts(context, self.tcp_hosts):
            return layers.TCPLayer(context)
        if udp_based and self._is_destination_in_hosts(context, self.udp_hosts):
            return layers.UDPLayer(context)
        
        # 5. 检查应用层协议
        # 5a) 已知 ALPN
        if context.client.alpn:
            if context.client.alpn in HTTP_ALPNS:
                return layers.HttpLayer(context, HTTPMode.transparent)
        
        # 5b) DNS 端口
        if context.server.address and context.server.address[1] in (53, 5353):
            return layers.DNSLayer(context)
        
        # 5c) UDP 默认回退
        if udp_based:
            return layers.UDPLayer(context)
        
        # 5d) 检查是否为 HTTP
        probably_no_http = (
            len(data_client) < 3
            or b" " not in data_client
            or not data_client[:3].isalpha()
            or data_client.startswith(b"SSH")
        )
        if ctx.options.rawtcp and probably_no_http:
            return layers.TCPLayer(context)
        
        # 5e) 默认假设为 HTTP
        return layers.HttpLayer(context, HTTPMode.transparent)
```

---

## 五、各模式汇入核心流程的统一节点

### 5.1 汇入点汇总

| 代理模式 | 顶层 Layer 类型 | 汇入统一流程的方式 | 关键设置 |
|---------|----------------|-------------------|---------|
| **regular** | `HttpProxy` | 直接创建 `NextLayer` | 无特殊设置 |
| **socks5** | `Socks5Proxy` | 握手后解析目标地址，创建 `NextLayer` | `context.server.address` |
| **transparent** | `TransparentProxy` | 连接建立时获取原始目标，创建 `NextLayer` | `context.server.address`, `client.sockname` |
| **reverse** | `ReverseProxy` | 从模式配置获取目标，创建 `NextLayer` | `context.server.address`, 可选 `server.sni` |
| **upstream** | `HttpUpstreamProxy` | 直接创建 `NextLayer`，HttpLayer 处理上游 | `context.server.via` |
| **wireguard** | `TransparentProxy` | 从虚拟网络获取目标，复用透明代理流程 | `context.server.address` (从 extra_info 获取) |
| **local** | `TransparentProxy` | 操作系统重定向，复用透明代理流程 | `context.server.address` |
| **tun** | `TransparentProxy` | TUN 接口解析目标，复用透明代理流程 | `context.server.address` |
| **dns** | `DNSLayer` | 直接进入 DNS 协议处理 | 无 NextLayer，直接处理 |

### 5.2 关键统一节点

#### 节点 1: ServerInstance.handle_stream()

所有模式的连接都经过这个统一的入口处理：

```python
# mode_servers.py:185-219
async def handle_stream(self, reader, writer=None):
    if writer is None:
        writer = reader
    
    # 1. 创建统一的连接处理器
    handler = ProxyConnectionHandler(
        ctx.master, reader, writer, ctx.options, self.mode
    )
    
    # 2. 各模式特殊处理（设置目标地址等）
    if isinstance(self.mode, mode_specs.TransparentMode):
        # 获取原始目标地址
        original_dst = platform.original_addr(socket)
        handler.layer.context.client.sockname = original_dst
        handler.layer.context.server.address = original_dst
    
    elif isinstance(self.mode, (mode_specs.WireGuardMode, ...)):
        # 从虚拟网络获取目标
        handler.layer.context.server.address = writer.get_extra_info(
            "remote_endpoint", handler.layer.context.client.sockname
        )
    
    # 3. 创建顶层 Layer（各模式不同）
    handler.layer = self.make_top_layer(handler.layer.context)
    
    # 4. 注册连接并进入统一处理流程
    with self.manager.register_connection(handler.layer.context.client.id, handler):
        await handler.handle_client()  # 统一入口！
```

#### 节点 2: ConnectionHandler.handle_client()

这是所有连接的统一处理入口：

```
所有模式
    ↓
handle_client()
    ├── TimeoutWatchdog (统一超时管理)
    ├── ClientConnectedHook (统一生命周期钩子)
    ├── server_event(events.Start()) (统一事件驱动)
    │       ↓
    │   Layer.handle_event() (统一层栈处理)
    │       ↓
    │   NextLayerHook (统一决策点)
    │       ↓
    │   next_layer addon (统一决策逻辑)
    └── handle_connection() (统一 IO 循环)
```

#### 节点 3: NextLayerHook

这是模式相关处理和模式无关处理的分界点：

```
┌─────────────────────────────────────────────────────────────────┐
│                    模式相关处理（各模式不同）                      │
├─────────────────────────────────────────────────────────────────┤
│  HttpProxy          Socks5Proxy        TransparentProxy         │
│       │                  │                     │                 │
│       └──────────────────┼─────────────────────┘                 │
│                          ↓                                         │
│                    NextLayer 实例                                  │
│                          │                                         │
│                    NextLayerHook (触发)                            │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                    模式无关处理（统一核心流程）                     │
├─────────────────────────────────────────────────────────────────┤
│  next_layer addon._next_layer()                                  │
│       │                                                          │
│       ├── 检查 ignore/allow 列表                                 │
│       ├── 检查数据特征 (TLS/HTTP/QUIC)                           │
│       ├── 检查配置 (tcp-hosts 等)                                │
│       └── 创建 HttpLayer / TLSLayer / TCPLayer 等               │
│                                                                  │
│  之后的所有处理都是模式无关的：                                    │
│  - 事件/命令循环                                                  │
│  - 钩子触发                                                        │
│  - 连接管理                                                        │
│  - 流量处理                                                        │
└─────────────────────────────────────────────────────────────────┘
```

### 5.3 透明代理类模式的统一处理

WireGuard、Local、TUN 这三种模式都复用了透明代理的处理流程：

```
┌─────────────────────────────────────────────────────────────────┐
│                     虚拟网络层                                    │
├─────────────────────────────────────────────────────────────────┤
│  WireGuard        Local Redirector        TUN Interface         │
│  (UDP 隧道)        (操作系统拦截)           (虚拟网卡)            │
│       │                  │                     │                 │
│       └──────────────────┼─────────────────────┘                 │
│                          ↓                                         │
│              虚拟连接到达 handle_stream()                          │
│                          │                                         │
│              writer.get_extra_info("remote_endpoint")            │
│                          │                                         │
│              context.server.address = remote_endpoint             │
└─────────────────────────────────────────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────────────┐
│                   复用透明代理层                                   │
├─────────────────────────────────────────────────────────────────┤
│  make_top_layer() → TransparentProxy(context)                   │
│                          │                                         │
│  TransparentProxy._handle_event():                               │
│      assert context.server.address  # 已设置                      │
│      child_layer = NextLayer(context)                            │
│      yield from finish_start()                                   │
│                          │                                         │
│              进入统一的 NextLayer 决策流程                         │
└─────────────────────────────────────────────────────────────────┘
```

---

## 六、关键数据流示例

### 6.1 Regular 模式 HTTPS 请求完整流程

```
客户端: CONNECT example.com:443 HTTP/1.1
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 连接建立与模式层处理                                       │
├─────────────────────────────────────────────────────────────────┤
│  1. RegularInstance.listen() → 接受连接                           │
│  2. handle_stream() → ProxyConnectionHandler                     │
│  3. make_top_layer() → HttpProxy(context)                        │
│  4. handle_client() → server_event(Start())                      │
│  5. HttpProxy._handle_event(Start):                              │
│       child_layer = NextLayer(context)                           │
│       yield NextLayerHook(self)                                  │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: NextLayer 决策                                            │
├─────────────────────────────────────────────────────────────────┤
│  next_layer._next_layer():                                       │
│     - 栈匹配: [HttpProxy]                                         │
│     - 数据检查: starts_like_tls_record? 否 (是 HTTP CONNECT)    │
│     - 决策: _setup_explicit_http_proxy()                         │
│         → HttpLayer(context, HTTPMode.regular)                   │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3: HttpLayer 处理 CONNECT 请求                               │
├─────────────────────────────────────────────────────────────────┤
│  1. Http1Server 解析: "CONNECT example.com:443 HTTP/1.1"        │
│  2. ReceiveHttp(RequestHeaders) → HttpStream                     │
│  3. HttpStream.state_wait_for_request_headers():                 │
│       - flow.request.method == "CONNECT"                         │
│       - yield from handle_connect()                               │
│  4. handle_connect_regular():                                     │
│       - context.server.address = ("example.com", 443)           │
│       - child_layer = NextLayer(context)                          │
│       - yield NextLayerHook                                       │
│       - 发送 "HTTP/1.1 200 Connection established\r\n\r\n"       │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 4: TLS 握手决策                                              │
├─────────────────────────────────────────────────────────────────┤
│  客户端发送 TLS ClientHello                                       │
│       ↓
│  DataReceived(client, client_hello_bytes)                        │
│       ↓
│  NextLayer._next_layer():                                         │
│     - 数据检查: starts_like_tls_record(data_client) → True       │
│     - 决策:                                                        │
│         server_tls = ServerTLSLayer(context)                     │
│         server_tls.child_layer = ClientTLSLayer(context)         │
│         return server_tls                                          │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 5: TLS 层处理                                                │
├─────────────────────────────────────────────────────────────────┤
│  ClientTLSLayer:                                                  │
│     - 解析 ClientHello → context.client.sni = "example.com"     │
│     - yield TlsClienthelloHook(data)                              │
│     - yield from start_tls()                                      │
│     - TlsStartClientHook → tlsconfig addon 伪造证书              │
│     - TLS 握手完成                                                 │
│     - 提取明文: "GET / HTTP/1.1\r\n..."                          │
│     - child_layer = NextLayer(context)                            │
│     - yield NextLayerHook                                          │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 6: 最终 HTTP 处理                                            │
├─────────────────────────────────────────────────────────────────┤
│  NextLayer._next_layer():                                         │
│     - context.client.alpn = b"http/1.1"                          │
│     - 决策: HttpLayer(context, HTTPMode.transparent)             │
│                                                                  │
│  HttpLayer → HttpStream:                                          │
│     - 解析 "GET / HTTP/1.1\r\n..."                                │
│     - yield HttpRequestHeadersHook(flow)                          │
│     - (可选) addon 修改 request                                    │
│     - 建立上游连接: OpenConnection(server)                        │
│     - 发送请求到上游: "GET / HTTP/1.1\r\n..."                    │
│     - 接收响应: "HTTP/1.1 200 OK\r\n..."                         │
│     - yield HttpResponseHook(flow)                                │
│     - (可选) addon 修改 response                                   │
│     - 发送响应到客户端                                             │
└─────────────────────────────────────────────────────────────────┘
         ↓
客户端: 收到响应数据
```

### 6.2 SOCKS5 模式 HTTP 请求流程

```
客户端: SOCKS5 握手 → CONNECT example.com:80 → HTTP 请求
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: SOCKS5 协议处理                                           │
├─────────────────────────────────────────────────────────────────┤
│  Socks5Proxy 状态机:                                              │
│                                                                  │
│  1. state_greet:                                                 │
│     - 收到: 0x05 0x01 0x00 (版本5，1种方法，无认证)             │
│     - 发送: 0x05 0x00 (版本5，选择无认证)                        │
│     - 切换到 state_connect                                        │
│                                                                  │
│  2. state_connect:                                               │
│     - 收到: 0x05 0x01 0x00 0x03 0x0b example.com 0x00 0x50   │
│             (VER=5, CMD=CONNECT, RSV=0, ATYP=域名,              │
│              LEN=11, "example.com", PORT=80)                    │
│     - 解析: host="example.com", port=80                          │
│     - 设置: context.server.address = ("example.com", 80)        │
│     - 创建: child_layer = NextLayer(context)                     │
│     - 发送: 0x05 0x00 0x00 0x01 0x00 0x00 0x00 0x00 0x00 0x00│
│             (成功响应)                                             │
│     - yield from finish_start()                                   │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2-6: 与 Regular 模式相同的统一处理流程                        │
├─────────────────────────────────────────────────────────────────┤
│  NextLayerHook → HttpLayer → HttpRequestHook → 上游连接 → ...   │
└─────────────────────────────────────────────────────────────────┘
```

### 6.3 反向代理模式 HTTPS 请求流程

```
客户端: https://localhost:8443/  (mitmproxy 反向代理到 example.com)
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 1: 反向代理初始化                                            │
├─────────────────────────────────────────────────────────────────┤
│  1. 用户配置: --mode reverse:https://example.com                │
│  2. ReverseMode: scheme=https, address=('example.com', 443)    │
│  3. ReverseInstance 监听 8443 端口                               │
│  4. handle_stream() → ProxyConnectionHandler                     │
│  5. make_top_layer() → ReverseProxy(context)                     │
│  6. handle_client() → server_event(Start())                      │
│                                                                  │
│  ReverseProxy._handle_event(Start):                              │
│     - context.server.address = ('example.com', 443)             │
│     - context.server.sni = 'example.com'                         │
│     - child_layer = NextLayer(context)                           │
│     - yield from finish_start()                                  │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 2: next_layer 决策 (反向代理专用路径)                         │
├─────────────────────────────────────────────────────────────────┤
│  next_layer._next_layer():                                       │
│     - 栈匹配: stack_match([modes.ReverseProxy]) → True          │
│     - 调用: _setup_reverse_proxy(context, data_client)           │
│                                                                  │
│  _setup_reverse_proxy(scheme='https'):                          │
│     - transport_protocol = 'tcp' (默认)                          │
│     - 创建层栈:                                                   │
│         stack /= ServerTLSLayer(context)    # 与客户端 TLS      │
│         stack /= ClientTLSLayer(context)    # 与服务端 TLS      │
│         stack /= HttpLayer(context, HTTPMode.transparent)        │
│     - return stack[0]  # ServerTLSLayer                          │
└─────────────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────────────┐
│ Step 3-6: 与其他模式相同的统一处理流程                              │
├─────────────────────────────────────────────────────────────────┤
│  ServerTLSLayer (伪造证书) → ClientTLSLayer (验证上游证书)       │
│       ↓                                                           │
│  HttpLayer → HttpStream → request/response hooks                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 七、架构设计亮点

### 7.1 关注点分离

| 关注点 | 负责模块 | 说明 |
|-------|---------|------|
| **连接监听** | `ServerInstance` 子类 | 不同模式的网络监听方式不同 |
| **协议握手** | 顶层模式层 (`HttpProxy`, `Socks5Proxy` 等) | 处理 SOCKS5、HTTP CONNECT 等模式特有协议 |
| **目标解析** | 模式层 + 平台相关代码 | 透明代理通过 `SO_ORIGINAL_DST`，SOCKS5 通过协议解析 |
| **层决策** | `next_layer` addon | 统一的协议识别和层选择逻辑 |
| **协议处理** | `HttpLayer`, `TLSLayer` 等 | 与模式无关的核心协议实现 |
| **事件调度** | `ConnectionHandler` | 统一的事件/命令循环 |
| **钩子扩展** | Addon 系统 | 统一的生命周期钩子 |

### 7.2 可扩展性

1. **新增代理模式**：
   - 继承 `ProxyMode` 定义新模式规格
   - 继承 `ServerInstance` 实现服务器监听
   - 继承 `Layer` 实现顶层模式层（或复用现有层）
   - 新模式自动通过 `__init_subclass__` 注册

2. **新增协议层**：
   - 继承 `Layer` 实现新协议处理
   - 在 `next_layer` addon 中添加决策逻辑
   - 或通过自定义 addon 监听 `NextLayerHook`

3. **功能扩展**：
   - 通过 addon 系统监听各种钩子
   - 无需修改核心代码

### 7.3 统一与灵活的平衡

| 方面 | 如何实现 |
|-----|---------|
| **统一的超时管理** | `TimeoutWatchdog` 应用于所有连接 |
| **统一的生命周期** | `ClientConnectedHook`、`ClientDisconnectedHook` 等 |
| **统一的流量控制** | `ConnectionHandler.server_event()` 处理所有命令 |
| **统一的日志格式** | `log_prefix` 包含客户端地址 |
| **灵活的模式层** | 每种模式可以有完全不同的握手逻辑 |
| **灵活的层栈** | `NextLayer` 支持动态决策，甚至可由用户自定义 |
| **灵活的传输层** | 支持 TCP、UDP，抽象为统一的 `Stream` 接口 |

---

## 八、代码参考索引

### 8.1 核心文件

| 文件路径 | 职责 |
|