# 环境变量规格（运行期 + 构建/CI）

> 本文覆盖三类环境变量：  
> 1) **运行期（ttyd 二进制本身读取/写入）**  
> 2) **子进程环境（ttyd 设置给被执行命令）**  
> 3) **构建/CI/脚本环境（用于复刻仓库的构建与发布链路）**

---

## 1. 运行期（ttyd 进程读取的环境变量）

| 变量名 | 类型 | 是否必需 | 默认 | 读取位置 | 语义 |
|---|---|---:|---|---|---|
| `HOME` | string | 否 | OS 决定 | `src/server.c:428-435` | 仅在 `--index` 的参数以 `~/` 开头时，用于将其展开为绝对路径：`<HOME> + <optarg+1>`。若 `HOME` 不存在，将导致 `strlen(home)` 触发未定义行为；复刻实现可选择更稳健，但需保持现有行为（建议：在规格实现中显式要求 `HOME` 必须存在，或在缺失时按原实现“崩溃/错误”处理）。 |

## 2. 运行期（ttyd 写入/注入给子进程的环境变量）

事实源：`src/protocol.c:128-148`（`build_env()`），以及 `src/pty.c` 的 Unix/Windows 分支对 envp 的处理。

| 变量名 | 类型 | 是否必需 | 默认 | 设置时机 | 语义 |
|---|---|---:|---|---|---|
| `TERM` | string | 是 | `xterm-256color`（可由 `--terminal-type` 覆盖） | 每次 spawn 子进程前构造 envp | 子进程的终端类型报告。 |
| `TTYD_USER` | string | 否 | unset | 当 `--auth-header` 模式从 WS header 取到用户标识时设置 | 将“反向代理鉴权的用户标识”传给子进程。仅在 `pss->user` 非空时设置。 |

重要补充（必须复刻）：
- ttyd **不会**清空子进程的原始环境，仅追加/覆盖 `TERM` 与可选 `TTYD_USER`：  
  - Unix：子进程中循环 `putenv(*p)`（`src/pty.c:433-436`），其余 `environ` 保留  
  - Windows：`_wputenv` 写入当前进程环境后再 `CreateProcessW`（`src/pty.c:329-337`）

---

## 3. 前端构建期环境变量（webpack/开发模式）

事实源：`html/src/index.tsx` 与 `html/webpack.config.js`。

| 变量名 | 类型 | 是否必需 | 默认 | 语义 |
|---|---|---:|---|---|
| `NODE_ENV` | string | 是（构建脚本语义上） | 未设置则视为非 production（`devMode = NODE_ENV !== 'production'`） | 决定 webpack dev/prod 配置与输出文件名是否带 hash；并决定是否在 `html/src/index.tsx` 中 `require('preact/debug')`。 |

---

## 4. 构建/交付脚本环境变量（用于复刻仓库）

### 4.1 `scripts/cross-build.sh`

事实源：`scripts/cross-build.sh:8-18`。

| 变量名 | 默认 | 语义 |
|---|---|---|
| `CROSS_ROOT` | `/opt/cross` | 存放 musl 交叉工具链的目录，并追加到 `PATH`。 |
| `STAGE_ROOT` | `/opt/stage` | 每个 target 的依赖安装前缀根目录。 |
| `BUILD_ROOT` | `/opt/build` | 每个 target 的源码解压/编译根目录。 |
| `BUILD_TARGET` | `x86_64` | 构建目标别名/架构选择（可为 `amd64/arm64/armv7/...`，脚本会做映射）。 |
| `ZLIB_VERSION` | `1.3.2` | zlib 版本。 |
| `JSON_C_VERSION` | `0.17` | json-c 版本。 |
| `MBEDTLS_VERSION` | `2.28.5` | mbedTLS 版本。 |
| `LIBUV_VERSION` | `1.44.2` | libuv 版本。 |
| `LIBWEBSOCKETS_VERSION` | `4.3.6` | libwebsockets 版本。 |

### 4.2 GitHub Actions / Docker / Release 环境变量与 secrets

事实源：`.github/workflows/*.yml` 与 `Dockerfile*`。

| 名称 | 类型 | 作用域 | 语义 |
|---|---|---|---|
| `GITHUB_REF` | env | GitHub Actions | 用于区分 tag push 与 main push，并决定 docker tag（见 `.github/workflows/docker.yml:41-51`）。 |
| `DOCKER_HUB_USER` | secret | GitHub Actions | docker hub 登录用户名（`.github/workflows/docker.yml:30-33`）。 |
| `DOCKER_HUB_TOKEN` | secret | GitHub Actions | docker hub 登录 token（同上）。 |
| `GITHUB_TOKEN` | secret | GitHub Actions | 登录 ghcr（`.github/workflows/docker.yml:34-38`）。 |

> 复刻实现提示：上述 secrets 不属于代码库逻辑，而是 CI 运行时依赖。复刻仓库时应提供 `.env.example` 或 CI 配置说明（可写入 `spec/07_Infrastructure/CI.md`），但本规格不会在正文中包含任何真实密钥值。

