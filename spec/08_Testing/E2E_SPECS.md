# 端到端测试规格（E2E Specs）

> 端到端测试验证“真实浏览器 + 真实 WS + 真实 xterm UI”是否符合规格。  
> 推荐用 Playwright/Cypress；本规格不强制具体框架，但要求覆盖以下用例，并对 UI 微文案做逐字断言（pixel-clone 目标）。

---

## 0. 基线与测试环境（Baseline）

- 浏览器：Chromium（稳定版）
- 视口：`1280x720`（建议）
- 缩放：100%
- 运行方式：
  1) 启动 `ttyd` 后端（随机端口或固定端口）
  2) 浏览器访问 `http://localhost:<port>/`

---

## 1. 页面加载与基本布局

### 用例 1.1：首次访问渲染终端容器

断言：
- `document.body` 存在 `#terminal-container`
- 页面无滚动条（`html, body { overflow:hidden }`）
- `.terminal` 具有 padding `5px`，高度 `calc(100% - 10px)`（可通过 computed style 校验）

---

## 2. WS 建连与首包 hello

### 用例 2.1：WS 建立后标题设置

步骤：
1) 等待 WS 建立并收到 `SET_WINDOW_TITLE`

断言：
- `document.title` 被设置为服务端下发字符串（包含 `<command> (` 与 `)`）
- 若后续服务端发送 xterm title change 且未设置 `titleFixed`，`document.title` 变为：`<data> + " | " + <original title>`（见前端实现）

---

## 3. 断开与重连（微文案逐字）

### 用例 3.1：模拟 WS 非正常关闭触发自动重连

前置：确保 `disableReconnect` 未开启（默认）。

步骤：
1) 在测试中关闭 WS（模拟非 1000 close code，或 kill 后端使其断线）

断言（逐字）：
- overlay 先显示：`Connection Closed`
- 随后显示：`Reconnecting...`
- 重连成功后显示：`Reconnected`（300ms 后消失）

### 用例 3.2：正常关闭（1000）进入“按回车重连”模式

步骤：
1) 让服务端以 close code 1000 关闭（例如子进程 exit_code==0）

断言（逐字）：
- overlay 显示：`Press ⏎ to Reconnect`
- 按 Enter 后：
  - overlay 显示：`Reconnecting...`
  - 然后成功连上并可继续输出

---

## 4. beforeunload 离开提示（微文案逐字）

### 用例 4.1：WS OPEN 时触发 beforeunload

步骤：
1) 确保 WS 处于 OPEN
2) 触发 `beforeunload`（Playwright 可监听页面 dialog）

断言（逐字）：
- 提示文案为：`Close terminal? this will also terminate the command.`

### 用例 4.2：disableLeaveAlert=true 时不提示

步骤：
1) 通过 server prefs 或 URL query 设置 `disableLeaveAlert=true`

断言：
- 不再注册 beforeunload handler（或触发时不返回 message）

---

## 5. 复制行为（selection -> execCommand('copy') -> ✂ overlay）

### 用例 5.1：选择文本后出现剪刀 overlay

步骤：
1) 在终端里输出一段文本（例如 `echo hello`）
2) 用鼠标或键盘选择文本（使 `terminal.getSelection() != ''`）

断言：
- 尝试执行 `document.execCommand('copy')`
- overlay 显示字符：`✂`（U+2702）

---

## 6. 文件传输 UI（Modal）

### 用例 6.1：启用 Zmodem/Trzsz 时触发文件选择弹窗

前置：通过 server prefs 或 URL query 打开：
- `enableZmodem=true` 或 `enableTrzsz=true`

步骤：
1) 触发 `ZmodemAddon` 的 send flow（可通过模拟 session.detect send 触发 `onSend()`，或在终端中触发对应协议）
2) 应调用 `Terminal` 组件的 `showModal()`

断言（逐字）：
- Modal 内按钮文案为：`Choose files…`（注意：是单个 Unicode 省略号）

---

## 7. rendererType 切换（webgl/canvas/dom）

### 用例 7.1：默认 rendererType=webgl，WebGL 不可用时降级

前置：默认 `clientOptions.rendererType = 'webgl'`。

步骤：
1) 在不支持 WebGL 的环境或通过注入让 WebglAddon 抛错

断言：
- 控制台日志包含：`WebGL renderer could not be loaded, falling back to canvas renderer`
- 最终使用 Canvas 或 DOM renderer

---

## 8. `closeOnDisconnect` 行为

### 用例 8.1：closeOnDisconnect=true 时断开后关闭窗口

步骤：
1) 设置 `closeOnDisconnect=true`（会同时禁用 reconnect）
2) 触发 WS close

断言：
- 调用 `window.close()`

