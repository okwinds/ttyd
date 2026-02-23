# 集成测试规格（Integration Specs）

> 集成测试验证“后端 HTTP/WS/PTY + 前端协议客户端”的组合行为是否符合规格。  
> 推荐实现方式：用任意语言（Go/Python/Node）实现测试客户端；或直接复用浏览器 E2E（见 `spec/08_Testing/E2E_SPECS.md`）。

---

## 0. 通用前置条件

### 0.1 启动命令
选择一个最简单且可预测的命令作为 `<command>`，例如：

- Unix-like：`bash` 或 `sh`
- Windows：`cmd.exe` 或 `powershell`

### 0.2 端口选择
建议用 `--port 0` 让系统随机分配端口，并通过日志或接口获取实际端口（`lws_get_vhost_listen_port` 在日志中输出）。

---

## 1. HTTP：`GET /token`（无鉴权）

### 用例 1.1：未启用 `--credential`

步骤：
1) 启动：`ttyd <command>`
2) 请求：`GET /token`

期望：
- 200
- `Content-Type` 为 `application/json;charset=utf-8`
- body 为 `{"token": ""}`（token 值为空串）

---

## 2. HTTP：`GET /`（内嵌 UI）

### 用例 2.1：客户端支持 gzip（Accept-Encoding: gzip）

前置：后端未定义 `LWS_WITH_HTTP_STREAM_COMPRESSION`（取决于编译）；若定义则跳到用例 2.2。

步骤：
1) `GET /` 携带 `Accept-Encoding: gzip`

期望：
- 200
- `Content-Type: text/html`
- `Content-Encoding: gzip`
- body 为 gzip bytes（首 bytes 为 `1f 8b 08`）

### 用例 2.2：客户端不支持 gzip（无 Accept-Encoding 或不含 gzip）

步骤：
1) `GET /` 不带 gzip accept header

期望：
- 200
- `Content-Type: text/html`
- 不带 `Content-Encoding: gzip`
- body 为可解析 HTML（以 `<!DOCTYPE html>` 开头，或至少包含 `<html`）

---

## 3. 鉴权：Basic Auth（HTTP）

### 用例 3.1：缺失 Authorization

步骤：
1) 启动：`ttyd -c user:pass <command>`
2) 请求：`GET /`

期望：
- 401
- `WWW-Authenticate: Basic realm="ttyd"`
- `Content-Length: 0`

### 用例 3.2：Authorization 错误

步骤：
1) 用错误 base64 发送 `Authorization: Basic ...`

期望：同 401

### 用例 3.3：Authorization 正确

步骤：
1) 发送 `Authorization: Basic <base64(user:pass)>`

期望：
- 能正常返回 `/` 与 `/token`（/token 返回 token 即 base64）

---

## 4. 鉴权：auth-header（HTTP）

### 用例 4.1：缺失自定义 header

步骤：
1) 启动：`ttyd -H X-Remote-User <command>`
2) 请求：`GET /`

期望：
- 407
- `Proxy-Authenticate: Basic realm="ttyd"`
- `Content-Length: 0`

### 用例 4.2：存在自定义 header

步骤：
1) 请求添加 header：`X-Remote-User: alice`

期望：
- 200 正常

---

## 5. WebSocket：首包 hello + spawn + output

### 用例 5.1：无 credential 的 WS hello

步骤：
1) 启动：`ttyd <command>`
2) 建立 WS 到 `/ws`，子协议 `tty`
3) 发送首包 JSON：`{"AuthToken":"","columns":80,"rows":24}`
4) 读取服务端消息：
   - 应先收到 `SET_WINDOW_TITLE`（命令字节 `'1'`）
   - 再收到 `SET_PREFERENCES`（命令字节 `'2'`）
   - 之后收到 `OUTPUT`（命令字节 `'0'`）若子进程有输出（例如 shell prompt）

期望：
- 服务端不会因 token 失败关闭连接

### 用例 5.2：启用 credential 时 token 不匹配

步骤：
1) 启动：`ttyd -c user:pass <command>`
2) WS 建连（同时也需在 WS handshake header 带正确 Authorization，否则 filter 会拒绝）
3) 发送首包 JSON：`{"AuthToken":"WRONG","columns":80,"rows":24}`

期望：
- 服务端以 POLICY_VIOLATION 关闭（close_reason）
- 客户端无法继续发送 INPUT（连接已关闭）

### 用例 5.3：启用 credential 时 token 匹配

步骤：
1) `GET /token` 获取 token（应为 base64）
2) WS 建连并发送 hello JSON 中 AuthToken=该 token

期望：
- 连接保持
- 初始消息按顺序下发
- 可以收到 OUTPUT

---

## 6. WebSocket：输入、resize、只读/可写

### 用例 6.1：只读模式忽略 INPUT

前置：不传 `-W`

步骤：
1) 建连并 spawn
2) 发送 INPUT：`'0' + "echo hi\\n"`

期望：
- 服务端不写入 PTY（终端不会出现 `hi` 输出）

### 用例 6.2：可写模式生效

前置：`ttyd -W <command>`

步骤：
1) 发送 INPUT：输入能被 shell 执行并产生输出

### 用例 6.3：resize 生效

步骤：
1) 发送 RESIZE：命令字节 `'1'` + JSON `{"columns":120,"rows":40}`

期望：
- 服务端调用 `pty_resize`；某些 shell/应用会调整布局（可通过运行 `stty size` 验证）

---

## 7. `--url-arg`：URL 参数注入

步骤：
1) 启动：`ttyd -a -W <command>`
2) 浏览器/客户端以 URL `/?arg=foo&arg=bar`（或 lws token fragments 等价）建立 WS

期望：
- spawn 时 argv 应为 `<command> ... foo bar`（追加在 server CLI argv 之后）

> 注意：实现未 URL decode，且只解析以 `arg=` 开头的 query fragment。

---

## 8. `--once` / `--exit-no-conn`：退出策略

### 用例 8.1：once

步骤：
1) 启动：`ttyd --once <command>`
2) 建立一个 WS 并断开

期望：
- `ttyd` 进程退出（exit 0）

### 用例 8.2：max-clients

步骤：
1) 启动：`ttyd --max-clients 1 <command>`
2) 并行建立两个 WS

期望：
- 第一个成功
- 第二个在 filter 阶段被拒绝（连接建立失败）

