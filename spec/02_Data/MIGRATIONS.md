# “迁移”（Migrations）说明：不适用（N/A）

`ttyd` 是一个无持久化存储的单二进制程序：

- 不包含数据库；
- 不包含 schema/migration 文件；
- 运行期状态全部存在于内存结构体中（见 `spec/02_Data/ENTITIES.md`）。

因此传统意义上的“迁移历史/回滚流程”在本仓库中**不适用**。

## 版本演进的替代关注点（复刻实现应关注）

虽然没有 DB migrations，但仍有三类“版本演进点”会影响复刻实现：

1) **构建版本号计算**（`TTYD_VERSION`）：见 `spec/00_Overview/PROJECT.md` 与 `spec/07_Infrastructure/BUILD.md`  
2) **前端到 `src/html.h` 的生成链路**：见 `spec/07_Infrastructure/FRONTEND_PIPELINE.md`  
3) **对外协议兼容性**（HTTP/WS）：见 `spec/03_API/*` 与 `ui-ux-spec/**`

