# 打包与发布规格（Snap / Release 产物）

> 本文定义 `snap/` 打包与 GitHub Release 产物结构。  
> 事实源：`snap/snapcraft.yaml`、`.github/workflows/release.yml`、`.github/workflows/backend.yml`。

---

## 1. Snapcraft（`snap/snapcraft.yaml`）

事实源：`snap/snapcraft.yaml`。

### 1.1 基本元数据

- `name: ttyd`
- `adopt-info: ttyd`
- `grade: stable`
- `confinement: classic`
- `base: core20`
- `compression: lzo`
- `license: MIT`
- `assumes: [command-chain]`

### 1.2 app 定义

apps:
- `ttyd`：
  - `command: usr/bin/ttyd`
  - `command-chain: [bin/homeishome-launch]`

> `command-chain` 表示启动时先运行链式脚本。snap 中通过 `stage-snaps` 引入 `homeishome-launch`。

### 1.3 parts：ttyd

- `source`: `https://github.com/tsl0922/ttyd`（git）
- `plugin`: `cmake`
- `cmake-parameters`: `-DCMAKE_INSTALL_PREFIX=/usr`
- `build-environment`: `LDFLAGS: "-pthread"`
- `override-pull`：
  - `snapcraftctl pull`
  - `snapcraftctl set-version "$(git describe --tags | sed 's/^v//' | cut -d "-" -f1)"`
- `build-packages`：
  - `build-essential`
  - `libjson-c-dev`
  - `libwebsockets-dev`
- `stage-packages`：
  - `libjson-c4`
  - `libwebsockets15`

### 1.4 parts：homeishome-launch

- `plugin: nil`
- `stage-snaps: [homeishome-launch]`

---

## 2. GitHub Release 产物结构

事实源：`.github/workflows/release.yml` 与 `backend.yml`。

### 2.1 artifact → release 的重命名规则

backend workflow 会上传 artifact：
- 名称：`ttyd.<target>`
- 内容：`build/ttyd*`

release workflow 下载后会整理为：

- `build/ttyd.<target>`（无扩展名）
- 若原文件名以 `.exe` 结尾，则输出名为 `build/ttyd.<target>.exe`

### 2.2 校验文件：SHA256SUMS

release workflow 会在 `build/` 下生成：

- `SHA256SUMS`：通过 `sha256sum ttyd.* > SHA256SUMS` 生成（`release.yml:32`）

### 2.3 发布动作

使用 `ncipollo/release-action@v1`：
- `artifacts: build/*`
- `allowUpdates: true`
- `draft: true`

---

## 3. 复刻实现的发布验收

复刻仓库应至少保证：

1) tag push 能触发 release pipeline；
2) 产物命名与 SHA256SUMS 生成方式与本规格一致；
3) release 为 draft（便于人工检查）。

