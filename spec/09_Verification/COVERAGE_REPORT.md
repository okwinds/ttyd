# 规格覆盖度报告（Spec Coverage Report）

生成日期：2026-02-23

> 本报告用于回答两个问题：  
> 1) 仓库里有哪些“必须复刻的元素”？  
> 2) 每个元素是否在规格中被明确覆盖？

说明：
- “覆盖”以“能实现同等行为”为标准，不要求逐行复述源码；
- 由于提取阶段禁止读取现存 docs/man/README，覆盖度以源码与配置为基准。

---

## 1. 覆盖摘要

### 1.1 后端源码（C）

| 文件 | 是否覆盖 | 覆盖位置 |
|---|---:|---|
| `src/server.h` | ✅ | `spec/02_Data/ENTITIES.md`、`spec/03_API/ENDPOINTS.md`、`spec/04_Business_Logic/*` |
| `src/server.c` | ✅ | `spec/01_Configuration/CLI_OPTIONS.md`、`spec/07_Infrastructure/BUILD.md`、`spec/04_Business_Logic/WORKFLOWS.md` |
| `src/http.c` | ✅ | `spec/03_API/ENDPOINTS.md`、`spec/03_API/AUTHENTICATION.md`、`spec/03_API/ERRORS.md` |
| `src/protocol.c` | ✅ | `spec/03_API/ENDPOINTS.md`、`spec/04_Business_Logic/RULES.md`、`spec/04_Business_Logic/STATE_MACHINES.md` |
| `src/pty.h` / `src/pty.c` | ✅ | `spec/02_Data/ENTITIES.md`、`spec/04_Business_Logic/RULES.md`、`spec/08_Testing/UNIT_SPECS.md` |
| `src/utils.h` / `src/utils.c` | ✅ | `spec/08_Testing/UNIT_SPECS.md`（工具函数行为） |
| `src/html.h` | ✅（语义级） | `spec/07_Infrastructure/FRONTEND_PIPELINE.md`（不要求字节一致） |

### 1.2 前端源码（TypeScript/SCSS）

| 文件/目录 | 是否覆盖 | 覆盖位置 |
|---|---:|---|
| `html/src/index.tsx` | ✅ | `ui-ux-spec/04_Pages/PAGES.md`、`spec/00_Overview/PROJECT.md` |
| `html/src/components/app.tsx` | ✅ | `ui-ux-spec/01_Foundation/FOUNDATION.md`、`ui-ux-spec/07_Engineering_Constraints/ENGINEERING.md` |
| `html/src/components/terminal/**` | ✅ | `ui-ux-spec/02_Components/COMPONENTS.md`、`ui-ux-spec/03_Patterns/PATTERNS.md` |
| `html/src/components/modal/**` | ✅ | `ui-ux-spec/02_Components/COMPONENTS.md` |
| `html/src/style/index.scss` | ✅ | `ui-ux-spec/01_Foundation/FOUNDATION.md` |

> 注：`@xterm/xterm/css/xterm.css` 来自三方依赖，不在仓库中。规格以“锁定依赖版本 + 必须引入 CSS”方式覆盖（详见 `ui-ux-spec/07_Engineering_Constraints/ENGINEERING.md`）。

### 1.3 构建/CI/脚本/打包

| 元素 | 是否覆盖 | 覆盖位置 |
|---|---:|---|
| `CMakeLists.txt` | ✅ | `spec/07_Infrastructure/BUILD.md` |
| `cmake/GetGitVersion.cmake` | ✅ | `spec/07_Infrastructure/BUILD.md` |
| `scripts/cross-build.sh` | ✅ | `spec/01_Configuration/ENVIRONMENT.md`、`spec/07_Infrastructure/CI.md` |
| `scripts/mingw-build.sh` | ✅ | `spec/07_Infrastructure/BUILD.md`、`spec/07_Infrastructure/CI.md` |
| `.github/workflows/*` | ✅ | `spec/07_Infrastructure/CI.md` |
| `Dockerfile*` | ✅ | `spec/07_Infrastructure/CONTAINERS.md` |
| `snap/snapcraft.yaml` | ✅ | `spec/07_Infrastructure/PACKAGING.md` |
| `app.rc.in` | ✅ | `spec/07_Infrastructure/BUILD.md` |

---

## 2. 已覆盖但建议进一步自动化验证（Unverified）

| 领域 | 原因 | 推荐动作 |
|---|---|---|
| PTY/ConPTY 跨平台行为 | 需要真实 OS/终端环境 | 按 `spec/08_Testing/INTEGRATION_SPECS.md` 分平台执行 |
| gzip/解压分支 | 受编译宏与请求头影响 | 对两种请求头与两种编译形态做组合测试 |
| Zmodem/Trzsz/Sixel | 依赖终端侧协议与浏览器行为 | 用 Playwright 执行 E2E 与拖拽/文件选择交互 |

---

## 3. 已知“行为细节/技术债”记录位置

- PTY pause/resume 标志位不一致：`spec/04_Business_Logic/RULES.md`（规则 12.3）  
- `--client-option -t` 的多项解析细节：`spec/01_Configuration/CLI_OPTIONS.md`  

