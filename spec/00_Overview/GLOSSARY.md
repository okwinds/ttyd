# 术语表（Glossary）

> 用途：让实现者在不阅读源码的情况下理解本仓库的关键概念。  
> 约束：定义以源码中“行为与数据结构”为准（本规格提取阶段不读取仓库现存文档）。

## A. 核心对象与结构体（后端）

| 术语 | 定义 | 代码定位（事实源） |
|---|---|---|
| `server` | 全局 `struct server *server`，保存运行期配置与资源（uv loop、鉴权配置、base-path、once/max-clients 等）。 | `src/server.h:64-86`、`src/server.c:316-317` |
| `endpoints` | 全局 URL 路径集合：ws/index/token/parent。默认 `{"/ws","/","/token",""}`，可被 `--base-path` 前缀化。 | `src/server.h:20-25`、`src/server.c:23-24`、`src/server.c:446-457` |
| `wsi` | libwebsockets 的连接句柄 `struct lws*`。 | 贯穿 `src/http.c`、`src/protocol.c` |
| `pss_http` | HTTP per-session 数据：记录 path 与 HTTP 分块发送的 buffer/ptr/len。 | `src/server.h:32-37`、`src/http.c` |
| `pss_tty` | WS per-session 数据：鉴权状态、URL arg、累积接收 buffer、关联 PTY process、待发送输出 buffer 等。 | `src/server.h:39-57`、`src/protocol.c` |
| `pty_process` | 子进程抽象：封装 PTY/ConPTY、in/out uv pipe、退出回调等。 | `src/pty.h:29-54`、`src/pty.c` |
| `pty_buf_t` | PTY 输出缓冲：`base`（字节数组）+ `len`。 | `src/pty.h:19-23`、`src/pty.c:49-61` |
| `pty_ctx_t` | `pty_process.ctx` 的上下文：指回 `pss_tty` 并标记 `ws_closed`。用于 WS 已关闭时丢弃后续 PTY 输出。 | `src/server.h:59-62`、`src/protocol.c:72-110` |

## B. WebSocket 业务帧协议（命令字节 + payload）

> WS 中业务数据以 **binary frame** 传输，格式为：  
> - **命令码**：1 字节（ASCII 字符）  
> - **payload**：命令相关的数据（可能为空）  
> 命令码定义在后端 `src/server.h`，前端 `html/src/components/terminal/xterm/index.ts` 用 `enum Command` 对齐。

### B.1 客户端 → 服务端（client message）

| 命令 | 值 | payload 语义 |
|---|---:|---|
| `INPUT` | `'0'` | 要写入 PTY 的字节流。前端既可能发送 UTF-8 文本，也可能发送二进制（`onBinary`）。 |
| `RESIZE_TERMINAL` | `'1'` | UTF-8 JSON：`{"columns":<int>,"rows":<int>}`。 |
| `PAUSE` | `'2'` | 无 payload。流控：请求服务端暂停输出读取。 |
| `RESUME` | `'3'` | 无 payload。流控：请求服务端恢复输出读取。 |
| `JSON_DATA` | `'{'` | **首包 JSON** 的首字符。连接建立后客户端先发送 JSON（不带命令字节前缀），服务端通过首字符 `{` 识别并解析。 |

### B.2 服务端 → 客户端（server message）

| 命令 | 值 | payload 语义 |
|---|---:|---|
| `OUTPUT` | `'0'` | PTY 输出字节流（原样）。 |
| `SET_WINDOW_TITLE` | `'1'` | UTF-8 文本。前端设置 `document.title`。 |
| `SET_PREFERENCES` | `'2'` | UTF-8 JSON 字符串。前端合并默认配置、服务端 prefs、URL query overrides 后应用。 |

## C. 关键概念

| 术语 | 定义 |
|---|---|
| PTY | Pseudo Terminal。Unix-like 使用 `forkpty()` 创建子进程与 master FD。 |
| ConPTY | Windows pseudo console。通过 `CreatePseudoConsole` 等 API 管理，并用 named pipe 与进程 I/O 连接。 |
| 首包 JSON（Hello） | WS 建连后客户端发送的 JSON：`{ AuthToken, columns, rows }`。服务端解析后 spawn 子进程，并（当启用 credential 时）校验 `AuthToken`。 |
| base-path | `--base-path` 将默认端点 `/`、`/token`、`/ws` 等添加前缀，适配反向代理挂载路径。 |
| Origin 校验 | `--check-origin` 启用后，要求 `Origin` 解析出来的 host[:port] 与 `Host` 头一致，否则拒绝 WS。 |

## D. 缩写（Abbreviations）

| 缩写 | 全称 |
|---|---|
| LWS | libwebsockets |
| UV | libuv |
| WS | WebSocket |
| PSS | per-session storage（lws 的 per-connection user 数据） |

