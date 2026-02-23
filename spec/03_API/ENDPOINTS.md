# 对外接口规格：HTTP + WebSocket（Endpoints）

> 本文定义 `ttyd` 对外暴露的 HTTP 路由与 WebSocket 协议端点。  
> 事实源：`src/http.c`、`src/protocol.c`、`src/server.c`、`src/server.h`、以及前端实现 `html/src/components/terminal/xterm/index.ts`。

---

## 0. 路径命名与 base-path 规则

### 0.1 默认端点
全局 `endpoints` 默认值（`src/server.c:23`）：

| 逻辑端点 | 默认路径 |
|---|---|
| WS | `/ws` |
| index | `/` |
| token | `/token` |
| parent | `""` |

### 0.2 `--base-path` 重写
若启用 `--base-path <base>`（`src/server.c:446-457`）：

- 会去掉 `<base>` 末尾的所有 `/`；
- 然后将四个端点统一变为：`<base> + <原端点>`；
- `parent` 也会被设置为 `<base>`（因为原值为 `""`）。

举例：
- `--base-path /mounted/here/` → 实际：
  - index: `/mounted/here/`
  - token: `/mounted/here/token`
  - ws: `/mounted/here/ws`
  - parent: `/mounted/here`

---

## 1. HTTP：鉴权前置规则（所有 HTTP 路由）

`callback_http` 在处理任何路径前都会执行 `check_auth()`（`src/http.c:103-111`），其规则详见 `spec/03_API/AUTHENTICATION.md`。本文只描述各端点的行为；鉴权失败时的状态码见 `spec/03_API/ERRORS.md`。

---

## 2. HTTP：`GET <token>`（默认 `/token`）

事实源：`src/http.c:116-131`。

### 2.1 用途
向浏览器返回 WS 首包 JSON 的 `AuthToken` 字段值（当启用 `--credential` 时用于 WS token 校验；即使未启用也会返回空串）。

### 2.2 请求
- 方法：`GET`
- Path：`endpoints.token`（默认 `/token`）
- Body：无
- Query：无

### 2.3 响应：200 OK
当鉴权通过时：
- Status：200
- Header：
  - `Content-Type: application/json;charset=utf-8`
  - `Content-Length: <n>`
- Body（**注意：是 JSON 字符串，token 值可能为空**）：

```json
{"token":"<token>"}
```

其中 `<token>` 的取值规则：
- 若启用 `--credential`：为 base64 编码后的 `username:password`（`server->credential`）
- 否则：空字符串 `""`

### 2.4 与 WebSocket 的契约（必须一致）
前端行为（事实源：`html/src/components/terminal/xterm/index.ts:131-141, 263-266`）：
- 前端会 `fetch(tokenUrl)` 读取 `json.token` 并保存到 `this.token`；
- WS open 时发送首包 JSON：`{ AuthToken: this.token, columns, rows }`。

---

## 3. HTTP：`GET <index>`（默认 `/`）

事实源：`src/http.c:144-179`。

### 3.1 用途
返回 Web UI 的 HTML 页面（内嵌 SPA 资源）；或返回用户指定的 `--index` 文件。

### 3.2 请求
- 方法：`GET`
- Path：`endpoints.index`（默认 `/`）
- Query：可有（前端会使用 `window.location.search`；服务端对 index 本身不解析 query）

### 3.3 响应：自定义 index 文件模式（`--index`）
当 `server->index != NULL` 时：
- 使用 `lws_serve_http_file(wsi, server->index, "text/html", ...)` 返回该文件内容（`src/http.c:149-153`）。
- 若 `lws_serve_http_file` 返回 `<0` 或返回 `>0` 且事务完成，则 callback 返回 `1`（表示错误/结束）。

### 3.4 响应：内嵌 index 模式（默认）

#### 3.4.1 Content-Type
始终为 `text/html`（`src/http.c:149`）。

#### 3.4.2 gzip/解压策略（必须复刻）
内嵌资源 `index_html[]` 本质上是 gzip bytes（其首字节为 `0x1f 0x8b 0x08 ...`，见 `src/html.h`），服务端会根据编译宏与请求头选择“直接返回 gzip”或“解压后返回”：

- 若定义 `LWS_WITH_HTTP_STREAM_COMPRESSION`：
  - 服务端强制 `uncompress_html()` 解压到 `html_cache` 并返回未压缩 HTML（`src/http.c:160-162`）。
- 否则：
  - 若请求头 `Accept-Encoding` 包含 `gzip`：
    - 返回原始 `index_html` gzip bytes，并添加 `Content-Encoding: gzip`（`src/http.c:163-165`）
  - 否则：
    - 解压到 `html_cache` 并返回未压缩 HTML（`src/http.c:166-168`）

#### 3.4.3 缓存行为
`uncompress_html()` 会把解压结果缓存到静态 `html_cache` 中，仅首次请求解压（`src/http.c:52-80`）。

---

## 4. HTTP：`GET <parent>`（base-path redirect）

事实源：`src/http.c:133-142`。

### 4.1 用途
当启用 `--base-path` 时，将 `/base-path` 302 重定向到 `/base-path/`（即 `endpoints.index`）。

### 4.2 条件
仅当 `strcmp(pss->path, endpoints.parent) == 0` 时触发。默认 `endpoints.parent == ""`，因此默认不触发。

### 4.3 响应：302 Found
- Status：302
- Header：
  - `Location: <endpoints.index>`
  - `Content-Length: 0`
