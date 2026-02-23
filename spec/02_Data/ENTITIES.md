# 数据实体（内存模型）规格

> 本项目没有数据库；“实体（Entity）”指运行期的**内存结构体**、以及前后端交互时的**JSON/二进制消息结构**。  
> 事实源：`src/server.h`、`src/server.c`、`src/http.c`、`src/protocol.c`、`src/pty.h`、`src/pty.c`、`html/src/**`。

---

## 1. 全局配置实体：`struct server`

### 1.1 目的（Purpose）
保存 `ttyd` 进程级配置、运行状态与资源句柄（尤其是 `uv_loop_t*`），供 HTTP/WS/PTY 子系统共享使用。

### 1.2 字段定义（Fields）
字段来自 `src/server.h:64-86`。下面给出“语义 + 默认值 + 所有权/生命周期”。

| 字段 | 类型 | 默认值 | 语义 | 生命周期/所有权 |
|---|---|---|---|---|
| `client_count` | `int` | `0` | 当前 WS 客户端连接数（只在 WS 层增减）。 | 进程生命周期内有效。 |
| `prefs_json` | `char*` | `NULL` → 构造后非空 | 服务端偏好设置 JSON 字符串，会通过 WS 初始消息下发给前端。 | 在 `main()` 解析 CLI 后由 `strdup(json_object_to_json_string(...))` 生成；`server_free()` 释放。 |
| `credential` | `char*` | `NULL` | Basic Auth 的 base64( `username:password` )；同时作为 `/token` 返回值与 WS `AuthToken` 校验值。 | CLI `-c` 设置后 `strdup()`；`server_free()` 释放。 |
| `auth_header` | `char*` | `NULL` | 反向代理鉴权 header 名（会在启动后被 lower-case 且追加 `:`）。 | CLI `-H` 设置后 `strdup()`，启动后 `xrealloc` 扩容并 `strcat(":")`；`server_free()` 释放。 |
| `index` | `char*` | `NULL` | 自定义 index.html 文件路径（CLI `-I`）。 | `strdup`/`malloc`；`server_free()` 释放。 |
| `command` | `char*` | `NULL` | 由 `<command> [args...]` 拼成的人类可读命令行（空格拼接）。用于日志与窗口标题。 | `server_new()` 生成；`server_free()` 释放。 |
| `argv` | `char**` | `NULL` | 子进程 argv（拷贝自 CLI command 部分）。末尾以 `NULL` 结尾。 | `server_new()` 分配数组并 `strdup` 每项；`server_free()` 逐项释放再释放数组。 |
| `argc` | `int` | `0` | `argv` 的数量（不含末尾 NULL）。 | 同上。 |
| `cwd` | `char*` | `NULL` | 子进程工作目录（CLI `-w`）。 | `strdup`；`server_free()` 释放。 |
| `sig_code` | `int` | `SIGHUP` | 客户端断开时 kill 子进程使用的信号（Unix）；Windows 分支无信号概念，使用 `TerminateProcess`。 | `server_new()` 默认；CLI `-s` 覆盖。 |
| `sig_name` | `char[20]` | `"SIGHUP"` | `sig_code` 的人类可读名称。 | 在设置 `sig_code` 后用 `get_sig_name()` 填充。 |
| `url_arg` | `bool` | `false` | 是否允许浏览器通过 URL query `arg=` 追加子进程参数。 | CLI `-a` 打开。 |
| `writable` | `bool` | `false` | 是否允许客户端发送 `INPUT` 写入 PTY。默认只读。 | CLI `-W` 打开。 |
| `check_origin` | `bool` | `false` | 是否启用 Origin 校验（WS 连接过滤）。 | CLI `-O` 打开。 |
| `max_clients` | `int` | `0` | 最大允许连接数；0 表示不限。 | CLI `-m` 设置。 |
| `once` | `bool` | `false` | 是否只允许一个客户端，并在断开后退出 ttyd。 | CLI `-o` 打开。 |
| `exit_no_conn` | `bool` | `false` | 当所有客户端断开时退出 ttyd。 | CLI `-q` 打开。 |
| `socket_path` | `char[255]` | `""` | UNIX 域 socket 路径（当 `--interface` 以 `.sock/.socket` 结尾时设置）。 | `server_free()` 会尝试 `unlink()`。 |
| `terminal_type` | `char[30]` | `"xterm-256color"` | 子进程环境变量 `TERM` 的值。 | CLI `-T` 覆盖（`strncpy`）。 |
| `loop` | `uv_loop_t*` | 非空 | libuv loop。lws 被配置为使用该 “foreign loop”。 | `server_new()` 分配并 `uv_loop_init`；`server_free()` `uv_loop_close` 后释放。 |

### 1.3 初始化与释放（Lifecycle）

