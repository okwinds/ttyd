# 业务规则（Business Rules）

> “业务规则”指运行期必须遵守的约束与分支行为，复刻实现必须逐条满足。  
> 事实源：`src/server.c`、`src/http.c`、`src/protocol.c`、`src/pty.c`、前端 `html/src/**`。

---

## 规则 1：启动命令必填

- **描述**：`ttyd` 必须有 `<command> [args...]`；否则拒绝启动。
- **触发**：CLI 解析后，准备创建 lws context 前。
- **实现逻辑**（事实源：`src/server.c:529-532`）：
  - 若 `server->command == NULL || strlen(server->command)==0`：
    - 输出：`ttyd: missing start command`
    - `return -1`

---

## 规则 2：HTTP/WS 的鉴权策略二选一（`--auth-header` 优先）

- **描述**：当 `--auth-header` 设置时，HTTP 与 WS 都只检查该 header；Basic credential 即使设置也不会用于 HTTP/WS header 校验，但 WS 首包 token 校验仍取决于 `server->credential != NULL`。
- **触发**：HTTP `check_auth()` 与 WS `check_auth()`。
- **实现事实**：
  - HTTP：`if (server->auth_header != NULL) { ... } else if (server->credential != NULL) { ... }`（`src/http.c:29-44`）
  - WS：同样优先 `auth_header`（`src/protocol.c:183-195`）
- **边界**：
  - `--auth-header` 模式下，WS 会把 header 值复制到 `pss->user`，并注入 `TTYD_USER`（见规则 9）。

---

## 规则 3：`--base-path` 对四个端点做统一前缀化

- **描述**：`--base-path` 会同时影响 HTTP index/token 与 WS 路径；并额外启用 “parent redirect”。
- **触发**：CLI 解析阶段。
- **实现逻辑**：见 `spec/01_Configuration/CLI_OPTIONS.md` 的 base-path 章节（事实源 `src/server.c:446-457`）。
- **边界**：
  - base-path 输入会 trim 末尾 `/`，空串则忽略。

---

## 规则 4：WS 路径必须匹配 `endpoints.ws`

- **描述**：WS 只接受 path 为 `endpoints.ws` 的请求（含 base-path 前缀）。
- **触发**：`LWS_CALLBACK_FILTER_PROTOCOL_CONNECTION`
- **实现逻辑**（事实源：`src/protocol.c:214-221`）：
  - 从 `WSI_TOKEN_GET_URI`（或 H2 colon path）拷贝 URI 到 `pss->path`
  - 使用 `strncmp(pss->path, endpoints.ws, n) != 0` 判断非法并拒绝
- **可观察结果**：
  - 非法路径会被直接拒绝（return 1），客户端握手失败。

---

## 规则 5：连接数限制（`--once` / `--max-clients`）

- **描述**：
  - `--once`：只允许 1 个客户端；第二个客户端在 filter 阶段被拒绝；最后一个客户端断开后 ttyd 退出。
  - `--max-clients N`：最多 N 个客户端；达到上限后新客户端被拒绝。
- **触发**：WS filter 阶段。
- **实现逻辑**（事实源：`src/protocol.c:204-211, 382-387`）：
  - `once && client_count > 0` → return 1
  - `max_clients > 0 && client_count == max_clients` → return 1
  - WS CLOSED 时：若 `(once || exit_no_conn) && client_count == 0` → `exit(0)`

---

## 规则 6：Origin 校验（`--check-origin`）

- **描述**：当启用 `--check-origin` 时，WS 请求必须满足 `Origin` 的 host[:port] 与 `Host` 头完全一致（忽略大小写）。
- **触发**：WS filter 阶段。
- **实现逻辑**（事实源：`src/protocol.c:50-70, 223-228`）：
  - `lws_hdr_copy(ORIGIN)` 得到完整 Origin URL
  - `lws_parse_uri` 得到 `address` 与 `port`
  - 将 `address[:port]` 与 `Host` 比较（`strcasecmp`）
- **边界**：
  - 若 `Origin` 头缺失或 URI 解析失败 → 校验失败 → WS 被拒绝。

---

## 规则 7：只读/可写（`--writable`）

- **描述**：默认只读；客户端输入不会写入 PTY。
- **触发**：WS RECEIVE 处理 `INPUT`。
- **实现逻辑**（事实源：`src/protocol.c:308-315`）：
  - 若 `!server->writable`：直接 `break`（忽略 INPUT）
  - 否则 `pty_write(process, payload)`