- Body：空

---

## 5. HTTP：Not Found

事实源：`src/http.c:144-147`。

- 当 path 既不是 `token`、也不是 `parent`、也不是 `index` 时：
  - `lws_return_http_status(wsi, HTTP_STATUS_NOT_FOUND, NULL);`
  - 之后尝试复用连接（HTTP keep-alive）；若事务完成则返回 `-1` 让 lws 回收。

---

## 6. WebSocket：`GET <ws>`（默认 `/ws`，子协议 `tty`）

事实源：`src/protocol.c`、`src/server.h` 与前端 `html/src/components/terminal/xterm/index.ts`。

### 6.1 握手与子协议

- Path：`endpoints.ws`（默认 `/ws`，可能被 `--base-path` 前缀化）
- 子协议（subprotocol）：`tty`
  - 前端创建：`new WebSocket(wsUrl, ['tty'])`（`html/src/components/terminal/xterm/index.ts:249-257`）
  - 后端协议表：`{"tty", callback_tty, sizeof(pss_tty), 0}`（`src/server.c:28-31`）

### 6.2 WS 连接过滤（Filter）
在 `LWS_CALLBACK_FILTER_PROTOCOL_CONNECTION` 阶段进行以下检查（`src/protocol.c:203-229`）：

1) `--once`：若 `server->once && server->client_count > 0` → 拒绝连接（return 1）
2) `--max-clients`：若 `max_clients>0 && client_count == max_clients` → 拒绝连接（return 1）
3) 鉴权：`check_auth(wsi, pss)` 为 false → 拒绝连接（return 1）
4) 路径合法性：
   - 读取 `WSI_TOKEN_GET_URI`（HTTP/1.1）或 `WSI_TOKEN_HTTP_COLON_PATH`（HTTP/2）
   - `strncmp(pss->path, endpoints.ws, n) != 0` → 拒绝连接（return 1）
5) `--check-origin`：若启用且 `check_host_origin(wsi)==false` → 拒绝连接（return 1）

### 6.3 WS 首包 JSON（Hello message）

#### 6.3.1 前端发送
WS open 时发送（非命令帧，直接 JSON 字符串）：

```json
{"AuthToken":"<token>","columns":<cols>,"rows":<rows>}
```

事实源：`html/src/components/terminal/xterm/index.ts:263-266`。

#### 6.3.2 服务端解析与鉴权
服务端在 `LWS_CALLBACK_RECEIVE` 中，当 `command == '{'`（即首字节为 `{`）时进入 JSON_DATA 分支（`src/protocol.c:328-350`）：

- 若 `pss->process != NULL`：忽略（break）
- 解析 `columns/rows`（从 JSON 里取 `columns` 与 `rows`，见 `src/protocol.c:38-48`）
- 若启用 `--credential`（`server->credential != NULL`）：
  - 从 JSON 取 `AuthToken`，要求 `AuthToken == server->credential`（字符串相等）
  - 若不匹配：`lws_close_reason(... POLICY_VIOLATION ...)` 并返回 `-1`（拒绝）
- 通过后 spawn 子进程，并让后续 writeable 回调发送初始消息

### 6.4 WS 业务帧：命令字节 + payload

#### 6.4.1 客户端 → 服务端
事实源：`src/server.h:7-12`、`src/protocol.c:307-354`。

| 命令 | 值 | payload | 服务端动作 |
|---|---:|---|---|
| INPUT | `'0'` | bytes | 若 `--writable` 启用则 `pty_write(payload)`；否则忽略。 |
| RESIZE_TERMINAL | `'1'` | JSON：`{"columns":..,"rows":..}` | 更新 `process->columns/rows` 并 `pty_resize()`。 |
| PAUSE | `'2'` | 空 | `pty_pause(process)` |
| RESUME | `'3'` | 空 | `pty_resume(process)` |

鉴权门槛（必须复刻）：
- 若启用 `--credential`，且 `pss->authenticated==false`，并且当前 message 不是 JSON_DATA，则服务端直接 `return 1`（`src/protocol.c:296-300`）。  
  也就是说：在启用 credential 时，客户端**必须先发送首包 JSON 完成 token 鉴权**，才能发送 INPUT/RESIZE/PAUSE/RESUME。

#### 6.4.2 服务端 → 客户端
事实源：`src/server.h:14-17`、`src/protocol.c:13-36, 167-181`。

| 命令 | 值 | payload | 客户端动作 |
|---|---:|---|---|
| OUTPUT | `'0'` | PTY bytes | 写入 xterm（并触发前端流控 pending 统计）。 |
| SET_WINDOW_TITLE | `'1'` | UTF-8 文本 | 设置 `document.title`。 |
| SET_PREFERENCES | `'2'` | UTF-8 JSON | 合并默认 prefs + server prefs + URL query prefs，并应用。 |

### 6.5 WS 初始消息（title/prefs）
服务端在 `LWS_CALLBACK_SERVER_WRITEABLE` 中，在 `initialized==false` 时发送两条初始消息（`src/protocol.c:253-267`）：

1) `SET_WINDOW_TITLE`：payload 为 `"<server->command> (<hostname>)"`  
2) `SET_PREFERENCES`：payload 为 `server->prefs_json`

完成后 `initialized=true` 并调用 `pty_resume(process)`。

### 6.6 WS 关闭语义
见 `spec/03_API/ERRORS.md`（关闭码、policy violation、process exit 触发关闭等）。

