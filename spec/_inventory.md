# 元素清单（Inventory，已人工修订）
root: <repo-root>
generated: 2026-02-23（人工修订于同日）

> 目的：列出仓库中所有“需要被规格覆盖”的实现与配置元素。  
> 注意：本清单不是规格本身；规格正文在 `spec/SPEC_INDEX.md`。

---

## 1) 后端实现（C）

- `src/server.h`：协议命令码、全局结构体（server/endpoints/pss）
- `src/server.c`：CLI 解析、lws context/vhost 创建、uv loop、信号处理
- `src/http.c`：HTTP 路由（/ /token /parent）、鉴权、gzip/解压、缓存
- `src/protocol.c`：WS 路由（/ws）、握手过滤、首包 JSON spawn、二进制帧协议、关闭策略
- `src/pty.h` / `src/pty.c`：PTY/ConPTY 子进程封装、pipe/async/thread、resize/kill、pause/resume
- `src/utils.h` / `src/utils.c`：辅助函数（malloc、signal、open_uri、Windows quoting）
- `src/html.h`：前端内嵌资源生成物（规格覆盖其语义与生成链路，不要求字节一致）

## 2) 前端实现（Preact + xterm.js）

- `html/src/index.tsx`：入口渲染、development debug 注入
- `html/src/components/app.tsx`：URL 拼接、默认 options（主题/字体/流控）
- `html/src/components/terminal/index.tsx`：容器、生命周期、Modal 触发与 sendFile
- `html/src/components/terminal/xterm/index.ts`：WS 协议实现、重连、preferences 应用、渲染器切换、微文案
- `html/src/components/terminal/xterm/addons/overlay.ts`：overlay 视觉与行为
- `html/src/components/terminal/xterm/addons/zmodem.ts`：Zmodem/Trzsz 文件收发与进度输出
- `html/src/components/modal/index.tsx` + `modal.scss`：Modal 结构与样式（含“Choose files…”文案）
- `html/src/style/index.scss`：全局布局与终端容器样式
- `html/src/template.html`：HtmlWebpackPlugin 模板（inline 标记）

## 3) 前端构建配置

- `html/package.json`：依赖版本（xterm/preact 等）、scripts（build/start/check）
- `html/yarn.lock`：依赖锁定
- `html/webpack.config.js`：webpack + devServer proxy
- `html/gulpfile.js`：inline+gzip+生成 `src/html.h`
- `html/tsconfig.json`：TS 编译配置（复刻时需一致）

## 4) 后端构建/版本脚本

- `CMakeLists.txt`：依赖探测、编译宏、安装规则
- `cmake/GetGitVersion.cmake`：版本号计算（git describe / commit）
- `app.rc.in`：Windows 资源文件模板

## 5) 交付/打包/发布

- `.github/workflows/backend.yml`：cross build matrix（产物）
- `.github/workflows/frontend.yml`：yarn check + build
- `.github/workflows/docker.yml`：multi-arch 镜像 build/push
- `.github/workflows/release.yml`：release 产物整理 + SHA256SUMS
- `Dockerfile` / `Dockerfile.alpine`：运行镜像
- `snap/snapcraft.yaml`：snap 打包
- `scripts/cross-build.sh`：交叉编译依赖 + 静态构建
- `scripts/mingw-build.sh`：MSYS2/MINGW 构建（安装 libwebsockets）

## 6) 仓库元信息（dotfiles/许可证/文档类资产）

> 说明：提取阶段禁止读取现存文档，因此仅将其作为“复刻时需要考虑的元素”列入清单。

- `.clang-format`：C/C++ 格式化规则（BasedOnStyle=Google，ColumnLimit=120 等）
- `.gitattributes`：将 `src/html.h` 标注为 `linguist-generated`
- `.gitignore`：构建产物/IDE/CMake 输出忽略规则
- `LICENSE`：许可证文件（从 `snap/snapcraft.yaml` 与 `html/package.json` 可推导为 MIT）
- `README.md`：项目说明（禁止读取其内容）
- `man/ttyd.1`：man page（禁止读取其内容；但 CMake 安装规则会安装该文件）
- `screenshot.gif`：展示动图（非功能性资产）