---

## 规则 8：URL 参数注入（`--url-arg`）

- **描述**：允许浏览器用 query `arg=...` 追加子进程参数。
- **触发**：WS ESTABLISHED。
- **实现逻辑**（事实源：`src/protocol.c:237-245`）：
  - 遍历 `WSI_TOKEN_HTTP_URI_ARGS` 的 fragment
  - 对每个 fragment，若以 `arg=` 开头，则将 `fragment[4:]` `strdup()` 放入 `pss->args[]`
- **边界**：
  - 该机制不做 URL decode；`arg=` 后内容按原样作为 argv 字符串。
  - 参数追加顺序与 URI_ARGS fragment 的顺序一致。

---

## 规则 9：子进程环境注入（TERM / TTYD_USER）

- **描述**：每次 spawn 子进程都设置：
  - `TERM=<server->terminal_type>`
  - 可选 `TTYD_USER=<pss->user>`（仅当 auth-header 模式取到 user）
- **触发**：spawn 子进程前构造 envp。
- **实现逻辑**：`build_env()`（事实源：`src/protocol.c:128-148`）。
- **边界（长度/截断）**：
  - `pss->user` 缓冲区大小为 30（`src/server.h:43`），由 lws copy 填充，可能发生截断。
  - `TTYD_USER` envp 单条分配长度为 40（`src/protocol.c:140-142`），`snprintf` 可能截断。

---

## 规则 10：WS 首包 JSON 触发 spawn 且（可选）校验 token

- **描述**：WS 连接建立后，必须由首包 JSON 触发子进程 spawn；当启用 `--credential` 时首包 JSON 还承担 token 校验。
- **触发**：WS RECEIVE 中 `command == '{'`。
- **实现逻辑**：见 `spec/03_API/ENDPOINTS.md` 的“WS 首包 JSON”章节（事实源 `src/protocol.c:328-350`）。

---

## 规则 11：初始消息必须先于终端输出（title/prefs）

- **描述**：服务端会在终端开始输出之前发送两条初始消息：窗口标题与偏好设置。
- **触发**：`LWS_CALLBACK_SERVER_WRITEABLE` 且 `initialized==false`。
- **实现逻辑**：`send_initial_message` + `initial_cmds`（事实源：`src/protocol.c:13-36, 253-267`）。

---

## 规则 12：输出背压与“暂停/恢复”控制（含已知实现细节）

### 12.1 服务端输出节流（每次只发送一个 pty_buf）
事实源：`src/pty.c:63-75` 与 `src/protocol.c:275-280`。

现有实现的节流模型为：

1) PTY `uv_read` 回调触发后，首先 `uv_read_stop(stream)`（即一次只读一段）
2) 将读到的数据封装为 `pty_buf_t`，挂到 `pss->pty_buf`
3) 通过 `lws_callback_on_writable(wsi)` 请求 WS 可写回调
4) 在 `SERVER_WRITEABLE` 中将该 `pty_buf` 写出为 `OUTPUT`，然后 `pty_resume(process)` 继续读取下一段

### 12.2 前端流控（PAUSE/RESUME）
事实源：`html/src/components/terminal/xterm/index.ts:207-227`。

前端用三元组 `limit/highWater/lowWater` 统计写入量：
- 当累计写入超过 `limit` 时，使用回调降低 `pending` 计数；
- 当 `pending > highWater` 时发送命令 `PAUSE`；
- 当 `pending < lowWater` 时发送命令 `RESUME`。

### 12.3 已知实现细节：后端 `paused` 标志位不一致
事实源：`src/pty.c:122-133` 与 `src/pty.c:470-471 / 352`。

现有后端实现中：
- `pty_spawn()` 成功后设置 `process->paused = true`
- `pty_resume()`：
  - 仅在 `process->paused == true` 时启动 `uv_read_start`
  - **不会**把 `paused` 置为 false
- `pty_pause()`：
  - 若 `process->paused == true`，直接 return（不会执行 `uv_read_stop`）
  - 同样不会设置 `paused`

因此，“暂停/恢复”的标志位语义与实际读写行为可能不一致（暂停可能不生效）。  
复刻实现必须以“可观察行为”为准：即便实现更合理，也需要保留该不一致以通过行为一致性验收（建议在复刻实现中用回归测试锁定这一点）。