- 初始化：`server_new(argc, argv, start)`（`src/server.c:163-204`）
  - 当 `start==argc`（无命令）时直接返回仅含默认值的 server（但随后会被 `missing start command` 拒绝）。
  - 会拷贝 CLI 中 `<command> [args...]` 段到 `argv/argc/command`。
  - 创建 `loop` 并 `uv_loop_init()`。
- 释放：`server_free()`（`src/server.c:206-230`）
  - 释放动态字符串、argv、prefs_json、loop
  - 若 `socket_path` 非空且存在，则 `unlink(socket_path)`（用于 UNIX 域 socket 清理）

---

## 2. URL 路径实体：`struct endpoints`

### 2.1 目的
集中保存 HTTP/WS 路由路径，便于 `--base-path` 统一前缀化。

### 2.2 字段
来自 `src/server.h:20-25`：

| 字段 | 默认值 | 语义 |
|---|---|---|
| `ws` | `"/ws"` | WebSocket 端点路径 |
| `index` | `"/"` | index 页面路径 |
| `token` | `"/token"` | token 端点路径 |
| `parent` | `""` | base-path 的“父路径”；用于将 `/base` 302 到 `/base/`（见 `src/http.c:133-142`） |

### 2.3 重写与内存所有权
事实源：`src/server.c:446-457`。

- 默认值指向**字符串字面量**；当启用 `--base-path` 时会 `strdup()` 新路径并写回字段。
- 现有实现不会在退出前显式 free `endpoints.*`（只在进程结束时由 OS 回收）。复刻实现可更严谨（记录所有权并释放），但对外行为应保持一致。

---

## 3. HTTP 会话实体：`struct pss_http`

### 3.1 目的
保存单个 HTTP 连接/请求的状态，用于分块发送内嵌 HTML 或 token JSON。

### 3.2 字段（`src/server.h:32-37`）

| 字段 | 类型 | 语义 |
|---|---|---|
| `path` | `char[128]` | 当前请求路径（例如 `/`、`/token`）。由 `LWS_CALLBACK_HTTP` 时 `snprintf` 写入。 |
| `buffer` | `char*` | 正在发送的 body buffer。可能指向：`strdup` 的 JSON，或 `index_html` 静态数组，或解压缓存 `html_cache`。 |
| `ptr` | `char*` | 当前发送指针（指向 `buffer` 内部偏移）。 |
| `len` | `size_t` | body 总长度。 |

### 3.3 关键所有权规则
事实源：`src/http.c:82-84`。

- 当 `buffer` 指向 `index_html` 或 `html_cache`（静态/缓存）时，不得 `free`；
- 其他情况（例如 token JSON `strdup(buf)`）在发送完成后必须释放。

---

## 4. WS 会话实体：`struct pss_tty`

### 4.1 目的
保存单条 WebSocket 连接对应的终端会话状态：鉴权、URL args、接收缓冲、PTY process 与待发送输出缓冲等。

### 4.2 字段（`src/server.h:39-57`）

| 字段 | 类型 | 语义 |
|---|---|---|
| `initialized` | `bool` | 是否已发送完初始消息（title/prefs）。未 initialized 时 `SERVER_WRITEABLE` 发送初始命令。 |
| `initial_cmd_index` | `int` | 初始命令下标（`SET_WINDOW_TITLE`、`SET_PREFERENCES`）。 |
| `authenticated` | `bool` | WS token 鉴权是否通过（仅当启用 `server->credential` 时有效）。 |
| `user` | `char[30]` | 在 `--auth-header` 模式下，从 header 复制来的用户标识。用于注入 `TTYD_USER`。 |
| `address` | `char[50]` | 对端地址字符串，用于日志。 |
| `path` | `char[128]` | 本次 WS 请求的 URI path。用于合法性校验与日志。 |
| `args` | `char**` | 通过 URL query `arg=` 追加的参数数组（每项 `strdup`）。 |
| `argc` | `int` | `args` 数量。 |
| `wsi` | `struct lws*` | 当前 WS 连接句柄。 |
| `buffer` | `char*` | WS 接收缓冲（用于处理 fragmented frame 拼接）。 |
| `len` | `size_t` | `buffer` 当前长度。 |
| `process` | `pty_process*` | 关联的 PTY 子进程对象（每个 WS 连接最多一个）。 |
| `pty_buf` | `pty_buf_t*` | 待发送给客户端的 PTY 输出缓冲（由 PTY read_cb 填充）。 |
| `lws_close_status` | `int` | 请求 lws 关闭连接的状态码（例如 1000/1006）。 |

### 4.3 生命周期要点
事实源：`src/protocol.c`。

- 建连（`LWS_CALLBACK_ESTABLISHED`）：
  - `initialized=false`、`authenticated=false`、`wsi=wsi`、`lws_close_status = NOSTATUS`
  - 若启用 `--url-arg`：解析 query 中的 `arg=` fragment 并追加到 `args`
  - `server->client_count++`
