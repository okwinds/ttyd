# `ttyd` 复刻级工程规格文档索引（以源码为唯一事实源）

本目录 `spec/` 是“可复刻实现整个仓库”的工程规格文档集合。约束与目标：

- **事实源**：只以源码与可执行配置为事实源（例如 `src/**`、`html/src/**`、`CMakeLists.txt`、`scripts/**`、`.github/workflows/**`、`Dockerfile*`、`snap/**`）。  
- **禁止读取现存文档**：在提取与编写规格时，不读取仓库现存的 `README*`、`man/**`、`docs/**` 与任何既有 `*.md` 文档。  
- **复刻目标**：实现者在不查看原仓库源码的前提下，能仅凭此规格复刻出功能等价的仓库（**不要求**对某些生成物做字节级一致性复现；但会定义生成流程与校验口径）。

---

## 0. 总览（必读）
- [00_Overview/PROJECT.md](00_Overview/PROJECT.md) - 项目身份、范围、技术栈、版本策略、仓库构成
- [00_Overview/ARCHITECTURE.md](00_Overview/ARCHITECTURE.md) - 架构图、关键执行链路、时序图、关键状态机
- [00_Overview/GLOSSARY.md](00_Overview/GLOSSARY.md) - 术语表（协议命令、结构体、回调、PTY/ConPTY 等）

## 1. 配置（运行期与构建期）
- [01_Configuration/ENVIRONMENT.md](01_Configuration/ENVIRONMENT.md) - 环境变量（读/写）、默认值、作用域
- [01_Configuration/FEATURE_FLAGS.md](01_Configuration/FEATURE_FLAGS.md) - 功能开关（CLI 参数、编译特性、依赖能力检测）
- [01_Configuration/CLI_OPTIONS.md](01_Configuration/CLI_OPTIONS.md) - CLI 参数总表（默认值/校验/副作用）

## 2. 数据与结构（内存模型）
- [02_Data/ENTITIES.md](02_Data/ENTITIES.md) - 全部核心数据结构（字段、所有权、生命周期、不变量）
- [02_Data/RELATIONSHIPS.md](02_Data/RELATIONSHIPS.md) - 结构之间的引用关系与资源拥有关系
- [02_Data/MIGRATIONS.md](02_Data/MIGRATIONS.md) - 数据迁移（本项目无 DB；记录为“不适用”的原因与替代“版本演进”策略）

## 3. 对外接口（HTTP + WebSocket）
- [03_API/ENDPOINTS.md](03_API/ENDPOINTS.md) - HTTP 路由、WebSocket 端点、base-path 行为、响应与头部细节
- [03_API/AUTHENTICATION.md](03_API/AUTHENTICATION.md) - 鉴权机制（Basic / 反代 header / WS Token 握手）
- [03_API/ERRORS.md](03_API/ERRORS.md) - 错误、异常、关闭码、退出码、日志与可观测行为

## 4. 核心业务逻辑（“终端共享”行为）
- [04_Business_Logic/RULES.md](04_Business_Logic/RULES.md) - 关键规则（只读/可写、once/max-clients、origin 校验等）
- [04_Business_Logic/WORKFLOWS.md](04_Business_Logic/WORKFLOWS.md) - 关键流程（启动→服务→连接→握手→PTY→传输→关闭）
- [04_Business_Logic/STATE_MACHINES.md](04_Business_Logic/STATE_MACHINES.md) - 状态机（服务进程、连接、PTY、前端重连）

## 7. 基础设施（构建/发布/打包）
> 该目录由本规格补充生成，用于“复刻整个仓库”所需的构建与交付。

- [07_Infrastructure/BUILD.md](07_Infrastructure/BUILD.md) - CMake 构建、依赖探测、编译宏、Windows 资源文件
- [07_Infrastructure/REPO_METADATA.md](07_Infrastructure/REPO_METADATA.md) - 仓库布局、dotfiles、license、生成物定位（以可执行配置为准）
- [07_Infrastructure/FRONTEND_PIPELINE.md](07_Infrastructure/FRONTEND_PIPELINE.md) - `html/` 构建到 `src/html.h` 的生成链路（不要求字节一致，但要可重建）
- [07_Infrastructure/CI.md](07_Infrastructure/CI.md) - GitHub Actions：backend/frontend/docker/release 工作流逐条规格
- [07_Infrastructure/CONTAINERS.md](07_Infrastructure/CONTAINERS.md) - Docker 镜像（Ubuntu/Alpine）结构与入口命令
- [07_Infrastructure/PACKAGING.md](07_Infrastructure/PACKAGING.md) - Snapcraft、release 产物结构、校验文件（SHA256SUMS）

## 8. 测试规格（用于验证复刻正确性）
- [08_Testing/UNIT_SPECS.md](08_Testing/UNIT_SPECS.md) - 单元测试规格（纯函数/工具、协议解析、参数解析等）
- [08_Testing/INTEGRATION_SPECS.md](08_Testing/INTEGRATION_SPECS.md) - 集成测试规格（HTTP/WS/PTY 组合、跨平台差异）
- [08_Testing/E2E_SPECS.md](08_Testing/E2E_SPECS.md) - 端到端测试规格（浏览器连接、重连、文件传输、关闭行为）

## 9. 复刻验收与覆盖度
- [09_Verification/REPLICATION_GUIDE.md](09_Verification/REPLICATION_GUIDE.md) - “从规格实现仓库”的步骤与验收口径
- [09_Verification/COVERAGE_REPORT.md](09_Verification/COVERAGE_REPORT.md) - 覆盖度报告（代码元素→规格条目映射）
- [09_Verification/KNOWN_GAPS.md](09_Verification/KNOWN_GAPS.md) - 已知差异/限制（必须显式记录，避免暗坑）

## 附：扫描产物（不作为事实源，只作工作台）
- [./_discovery.md](_discovery.md) - 自动化扫描的发现记录（需人工复核）
- [./_inventory.md](_inventory.md) - 元素清单（已按本仓库修订）
