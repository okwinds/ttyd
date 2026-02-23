# CI 规格（GitHub Actions）

> 本文逐条定义 `.github/workflows/**` 的行为，用于复刻仓库的 CI/CD。  
> 事实源：`.github/workflows/backend.yml`、`frontend.yml`、`docker.yml`、`release.yml`、以及 `scripts/cross-build.sh`。

---

## 1. `backend.yml`：跨平台静态构建（artifact）

事实源：`.github/workflows/backend.yml`。

### 1.1 触发条件
- push / pull_request 且变更路径匹配：
  - `.github/workflows/backend.yml`
  - `CMakeLists.txt`
  - `src/*`
  - `scripts/*`
- 也支持 `workflow_call`（可被 release workflow 调用）

### 1.2 job：`cross`

- runner：`ubuntu-22.04`
- matrix targets：
  - `i686, x86_64, arm, armhf, aarch64, mips, mipsel, mips64, mips64el, s390x, win32`
- steps：
  1) checkout
  2) apt 安装：`autoconf automake build-essential cmake curl file libtool`
  3) `env BUILD_TARGET=${{ matrix.target }} ./scripts/cross-build.sh`
  4) 上传 artifact：
     - name：`ttyd.<target>`
     - path：`build/ttyd*`

> 复刻实现必须确保 `scripts/cross-build.sh` 在 ubuntu 上可运行，并输出 `build/ttyd`（或 `build/ttyd.exe`）。

---

## 2. `frontend.yml`：前端 lint + build

事实源：`.github/workflows/frontend.yml`。

### 2.1 触发条件
- push / pull_request 且变更路径匹配：
  - `.github/workflows/frontend.yml`
  - `html/*`（注意：只匹配一层；但实际变更通常在 `html/src/**`，仍会触发？复刻仓库可考虑更合理的 glob，但为行为一致应保持现状）

### 2.2 job：`build`

- runner：`ubuntu-22.04`
- steps：
  1) checkout
  2) setup-node：node 18
  3) 在 `html` 目录执行：
     - `corepack enable`
     - `corepack prepare yarn@stable --activate`
     - `yarn install`
     - `yarn run check`（gts check）
     - `yarn run build`（webpack prod + gulp 生成 `src/html.h`）

---

## 3. `docker.yml`：多架构镜像构建与推送

事实源：`.github/workflows/docker.yml`、`Dockerfile`、`Dockerfile.alpine`。

### 3.1 触发条件
- push 到 `main`
- push tag（`tags: ["*"]`）

### 3.2 构建步骤

1) checkout
2) 安装依赖（同 backend）
3) 构建 multi-arch 二进制：
   - 创建 `dist/`
   - 循环 `arch in amd64 armv7 arm64 s390x`：
     - `env BUILD_TARGET=$arch ./scripts/cross-build.sh`
     - 若 arch==armv7，则将目录名改为 `arm`
     - `mkdir -p dist/$arch && cp build/ttyd dist/$arch/ttyd`
4) setup qemu + buildx
5) 登录 docker hub（secrets：`DOCKER_HUB_USER/DOCKER_HUB_TOKEN`）
6) 登录 ghcr（使用 `GITHUB_TOKEN`）
7) 计算 docker tags：
   - 若为 tag push：`tsl0922/ttyd:<tag>` 与 `<tag>-alpine`
   - 否则：`tsl0922/ttyd:latest` 与 `:alpine`
8) build/push：
   - 使用 `Dockerfile`：platforms `linux/amd64, linux/arm/v7, linux/arm64, linux/s390x`
   - 使用 `Dockerfile.alpine`：同平台
   - tags 同时推送到 docker hub 与 `ghcr.io/...`

---

## 4. `release.yml`：GitHub Release 产物整理

事实源：`.github/workflows/release.yml`。

### 4.1 触发条件
- push tags：`["*"]`

### 4.2 jobs

#### 4.2.1 `build`
- 复用 `backend.yml`（workflow_call）

#### 4.2.2 `publish`

步骤：
1) checkout
2) 检查版本一致性：
   - `TAG=$(git describe --tags --match "...")`
   - `VERSION=$(grep project CMakeLists.txt| awk '{print $3}')`
   - 若不一致：退出 1
3) download-artifact
4) 整理 build/：
   - `mkdir build`
   - 遍历 `ttyd.*/*`：
     - `target` 取 artifact 目录名（如 `ttyd.x86_64`）
     - 若文件名以 `.exe` 结尾，则输出名追加 `.exe`
     - 移动到 `build/$target`
   - 在 build/ 生成 `SHA256SUMS`：`sha256sum ttyd.* > SHA256SUMS`
5) 发布 release（`ncipollo/release-action@v1`）：
   - `artifacts: build/*`
   - `allowUpdates: true`
   - `draft: true`

> 复刻仓库应保持产物命名与 SHA256SUMS 生成方式，以便用户可以在 release 中下载并校验。