- 首包 JSON（`command == '{'`）：
  - 若 `server->credential != NULL`：校验 `AuthToken`；失败则 close_reason POLICY_VIOLATION 并返回 `-1`
  - 解析 `columns/rows` 并 spawn 子进程，写入 `pss->process`
- 运行中：接收 INPUT/RESIZE/PAUSE/RESUME；输出 OUTPUT；退出时设置 close_status 并关闭
- 断开（`LWS_CALLBACK_CLOSED`）：
  - `client_count--`
  - 释放接收 buffer、`pty_buf`、`args[i]`
  - 若 `process_running` 则 `pty_kill`（并标记 `ws_closed=true`）
  - 若 `--once/--exit-no-conn` 且 `client_count==0`：直接 `exit(0)`

---

## 5. PTY 子进程实体：`pty_process`

### 5.1 目的
跨平台封装“一个可读写的伪终端子进程”。

### 5.2 字段（见 `src/pty.h:29-54`）

字段较多，复刻实现需至少满足以下语义：

| 字段 | 语义 |
|---|---|
| `pid` | 子进程 pid（Windows 为 `dwProcessId`）。 |
| `exit_code` / `exit_signal` | 子进程退出码/信号（Unix：waitpid 解析；Windows：`GetExitCodeProcess`）。 |
| `columns`/`rows` | 初始终端尺寸（默认 80x24，可由 WS 首包覆盖）。 |
| `argv` | 子进程 argv（由 `server->argv` + `pss->args` 拼接而成，最后 `NULL` 结尾）。 |
| `envp` | 仅包含 `TERM` 与可选 `TTYD_USER` 的 env 数组（最后 `NULL` 结尾）。 |
| `cwd` | 子进程工作目录（可选）。 |
| `loop` | libuv loop，用于创建 pipe/async。 |
| `in`/`out` | `uv_pipe_t*`：子进程输入与输出管道（Unix master FD dup 两份；Windows named pipe connect）。 |
| `paused` | “暂停”标志位（现有实现存在不一致：spawn 时置 true，resume 不会置 false）。复刻时需保持一致行为。 |
| `read_cb` / `exit_cb` | 回调：读到数据/进程退出。 |
| `ctx` | 上下文指针：由 `protocol.c` 传入 `pty_ctx_t`。 |

### 5.3 spawn 的跨平台语义（必须复刻）
事实源：`src/pty.c`。

- Unix：
  - `forkpty(&master, NULL, NULL, &winsize)`
  - child：`setsid()`、可选 `chdir(cwd)`、`putenv` 写入 envp、`execvp(argv[0], argv)`
  - parent：`master` 设为 `O_NONBLOCK` + `FD_CLOEXEC`
  - `uv_pipe_open` 连接到 dup 出来的 `master`，分别作为 `in`/`out`
  - 创建 `uv_thread` 等待 `waitpid`，并通过 `uv_async_send` 回到 loop 线程执行退出回调
- Windows：
  - 动态加载 ConPTY API（`conpty_init()`）
  - 创建 named pipe，调用 `CreatePseudoConsole`，将伪控制台句柄挂到 `STARTUPINFOEXW` attribute list
  - 使用 `_wputenv` 写 env（注意：这会影响当前进程环境）
  - `CreateProcessW` 创建子进程
  - `RegisterWaitForSingleObject` 监听退出并 `uv_async_send`

---

## 6. 前端配置实体（Preferences / Options）

### 6.1 `ClientOptions`（前端自定义开关）
事实源：`html/src/components/terminal/xterm/index.ts:42-54` 与 `html/src/components/app.tsx:12-22`。

关键字段（需复刻）：
- `rendererType`: `'dom' | 'canvas' | 'webgl'`
- `disableLeaveAlert`: 控制 beforeunload 提示
- `disableResizeOverlay`: 控制 resize overlay
- `enableZmodem` / `enableTrzsz` / `enableSixel`
- `closeOnDisconnect`
- `isWindows`
- `unicodeVersion`
- `titleFixed?`
- `trzszDragInitTimeout`

### 6.2 `ITerminalOptions`（xterm.js options）
事实源：`html/src/components/app.tsx:23-48`。

必须复刻的默认值（部分）：
- `fontSize = 13`
- `fontFamily = 'Consolas,Liberation Mono,Menlo,Courier,monospace'`
- `theme`：见 `ui-ux-spec/01_Foundation/FOUNDATION.md`（必须写出所有颜色 hex）
- `allowProposedApi = true`

### 6.3 `FlowControl`
事实源：`html/src/components/app.tsx:49-53`。

默认值：
- `limit = 100000`
- `highWater = 10`
- `lowWater = 4`

