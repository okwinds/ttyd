# 项目发现报告（自动扫描产物，仅作工作台）
root: <repo-root>
generated: 2026-02-23T06:59:18Z

> 说明：该文件来自自动扫描脚本输出，可能不准确。最终规格以 `spec/SPEC_INDEX.md` 中的文档为准。

## 1. 技术栈（人工复核后的结论）

### 后端
- 语言：C（C99）
- 框架/库：libwebsockets + libuv + json-c + zlib（可选 TLS）
- 入口：`src/server.c:main`

### 前端
- 语言：TypeScript + SCSS
- 框架：Preact
- 终端：xterm.js + addons（含 Zmodem/Trzsz/Sixel）
- 构建：webpack + gulp

## 2. 仓库类型

本仓库是“CLI + 内嵌前端 UI 的单二进制服务端应用”，不是纯前端/纯库。

## 3. 目录结构（仅列实现相关目录）

- `src/`：后端实现（HTTP/WS/PTY）
- `html/`：前端工程（构建后生成 `src/html.h`）
- `cmake/`：版本号辅助脚本
- `scripts/`：交叉编译与 Windows MINGW 构建脚本
- `.github/workflows/`：CI/CD
- `snap/`：snapcraft 打包

> 注：`docs/`、`man/`、`README*` 属于“现存文档”，提取规格阶段禁止读取，但构建系统可能会安装 man page（见 `spec/07_Infrastructure/BUILD.md`）。
