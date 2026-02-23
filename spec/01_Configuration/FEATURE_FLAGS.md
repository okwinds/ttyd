# 功能开关与编译特性（Feature Flags）

> `ttyd` 的“开关”分为三类：  
> 1) **运行时开关**：CLI 参数（详见 `spec/01_Configuration/CLI_OPTIONS.md`）  
> 2) **编译特性开关**：由依赖库/平台宏决定是否编译某段代码/某个参数可用  
> 3) **前端偏好设置开关**：服务端下发的 `SET_PREFERENCES` + URL query override（见 `spec/03_API/ENDPOINTS.md` 与 `ui-ux-spec/**`）

---

## 1. 编译特性（后端 C）

### 1.1 平台宏

| 宏 | 语义 | 影响 |
|---|---|---|
| `_WIN32` | Windows 平台分支 | 启用 ConPTY 初始化检查（`src/server.c:308-313`），使用 ConPTY spawn 实现（`src/pty.c` Windows 分支），并默认下发前端偏好 `isWindows=true`（`src/server.c:343-345`）。 |

### 1.2 libwebsockets 版本与能力宏

| 条件 | 语义 | 影响 |
|---|---|---|
| `LWS_LIBRARY_VERSION_NUMBER >= 4000000` | lws v4+ | 启用 backoff/retry policy（`src/server.c:41-51`），支持 CLI `--ping-interval`（`src/server.c:65-67`、`src/server.c:458-467`）。 |
| `LWS_WITHOUT_EXTENSIONS` 未定义 | 允许 WS 扩展 | 注册 `permessage-deflate` 与 `deflate-frame` 扩展（`src/server.c:33-39`）。 |
| `LWS_ROLE_H2` 定义 | 支持 HTTP/2 role | WS 过滤中当 GET_URI 取不到时用 `WSI_TOKEN_HTTP_COLON_PATH`（`src/protocol.c:215-217`）。 |

### 1.3 TLS/SSL 能力宏

| 条件 | 语义 | 影响 |
|---|---|---|
| `LWS_OPENSSL_SUPPORT` 或 `LWS_WITH_TLS` | lws 支持 TLS | CLI 启用 `-S/-C/-K/-A` 分支（`src/server.c:480-496`、`src/server.c:563-576`）。 |
| `LWS_WITH_MBEDTLS` | lws 使用 mbedTLS | 若未定义且 OpenSSL 支持，CMake 会额外 `find_package(OpenSSL)` 并链接（`CMakeLists.txt:66-70`）。 |
| `LWS_CALLBACK_OPENSSL_PERFORM_CLIENT_CERT_VERIFICATION` 相关 | OpenSSL client cert verify 回调 | `http.c` 在该回调中输出 verify 错误并返回 1（`src/http.c:218-227`）。 |

### 1.4 IPv6 能力宏

| 条件 | 语义 | 影响 |
|---|---|---|
| `LWS_WITH_IPV6` | lws 编译启用 IPv6 | help 文案中出现 `-6, --ipv6`（`src/server.c:121-123`），并允许移除 `DISABLE_IPV6`（`src/server.c:477-479`）。 |

### 1.5 UNIX 域 socket 能力宏

| 条件 | 语义 | 影响 |
|---|---|---|
| `LWS_USE_UNIX_SOCK` 或 `LWS_WITH_UNIX_SOCK` | lws 支持 UNIX domain socket | 当 `--interface` 以 `.sock/.socket` 结尾时启用 `LWS_SERVER_OPTION_UNIX_SOCK`；否则报错并返回 `-1`（`src/server.c:548-559`）。 |

---

## 2. 前端偏好设置（Preferences）开关

> 前端偏好设置最终由三份来源合并（顺序为后者覆盖前者）：  
> 1) `App` 中的默认 `clientOptions`（`html/src/components/app.tsx:12-22`）  
> 2) 服务端通过 `SET_PREFERENCES` 下发的 JSON（`src/protocol.c:28-30`）  
> 3) URL query override（`html/src/components/terminal/xterm/index.ts:309-339`）

这些开关会影响：
- 渲染器类型：`rendererType`（dom/canvas/webgl）
- 重连：`disableReconnect`、`closeOnDisconnect`
- UI 行为：`disableLeaveAlert`、`disableResizeOverlay`
- 功能：`enableZmodem`、`enableTrzsz`、`enableSixel`
- 其他：`titleFixed`、`unicodeVersion`、`trzszDragInitTimeout`

具体前端行为见 `ui-ux-spec/02_Components/COMPONENTS.md` 与 `ui-ux-spec/03_Patterns/PATTERNS.md`。

