# UI 复刻 / 像素级标准（Replica / Pixel-Clone Standard）

本规范用于保证：**团队只看本 `ui-ux-spec/`，不看源码，也能像素级复刻出当前 UI（在同一基线下）**。

> 注意：本仓库 UI 由三方库 xterm.js 渲染大量 DOM/CSS。像素级复刻的前提是：依赖版本一致 + 引入同一份三方 CSS（见 `ui-ux-spec/07_Engineering_Constraints/ENGINEERING.md`）。

---

## 1) 单一基线（Single Baseline）

为了避免“不同系统/字体渲染导致像素差异”，本规范规定一套基线用于验收截图/视觉回归：

- 浏览器：Chromium（建议 Google Chrome Stable；建议版本区间 `>= 120`，以保证 WebGL/xterm addon 行为一致）
- 视口：`1280 x 720`（CSS px）
- Device Pixel Ratio：`1`
- 页面缩放：100%
- OS / 字体渲染：
  - Windows：优先命中 `Consolas`
  - macOS：优先命中 `Menlo`
  - Linux：优先命中 `Liberation Mono`
  - 如需严格一致，请在 CI 中使用同一 OS（推荐 Linux 容器）并安装对应字体
- 全局密度/分辨率开关：无（没有 root font-size 变化；也没有主题切换按钮）
- 主题/样式预设：单一暗色终端主题（颜色值见 `ui-ux-spec/01_Foundation/FOUNDATION.md`）

---

## 2) 硬规则（必须通过）

### 2.1 禁止占位符

以下内容在规格中一律禁止：
- 任何“需要读源码/看代码才能完成”的指令性描述（例如“请去对照代码”）
- 未完成标记（例如：`T-O-D-O / T-B-D / F-I-X-M-E` 一类）

### 2.2 禁止依赖性语言

禁止出现“必须对照截图、演示页面或样例工程才能做出来”的描述。规格必须自洽。

### 2.3 微文案逐字一致（Exact microcopy）

所有用户可见文本必须逐字一致，包括：
- `Choose files…`（注意：是 Unicode 省略号 `…`）
- `Press ⏎ to Reconnect`（注意：包含 `⏎` 字符）
- `Close terminal? this will also terminate the command.`（大小写与标点一致）
- overlay 提示：`Reconnected`、`Connection Closed`、`Reconnecting...`
- 剪刀字符：`✂`

### 2.4 可实现的细节必须齐全

对于每个描述的 UI（组件/页面/交互），必须写出：
- DOM 结构（或组件树 + 关键节点）
- 样式（明确 class 与 CSS 规则；含固定 px 值与 calc）
- 状态与交互（打开/关闭条件，点击/键盘/焦点规则）
- 对动态 UI 的确定性例子（例如：连接状态切换时的 overlay 文案）

---

## 3) 推荐工作流（写规格 / 验证规格）

1) 先写 `FOUNDATION.md`（tokens + global styles）
2) 再写 `COMPONENTS.md`（组件目录 + DOM + 样式 + 文案 + 状态机）
3) 再写 `PAGES.md` 与 `PATTERNS.md`（组合规则与交互流）
4) 最后跑 replica lint（严格模式）并修订到通过：

```bash
bash scripts/lint_replica_spec.sh --root ui-ux-spec
```
