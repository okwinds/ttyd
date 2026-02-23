# 鉴权规格（Authentication）

> `ttyd` 的鉴权并非“用户系统”，而是围绕终端共享的轻量校验。它支持两类模式：  
> 1) **Basic Auth（`--credential`）**：由 `Authorization: Basic <base64>` 驱动，并且 WS 首包还会校验 `AuthToken`。  
> 2) **反向代理鉴权（`--auth-header`）**：由上游反代注入自定义 header；ttyd 只校验该 header 存在，并把其值作为 `TTYD_USER` 注入子进程环境。

事实源：`src/http.c:28-44`、`src/protocol.c:183-195`、`src/server.c:396-407, 581-587`、以及前端 `html/src/components/terminal/xterm/index.ts`。

---

## 1. 模式 A：Basic Auth（`--credential username:password`）

### 1.1 服务器端数据表示
事实源：`src/server.c:396-404`。

- CLI 输入：`username:password`（必须包含 `:`）
- 保存形式：`server->credential = base64(username:password)`（用 `lws_b64_encode_string` 编码后 `strdup`）

### 1.2 HTTP 鉴权规则
事实源：`src/http.c:34-41`。

当 `server->credential != NULL` 且未启用 `--auth-header` 时：

- 读取 HTTP header `Authorization`；
- 要求：
  - header 长度 `len >= 7`
  - header 包含子串 `"Basic "`（实现为 `strstr`，并不严格要求前缀）
  - `buf + 6`（即 `"Basic "` 后的 base64 内容）必须与 `server->credential` 完全相等
- 不通过时：
  - 返回 `401 Unauthorized`
  - 并添加 `WWW-Authenticate: Basic realm="ttyd"`
  - `Content-Length: 0`

### 1.3 WebSocket 鉴权规则（两段式）

#### 1.3.1 第 1 段：WS 连接过滤阶段的 Authorization 校验
事实源：`src/protocol.c:203-213` 与 `check_auth()`（`src/protocol.c:183-195`）。

当 `server->credential != NULL` 且未启用 `--auth-header` 时：
- 在 `FILTER_PROTOCOL_CONNECTION` 阶段读取 `Authorization`，规则与 HTTP 类似：  
  `n >= 7 && strstr(buf,"Basic ") && strcmp(buf+6, server->credential)==0`
- 不通过：直接 `return 1` 拒绝建立 WS（不会进入 ESTABLISHED）

#### 1.3.2 第 2 段：WS 首包 JSON 的 `AuthToken` 校验
事实源：`src/protocol.c:328-349`。

即使已经通过了 WS handshake 阶段的 Authorization 校验，服务端仍然要求：

- 客户端 WS 建连后发送首包 JSON
- JSON 必须包含：
  - `AuthToken`：必须等于 `server->credential`
  - `columns/rows`：用于初始窗口尺寸
- 若 `AuthToken` 不匹配：
  - `lws_close_reason(... POLICY_VIOLATION ...)`
  - callback 返回 `-1`

> 复刻实现注意：这是一种“冗余校验”，但属于现有行为的一部分。实现者必须保持两段式鉴权，否则会出现“前端能连上但无法启动进程”或“绕过 token 校验”等行为差异。

### 1.4 前端配合流程（必须复刻）
事实源：`html/src/components/terminal/index.tsx:25-29` 与 `html/src/components/terminal/xterm/index.ts:131-141, 260-266`。

1) `componentDidMount`：
   - 先 `refreshToken()`（HTTP GET `/token`）
   - 再 `open(container)`
   - 再 `connect()`（WS `/ws`）
2) WS `open`：
   - 发送 JSON：`{ AuthToken: this.token, columns: terminal.cols, rows: terminal.rows }`

---

## 2. 模式 B：反向代理鉴权（`--auth-header <Header-Name>`）

### 2.1 header 名归一化规则（必须复刻）
事实源：`src/server.c:581-587`。

当设置了 `server->auth_header`：
- 会在启动时：
  1) 在末尾追加 `:`（例如 `X-Remote-User` → `X-Remote-User:`）
  2) 全部转为小写（`lowercase()`）

原因（实现注释）：lws 自定义 header 需要 lower-case 且以 `:` 结尾。

### 2.2 HTTP 鉴权规则
事实源：`src/http.c:29-32`。

当 `server->auth_header != NULL` 时：
- HTTP 请求必须包含该自定义 header（检查 header length > 0）
- 缺失时返回：
  - `407 Proxy Auth Required`
  - `Proxy-Authenticate: Basic realm="ttyd"`（注意：虽然是反代鉴权，但仍返回 Basic realm 头，这是现有实现行为）
  - `Content-Length: 0`

### 2.3 WS 鉴权规则
事实源：`src/protocol.c:183-186`。

当 `server->auth_header != NULL` 时：
- WS filter 阶段调用 `lws_hdr_custom_copy` 将该 header 的值复制到 `pss->user`（`char user[30]`）
  - 复制成功条件：返回值 > 0
  - 复制失败（header 缺失）→ `check_auth=false` → 拒绝 WS
- `pss->user` 将在 spawn 子进程时被用来设置 `TTYD_USER=<pss->user>`（见 `src/protocol.c:137-143`）

重要说明：
- 在 auth-header 模式下，若 `--credential` 未启用，则 WS 首包 JSON 不需要包含有效 `AuthToken` 才能 spawn（因为服务端只在 `server->credential != NULL` 时执行 token 校验）。

---

## 3. 无鉴权模式

当既未设置 `--auth-header`，也未设置 `--credential`：

- HTTP 端点均无需鉴权；
- WS filter 阶段 `check_auth()` 直接返回 true；
- WS 首包 JSON 无需 token 校验。

---

## 4. 权限模型（Permission Model）

`ttyd` 不实现角色/权限（RBAC/ABAC）。其权限控制只有两个维度：

1) **是否允许建立连接**（auth-header 或 Basic 通过与否）  
2) **是否允许写入终端**（是否启用 `--writable`；未启用时 `INPUT` 被忽略）

