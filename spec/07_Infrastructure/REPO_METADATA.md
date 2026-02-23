# 仓库元信息与布局规格（Repo Metadata & Layout）

本文件定义：为了“只看规格即可复刻出功能等价仓库”，仓库在**文件结构、元信息、工程约定**层面的必需约束。

> 约束提醒：在提取规格时禁止读取仓库现存文档（`README*`、`docs/**`、`man/**` 与任何既有 `*.md`）。  
> 因此，本文件对 `README.md`、`man/` 中内容只做“存在性/用途”描述，不对其文字内容做逐字规格。

---

## 1) 根目录清单（必须存在）

### 1.1 构建与发布

- `CMakeLists.txt`
  - 作用：后端构建入口（依赖探测、编译宏、安装规则、版本号拼接）。
  - 规格归属：`spec/07_Infrastructure/BUILD.md`
- `cmake/`
  - 作用：CMake 辅助模块（获取 git commit/describe 生成版本后缀）。
  - 规格归属：`spec/07_Infrastructure/BUILD.md`
- `.github/workflows/`
  - 作用：CI（backend/frontend/docker/release）。
  - 规格归属：`spec/07_Infrastructure/CI.md`
- `scripts/`
  - 作用：交叉编译与 Windows（mingw/msys2）构建脚本。
  - 规格归属：`spec/07_Infrastructure/BUILD.md`
- `Dockerfile` / `Dockerfile.alpine`
  - 作用：容器运行镜像（Ubuntu/Alpine）。
  - 规格归属：`spec/07_Infrastructure/CONTAINERS.md`
- `snap/snapcraft.yaml`
  - 作用：snap 打包元数据（包含 license 声明）。
  - 规格归属：`spec/07_Infrastructure/PACKAGING.md`

### 1.2 运行时实现

- `src/`
  - 作用：后端（C）实现与前端内嵌资源生成物 `src/html.h`。
  - 规格归属：`spec/00_Overview/ARCHITECTURE.md`、`spec/02_Data/*`、`spec/03_API/*`、`spec/04_Business_Logic/*`
- `html/`
  - 作用：前端（Preact + xterm.js）实现与其构建管线（最终生成 `src/html.h`）。
  - 规格归属：`spec/07_Infrastructure/FRONTEND_PIPELINE.md` 与 `ui-ux-spec/**`

---

## 2) 文档/素材类文件（存在性要求，但不规格其文本内容）

> 这些文件不影响服务运行语义；复刻时可以保留同名文件以满足开源发布与展示需求。

- `README.md`
  - 用途：项目说明（本规格不读取、也不复刻其文字内容）。
- `man/`
  - 用途：man page（本规格不读取其内容；但构建系统会在安装阶段复制 `man/ttyd.1` 到系统目录，见 `spec/07_Infrastructure/BUILD.md`）。
- `screenshot.gif`
  - 用途：展示动图（不影响功能；不做内容级验收）。

---

## 3) 许可证（从“可执行配置”推导的约束）

尽管本规格提取阶段不读取 `LICENSE` 文本内容，但仓库配置中明确声明许可证为 MIT：

- `snap/snapcraft.yaml` 存在 `license: MIT`
- `html/package.json` 存在 `"license": "MIT"`

因此复刻仓库必须满足：

- `LICENSE` 文件存在，且语义上为 MIT License（建议使用 OSI 标准 MIT 文本）。

---

## 4) Git 属性与忽略规则（必须一致）

### 4.1 `.gitattributes`

该仓库将 `src/html.h` 标记为生成物，以降低语言统计/代码审查噪声：

- `src/html.h linguist-generated`

> 复刻仓库必须保留上述规则（语义一致即可）。

### 4.2 `.gitignore`

复刻仓库必须提供与本仓库语义一致的忽略规则，至少覆盖：

- 编译产物与中间文件：`*.o`、`*.a`、`*.so*`、`*.dylib`、`*.dll`、`*.exe` 等
- CMake 产物：`CMakeCache.txt`、`CMakeFiles/`、`cmake_install.cmake`、`install_manifest.txt`、`build/` 等
- 常见 IDE：`.idea/`、`.vscode/`

> 说明：`.gitignore` 的逐字一致性不作为功能验收的一部分，但必须保证“默认构建后 git status 干净”。

---

## 5) 代码风格（必须可复现）

### 5.1 C/C++ 格式化（`.clang-format`）

复刻仓库必须提供下述格式化约束（逐项一致）：

- `BasedOnStyle: Google`
- `Language: Cpp`
- `ColumnLimit: 120`
- `IndentWidth: 2`
- `TabWidth: 2`
- `UseTab: Never`

> 说明：后端主要是 C 文件，但 clang-format 以 Cpp 语言规则格式化（属既定工程选择）。

---

## 6) 与“生成物不要求字节一致”相关的仓库约定

该仓库包含一个显著生成物：`src/html.h`（由 `html/` 构建产物 inline + gzip 后生成）。

复刻要求：

- 不要求 `src/html.h` 字节级一致；
- 但要求 `src/html.h` 在**语义**上等价，能提供同等的前端页面与静态资源（详见 `spec/07_Infrastructure/FRONTEND_PIPELINE.md` 的“生成链路与验收口径”）。

