# 错误、关闭码与退出码（Errors）

> `ttyd` 不提供统一的 JSON 错误格式；错误通过以下方式体现：  
> - HTTP 状态码 + 头部（401/407/404/302 等）  
> - WebSocket 关闭码（close status）或握手阶段拒绝（filter return 1）  
> - 进程标准错误输出（stderr）  
> - libwebsockets 日志（`lwsl_*`）  
> - 进程退出码（main 返回值 / `exit()`）

事实源：`src/server.c`、`src/http.c`、`src/protocol.c`、`src/pty.c`。

---

## 1. HTTP 错误与非 200 行为

### 1.1 鉴权失败
事实源：`src/http.c:14-44`。

| 场景 | HTTP 状态 | 关键响应头 | body |
|---|---:|---|---|
| `--auth-header` 模式缺失 header | `407 Proxy Auth Required` | `Proxy-Authenticate: Basic realm="ttyd"`；`Content-Length: 0` | 空 |
| `--credential` 模式缺失/不匹配 Authorization | `401 Unauthorized` | `WWW-Authenticate: Basic realm="ttyd"`；`Content-Length: 0` | 空 |

> 注意：两种模式都返回 `Basic realm="ttyd"`（现有实现如此，复刻需保持）。

### 1.2 Not Found
事实源：`src/http.c:144-147`。

| 场景 | HTTP 状态 |
|---|---:|
| path 既不是 `endpoints.token`、也不是 `endpoints.parent`、也不是 `endpoints.index` | `404 Not Found` |

### 1.3 base-path redirect
事实源：`src/http.c:133-142`。

| 场景 | HTTP 状态 |
|---|---:|
| 请求路径等于 `endpoints.parent` | `302 Found`（Location 指向 `endpoints.index`） |

### 1.4 TLS 客户端证书校验失败（可选）
事实源：`src/http.c:218-227`（仅在 OpenSSL 且非 mbedTLS 分支可用）。

当 `--ssl-ca` 启用并要求客户端证书时，若证书校验失败：
- 输出错误日志（包含 verify 错误、depth 等）
- callback 返回 `1`（请求失败）

---

## 2. WebSocket：拒绝连接（Filter 阶段）

事实源：`src/protocol.c:203-229`。

在 `LWS_CALLBACK_FILTER_PROTOCOL_CONNECTION` 阶段，服务端通过 `return 1` 直接拒绝连接，常见原因：

| 场景 | 触发条件 | 结果 |
|---|---|---|
| `--once` 已有客户端 | `server->once && server->client_count > 0` | 拒绝建立 WS |
| 达到 `--max-clients` | `max_clients>0 && client_count==max_clients` | 拒绝建立 WS |
| 鉴权失败 | `check_auth()==false` | 拒绝建立 WS |
| WS path 不合法 | `strncmp(pss->path, endpoints.ws, n) != 0` | 拒绝建立 WS |
| `--check-origin` 不通过 | `check_host_origin()==false` | 拒绝建立 WS |

> 该阶段没有统一的 close_reason body；客户端通常表现为连接失败或握手失败。

---

## 3. WebSocket：连接建立后关闭（Close Codes）

### 3.1 token 鉴权失败（Policy Violation）
事实源：`src/protocol.c:333-346`。

当启用 `--credential` 且首包 JSON 的 `AuthToken` 不匹配：
- 服务端调用 `lws_close_reason(wsi, LWS_CLOSE_STATUS_POLICY_VIOLATION, NULL, 0)`
- callback 返回 `-1`

### 3.2 子进程退出触发关闭（1000 / 1006）
事实源：`src/protocol.c:81-93, 95-106`。

当读回调认为 EOF 且进程已不再运行，或 exit_cb 触发时，会设置 `pss->lws_close_status`：

| 场景 | close status |
|---|---:|
| 子进程退出码 `exit_code == 0` | `1000`（正常关闭） |
| 子进程退出码 `exit_code != 0` | `1006`（异常/非正常） |

随后在 `LWS_CALLBACK_SERVER_WRITEABLE` 中检测到 `lws_close_status > NOSTATUS` 时：
- `lws_close_reason(wsi, pss->lws_close_status, NULL, 0)`
- callback 返回 `1`（触发关闭）

### 3.3 客户端主动断开触发 kill 子进程
事实源：`src/protocol.c:362-388`。

当 WS CLOSED 时：
- 若 `pss->process` 仍在运行：
  - `pty_pause(process)`
  - `pty_kill(process, server->sig_code)`（Unix）或 `TerminateProcess`（Windows）
- 若启用 `--once` 或 `--exit-no-conn` 且此时 `client_count==0`：
  - 进程会 `exit(0)` 立即退出

---

## 4. 进程启动/参数解析错误

事实源：`src/server.c`。

| 场景 | 输出/行为 | 退出 |
|---|---|---|
| `--credential` 无 `:` | 输出 `ttyd: invalid credential...` | `return -1` |
| `--signal` 无效 | 输出 `ttyd: invalid signal: ...` | `return -1` |
| `--index` stat 失败/是目录 | 输出错误信息 | `return -1` |
| 缺少 `<command>` | 输出 `ttyd: missing start command` | `return -1` |
| ConPTY init 失败（Windows） | 输出 `ERROR: ConPTY init failed...` | `return 1` |

> 备注：`main()` 返回 `-1` 时，shell 中的实际退出码通常为 `255`（取决于平台/壳层），复刻实现可选择直接 `exit(EXIT_FAILURE)` 以更明确，但若要做到行为一致需保持返回值语义。

---

## 5. 运行期关键错误（日志）

### 5.1 libwebsockets context/vhost 创建失败
事实源：`src/server.c:594-604`。

- `context == NULL`：日志 `libwebsockets context creation failed`，返回 `1`
- `vhost == NULL`：日志 `libwebsockets vhost creation failed`，返回 `1`

### 5.2 PTY spawn 失败
事实源：`src/protocol.c:155-159`。

- `pty_spawn(...) != 0`：
  - 日志：`pty_spawn: <errno> (<strerror>)`
  - `process_free(process)`
  - 返回 `false`（调用方 `spawn_process` 失败）
  - WS receive 分支会 `return 1`（连接将被关闭）

### 5.3 写入 PTY 失败
事实源：`src/protocol.c:310-314`。

- `pty_write(...)` 返回错误码：
  - 日志：`uv_write: <uv_err_name> (<uv_strerror>)`
  - callback 返回 `-1`

---

## 6. 前端可见错误/提示（与协议相关）

事实源：`html/src/components/terminal/xterm/index.ts`。

| 场景 | 前端 overlay 文案（逐字） |
|---|---|
| WS close 事件触发 | `Connection Closed` |
| 非正常关闭且允许自动重连 | `Reconnecting...` |
| 重新连接成功 | `Reconnected` |
| 正常关闭或不自动重连：等待用户按 Enter | `Press ⏎ to Reconnect` |

> 这些字符串属于 UI 可见文本，复刻实现必须逐字一致（详见 `ui-ux-spec/**`）。

