# 已知差异 / 未覆盖点（Known Gaps）

> 本文件用于“严肃、显式”记录：本规格相对源码的潜在模糊点、外部依赖点、以及当前实现存在的技术债/怪异行为。  
> 复刻实现若选择修复某些怪异行为，必须在此处记录并接受“行为不一致”的风险。

---

## 1. 有意不做字节级复刻的生成物

| 项目 | 说明 | 替代验收 |
|---|---|---|
| `src/html.h` | 由 `html/` 构建生成，包含 gzip bytes。规格不要求与原仓库字节级一致。 | 以“语义一致”验收：`GET /` 返回可运行 UI；gzip/解压分支都能工作（见 `spec/07_Infrastructure/FRONTEND_PIPELINE.md`）。 |

---

## 2. 三方依赖内容未内联到规格（但通过“锁版本 + 行为要求”覆盖）

| 依赖 | 原因 | 覆盖方式 |
|---|---|---|
| `@xterm/xterm/css/xterm.css` | CSS 内容在 node_modules，不属于仓库源码 | 规格要求使用 `html/package.json` 中的依赖版本并引入该 CSS；UI 行为按 `ui-ux-spec/**` 验收。 |
| libwebsockets/libuv/json-c/zlib | 依赖为外部库 | 规格定义最小版本与必须启用的编译特性（例如 `LWS_WITH_LIBUV`），并用接口行为验收。 |

---

## 3. 源码中的“怪异行为/技术债”（复刻应保持）

| 区域 | 行为/问题 | 影响 | 规格定位 |
|---|---|---|---|
| PTY pause/resume | `paused` 标志位不一致：spawn 设置 true，resume 不清 false，pause 因 guard 直接 return | 前端 PAUSE 可能不生效 | `spec/04_Business_Logic/RULES.md` 规则 12.3 |
| `--client-option -t` | case 分支内部 for-loop 复用 `optarg`，导致“无 -t 前缀的多项”不被正确解析 | 用户若错误用法可能困惑 | `spec/01_Configuration/CLI_OPTIONS.md` |
| endpoints 内存释放 | `--base-path` 会 `strdup` 新路径但进程退出前不 free | 轻微内存泄漏但进程短生命周期影响小 | `spec/02_Data/ENTITIES.md`（endpoints 所有权说明） |

---

## 4. 仍建议补齐的“实证性验证”（不是规格缺失，而是需要跑）

| 领域 | 为什么需要 | 推荐验证 |
|---|---|---|
| Windows ConPTY 分支 | 本地环境可能无法覆盖 | 在 Windows 10 1809+ 上跑集成/E2E |
| OpenSSL client cert verify 回调 | 依赖特定 lws 编译形态与证书环境 | 配置 `--ssl-ca` 并使用无效证书访问，验证错误日志与返回行为 |
| UNIX 域 socket | 依赖 lws 编译启用 unix sock | 用 `--interface /tmp/ttyd.sock` 启动并验证可连接 |

