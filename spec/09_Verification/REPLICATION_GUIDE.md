# 复刻实现指南（Replication Guide）

> 目标：一个不接触原仓库源码的实现者，仅凭 `spec/` 与 `ui-ux-spec/`，即可实现功能等价的 `ttyd` 仓库，并通过测试验收。  
> 本指南是“如何使用规格”的操作手册，不是源码解析文档。

---

## 0. 复刻的输出物（必须交付）

复刻仓库应至少包含：

1) 后端 C 源码（等价模块划分即可，但行为必须一致）：
   - HTTP handler
   - WS handler（协议一致）
   - PTY/ConPTY 子进程封装
   - CLI 参数解析与 lws/uv 初始化
2) 前端 Preact + xterm.js 工程：
   - UI 结构/微文案/交互一致
   - WS/HTTP 契约一致
3) 构建/发布链路：
   - CMake 构建
   - 前端构建到内嵌资源（生成 `src/html.h` 或等价机制）
   - GitHub Actions（backend/frontend/docker/release）
   - Dockerfile、snapcraft.yaml
4) 自动化验收（至少实现集成测试；推荐补齐 E2E）

---

## 1. 复刻顺序（强烈建议）

### Phase 1：对外接口优先（锁协议）

1) 先实现并锁定 **HTTP/WS 接口**（`spec/03_API/*`）：
   - `/`、`/token`、`/ws`
   - 鉴权（Basic/auth-header）
   - WS 命令字节与首包 JSON
2) 同步实现前端协议客户端（`ui-ux-spec/**` + `spec/03_API/ENDPOINTS.md`）：
   - URL 拼接规则
   - hello JSON
   - 重连文案与 overlay 行为

> 原因：ttyd 的核心价值在于“前后端协议闭环 + PTY 输出转发”。先锁协议能最大化避免返工。

### Phase 2：实现 PTY/ConPTY

按 `spec/02_Data/ENTITIES.md` 与 `spec/04_Business_Logic/*` 实现：
- Unix：`forkpty` + `uv_pipe` + `waitpid` thread + `uv_async`
- Windows：ConPTY + named pipe + CreateProcessW + wait callback + `uv_async`

### Phase 3：实现构建/内嵌资源与发布链路

按 `spec/07_Infrastructure/*`：
- CMake 依赖检测与编译宏
- 前端构建（webpack + gulp）并生成 `src/html.h`
- CI/Docker/Snap/Release 产物

---

## 2. 最小验收清单（必须全部通过）

### 2.1 后端行为验收

- [ ] `ttyd <command>` 能启动并监听端口（默认 7681，或 `--port 0` 随机端口）
- [ ] `GET /` 返回可工作的终端页面
- [ ] `GET /token` 返回 JSON，且 token 与 WS `AuthToken` 校验一致
- [ ] WS `/ws` 子协议 `tty`：
  - [ ] filter 阶段正确执行 once/max/auth/path/origin
  - [ ] hello JSON 触发 spawn
  - [ ] 初始消息按顺序发送（title → prefs）
  - [ ] OUTPUT/INPUT/RESIZE/PAUSE/RESUME 命令语义一致
- [ ] `--writable` 控制 INPUT 是否生效
- [ ] `--once`、`--exit-no-conn` 能在 client_count==0 时退出

### 2.2 前端 UI/UX 验收（pixel-clone）

- [ ] 全局布局、padding、高度计算与 `ui-ux-spec/01_Foundation/FOUNDATION.md` 一致
- [ ] 微文案逐字一致（见 `ui-ux-spec/02_Components/COMPONENTS.md` 与 `spec/03_API/ERRORS.md`）
- [ ] 重连状态机与 overlay 行为一致
- [ ] beforeunload 提示文案一致
- [ ] 复制剪刀 overlay 一致
- [ ] Modal 文件选择文案 `Choose files…` 一致

### 2.3 构建/交付验收

- [ ] CMake 能在目标平台编译并产出二进制
- [ ] `html/` 能构建并生成内嵌资源（`src/html.h` 或等价）
- [ ] GitHub Actions 工作流可运行并产生相同命名的 artifacts
- [ ] Docker 镜像运行默认命令 `ttyd -W bash`
- [ ] Release 产物命名与 SHA256SUMS 生成一致

---

## 3. 建议的自动化验证执行顺序

1) 先跑 **集成测试**（`spec/08_Testing/INTEGRATION_SPECS.md`）  
2) 再跑 **E2E**（`spec/08_Testing/E2E_SPECS.md`）  
3) 最后做 **CI/容器/发布链路 dry-run**（`spec/07_Infrastructure/*`）

---

## 4. 复刻实现的“不可偷懒点”（常见遗漏）

1) WS 的两段式鉴权（Authorization + AuthToken）在启用 `--credential` 时都存在  
2) base-path 同时影响 `/`、`/token`、`/ws` 与 `parent` redirect  
3) gzip/解压策略受编译宏与请求头影响  
4) UI 微文案必须逐字一致（包括 `⏎` 字符与省略号 `…`）  
5) PTY pause/resume 标志位不一致属于“现有行为”（详见 `spec/04_Business_Logic/RULES.md`）  

