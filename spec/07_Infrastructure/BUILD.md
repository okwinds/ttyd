# 构建规格（CMake / 依赖 / 编译宏）

> 本文定义如何从源码构建 `ttyd` 后端二进制，以及构建系统必须具备的依赖与编译条件。  
> 事实源：`CMakeLists.txt`、`cmake/GetGitVersion.cmake`、`app.rc.in`、`src/server.c`（ConPTY 检查）。

---

## 1. 必需依赖（编译期）

`CMakeLists.txt` 对外依赖要求：

| 依赖 | 版本/要求 | CMake 检测方式 | 用途 |
|---|---|---|---|
| libwebsockets | `>= 3.2.0` | `find_package(Libwebsockets 3.2.0 REQUIRED)` | HTTP + WS server |
| libuv | required | `find_path(uv.h)` + `find_library(uv)` | event loop + pipe IO |
| json-c | required | `find_path(json.h)` + `find_library(json-c)` | JSON 解析 |
| zlib | required | `find_package(ZLIB REQUIRED)` | 解压内嵌 HTML gzip |
| OpenSSL | 条件依赖 | `find_package(OpenSSL REQUIRED)`（仅当 lws OpenSSL 且非 mbedTLS） | TLS 支持（由 lws 决定） |

### 1.1 强制要求：libwebsockets 必须启用 libuv 支持
事实源：`CMakeLists.txt:58-65`。

- CMake 会 `check_symbol_exists(LWS_WITH_LIBUV "lws_config.h" LWS_WITH_LIBUV)`
- 若未启用：`message(FATAL_ERROR "libwebsockets was not build with libuv support ...")`

换言之：复刻构建必须确保 libwebsockets 编译时带 `-DLWS_WITH_LIBUV=ON`。

---

## 2. 版本号策略（TTYD_VERSION）

事实源：`CMakeLists.txt:5-17`、`cmake/GetGitVersion.cmake`。

### 2.1 CMake 声明版本
`project(ttyd VERSION 1.7.7 LANGUAGES C)` 是基础版本。

### 2.2 从 Git tag/commit 派生版本

- `get_git_version(GIT_VERSION SEM_VER)`：
  - `git describe --tags --match "[0-9]*.[0-9]*.[0-9]*" --abbrev=8`
  - 若失败：`GIT_VERSION="0.0.0"`
  - 将输出中的 `-<n>-g<hash>` 规范化为 `-<hash>`（regex replace）
  - 再提取 `SEM_VER`（`x.y.z`）
- `get_git_head(GIT_COMMIT)`：
  - `git --git-dir <repo>/.git rev-parse --short HEAD`
  - 若失败：`GIT_COMMIT="unknown"`

最终 `TTYD_VERSION` 规则：
1) 默认等于 `${PROJECT_VERSION}`
2) 若 `${SEM_VER}` > `${PROJECT_VERSION}`，则覆盖
3) 若 `${GIT_COMMIT}` 非空，则追加 `-${GIT_COMMIT}`

### 2.3 编译宏注入
`target_compile_definitions` 会定义：
- `TTYD_VERSION="<computed>"`（`CMakeLists.txt:86-88`）
- Windows 平台额外定义 `_WIN32_WINNT=0xa00` 与 `WINVER=0xa00`（`CMakeLists.txt:88-89`）

---

## 3. 源码文件与链接

### 3.1 源文件列表
事实源：`CMakeLists.txt:30`。

编译源文件：
- `src/utils.c`
- `src/pty.c`
- `src/protocol.c`
- `src/http.c`
- `src/server.c`

### 3.2 Windows 特殊处理
事实源：`CMakeLists.txt:72-76`、`app.rc.in`。

- 链接库追加：`shell32`、`ws2_32`
- 通过 `configure_file(app.rc.in -> build/app.rc)` 生成资源文件并加入 `SOURCE_FILES`
- `app.rc.in` 中定义 Windows version resource（产品名、版本等）

### 3.3 Unix-like 特殊处理
事实源：`CMakeLists.txt:77-81`。

- 尝试 `find_library(util)`，若存在则链接 `util`（用于部分平台的 PTY 相关函数，如 `forkpty` 的实现可能在 util 中）。

---

## 4. 安装产物（Install）

事实源：`CMakeLists.txt:91-94`。

- 安装二进制：`install(TARGETS ttyd DESTINATION <CMAKE_INSTALL_BINDIR>)`
- 安装 man page：`install(FILES man/ttyd.1 DESTINATION <CMAKE_INSTALL_MANDIR>/man1)`

> 由于本规格提取阶段禁止读取 `man/**`，因此不会基于 man 内容生成 CLI 文档；CLI 规格已由源码解析生成（见 `spec/01_Configuration/CLI_OPTIONS.md`）。

---

## 5. 本地构建步骤（复刻实现建议）

### 5.1 典型 Unix-like（系统包依赖）

```bash
mkdir -p build
cd build
cmake ..
cmake --build .
```

前提：系统已安装并可被 CMake 找到的依赖（libwebsockets+libuv/json-c/zlib 等）。

### 5.2 Windows（MSYS2/MINGW）提示

仓库提供了脚本 `scripts/mingw-build.sh`（详见 `spec/07_Infrastructure/CI.md` 与 `spec/07_Infrastructure/PACKAGING.md`）。

运行期约束（必须复刻）：
- Windows 启动时会运行 `conpty_init()`，若 ConPTY API 不可用则直接打印错误并返回 `1`（`src/server.c:308-313`）。

