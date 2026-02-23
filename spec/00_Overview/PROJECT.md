# `ttyd` 项目总览（复刻级）

## 1. 项目身份（Identity）

- **项目名**：`ttyd`
- **项目类型**：单二进制 CLI 程序 + 内嵌单页 Web UI（HTTP + WebSocket）
- **核心用途**：在服务端启动一个带 PTY/ConPTY 的子进程（例如 `bash`），将其标准输入/输出通过 WebSocket 流式转发到浏览器的 xterm.js 终端；浏览器向服务端发送键盘输入与窗口大小变更，从而实现“浏览器里的真实终端”。
- **主入口**：
  - 后端入口：`src/server.c:main(int argc, char **argv)`
  - 前端入口：`html/src/index.tsx`（构建后被“内嵌”为 `src/html.h`，由后端 HTTP handler 返回）

## 2. 本规格的范围与约束（Scope & Constraints）

### 2.1 事实源（Source of Truth）
本规格内容仅来自以下类型文件：
- 后端：`src/*.c`、`src/*.h`
- 前端：`html/src/**`、`html/webpack.config.js`、`html/gulpfile.js`、`html/package.json`、`html/yarn.lock`
- 构建与发布：`CMakeLists.txt`、`cmake/**`、`scripts/**`、`.github/workflows/**`、`Dockerfile*`、`snap/**`、`app.rc.in`

### 2.2 禁止读取现存文档（Hard Rule）
为满足你的约束：提取与编写过程中不读取仓库现存的 `README*`、`man/**`、`docs/**` 与任何既有 `*.md` 文档。

### 2.3 复刻目标（Replication Target）
- **功能等价复刻**：实现者仅凭本规格即可复刻出同等功能的 `ttyd` 仓库（后端 + 内嵌前端 UI + 构建/CI/容器/打包）。
- **不要求字节级生成物一致**：例如 `src/html.h` 的字节序列不要求与原仓库完全一致；但规格会定义“可重建”的生成链路与行为一致性口径。

## 3. 技术栈（Tech Stack）

### 3.1 后端（C）
- 语言：C99（附带 `_GNU_SOURCE`）见 `CMakeLists.txt:19-28`
- 网络：libwebsockets（HTTP + WebSocket server）
  - 协议注册：`src/server.c:28-31`（`protocols[]`：`http-only` 与 `tty`）
  - HTTP handler：`src/http.c:callback_http`
  - WS handler：`src/protocol.c:callback_tty`
- 事件循环：libuv（`struct server.loop`，见 `src/server.h:85`）
- JSON：json-c（解析窗口尺寸等，见 `src/protocol.c:38-48`）
- 压缩：zlib（内嵌 HTML gzip 的解压缓存，见 `src/http.c:52-80`）
- TLS：由 libwebsockets 编译特性决定；当启用 OpenSSL 且非 mbedTLS 时由 CMake 链接 OpenSSL（见 `CMakeLists.txt:61-70`）
- PTY：
  - Unix-like：`forkpty`（见 `src/pty.c:419-484`）
  - Windows：ConPTY（见 `src/pty.c:167-371`）

### 3.2 前端（TypeScript + Preact）
- 框架：Preact（入口 `html/src/index.tsx`）
- 终端：xterm.js（`@xterm/xterm`）+ addon（Fit/WebLinks/WebGL/Canvas/Clipboard/Unicode11/Image）
- 文件传输：Zmodem（`zmodem.js`）与 trzsz（`trzsz`）
- 构建：webpack（打包）+ gulp（内联资源、gzip、生成 `src/html.h`）
- 包管理：Yarn 3（`packageManager: yarn@3.6.3`）

## 4. 运行时平台与关键依赖（Runtime Platforms）

### 4.1 操作系统
- Windows：启动时调用 `conpty_init()`，失败则报错退出（见 `src/server.c:308-313`）
- Linux/macOS/BSD：依赖 PTY API 与 `ioctl(TIOCSWINSZ)` 等

### 4.2 监听形态
- TCP：默认端口 `7681`；`--port 0` 代表随机端口（见 `src/server.c:320-387`）
- UNIX 域 socket：当 `--interface` 以 `.sock` / `.socket` 结尾且 libwebsockets 支持时启用（见 `src/server.c:546-561`）

## 5. 构建时版本策略（Build-time Versioning）

构建版本 `TTYD_VERSION` 计算规则：

1) 基础版本来自 `project(ttyd VERSION 1.7.7 LANGUAGES C)`（见 `CMakeLists.txt:5`）  
2) 若 Git tag 的 semver 前缀 `x.y.z` 大于基础版本，则覆盖之（见 `CMakeLists.txt:9-14` 与 `cmake/GetGitVersion.cmake:21`）  
3) 若能取到短 commit hash，则追加 `-<commit>`（见 `CMakeLists.txt:15-17` 与 `cmake/GetGitVersion.cmake:29-46`）  
4) 通过 `target_compile_definitions` 编译进二进制（见 `CMakeLists.txt:86-89`），并被 `--version` / 日志输出使用（见 `src/server.c:355-356, 578-579`）

## 6. “复刻成功”的最低验收（MVP）

一个复刻实现至少满足：

- `ttyd -W bash` 能启动，浏览器访问 `/` 能看到终端；
- `GET /token` 返回 JSON `{"token":"<base64(username:password)>"}`
  - 若未启用 `--credential`，则 token 为 `""`（空字符串）
- WS：
  - 端点为 `/ws`（或被 `--base-path` 前缀化）
  - 子协议为 `tty`
  - 建连后客户端发送首包 JSON（包含 `columns/rows`，若启用 credential 则还需 `AuthToken`）
  - 输入/输出/resize 与本规格定义的“1 字节命令码 + payload”一致
