# 设计基础（Foundation，像素级）

> 本文列出 UI 的“可复刻基础”：基线、tokens（颜色/字体/间距/圆角/动效）、以及全局样式。  
> 事实源：`html/src/components/app.tsx`、`html/src/style/index.scss`、`html/src/components/modal/modal.scss`、`html/src/components/terminal/xterm/addons/overlay.ts`。

---

## 1) Replica 基线（用于截图/视觉回归）

基线定义见 `ui-ux-spec/00_Guides/REPLICA_STANDARD.md`。本文件只列 UI 内部“固定值”。

---

## 2) Tokens（解析后的确定值）

### 2.1 终端主题色（xterm theme）

事实源：`html/src/components/app.tsx:26-46`。

| Token | 值 |
|---|---|
| `term.foreground` | `#d2d2d2` |
| `term.background` | `#2b2b2b` |
| `term.cursor` | `#adadad` |
| `term.ansi.black` | `#000000` |
| `term.ansi.red` | `#d81e00` |
| `term.ansi.green` | `#5ea702` |
| `term.ansi.yellow` | `#cfae00` |
| `term.ansi.blue` | `#427ab3` |
| `term.ansi.magenta` | `#89658e` |
| `term.ansi.cyan` | `#00a7aa` |
| `term.ansi.white` | `#dbded8` |
| `term.ansi.brightBlack` | `#686a66` |
| `term.ansi.brightRed` | `#f54235` |
| `term.ansi.brightGreen` | `#99e343` |
| `term.ansi.brightYellow` | `#fdeb61` |
| `term.ansi.brightBlue` | `#84b0d8` |
| `term.ansi.brightMagenta` | `#bc94b7` |
| `term.ansi.brightCyan` | `#37e6e8` |
| `term.ansi.brightWhite` | `#f1f1f0` |

### 2.2 终端字体（xterm options）

事实源：`html/src/components/app.tsx:24-25`。

- `term.fontSize = 13`（px）
- `term.fontFamily = "Consolas,Liberation Mono,Menlo,Courier,monospace"`

### 2.3 Overlay（提示层）样式 token

事实源：`html/src/components/terminal/xterm/addons/overlay.ts`（inline style）。

- `overlay.borderRadius = 15px`
- `overlay.fontSize = xx-large`
- `overlay.opacity = 0.75`
- `overlay.padding = 0.2em 0.5em 0.2em 0.5em`
- `overlay.position = absolute`
- `overlay.userSelect = none`（分别写了 `-webkit-` 与 `-moz-`）
- `overlay.transition = opacity 180ms ease-in`（webkit/mozilla）
- `overlay.color = #101010`
- `overlay.backgroundColor = #f0f0f0`
- `overlay.hide`：先设 `opacity: 0`，200ms 后从 DOM 移除并把 opacity 复位为 `0.75`

### 2.4 Modal（文件选择弹窗）样式 token

事实源：`html/src/components/modal/modal.scss`。

- `modal.zIndex = 40`
- `modal.backgroundColor = #4a4a4acc`（半透明遮罩）
- `modal.box.backgroundColor = #fff`
- `modal.box.color = #4a4a4a`
- `.file-cta.backgroundColor = #f5f5f5`
- `.file-cta.color = #6200ee`
- `.file-cta.borderColor = #dbdbdb`
- `.file-cta.borderRadius = 3px`
- `.file-cta.fontWeight = 500`

---

## 3) Global styles（全局样式）

事实源：`html/src/style/index.scss`。

### 3.1 页面高度链（必须一致）

```css
html,
body {
  height: 100%;
  min-height: 100%;
  margin: 0;
  overflow: hidden;
}
```

含义：
- 页面禁止滚动（overflow hidden）
- 通过 `height:100%` 把高度传递给终端容器，确保终端铺满视口

### 3.2 终端容器样式

```css
#terminal-container {
  width: auto;
  height: 100%;
  margin: 0 auto;
  padding: 0;
}

#terminal-container .terminal {
  padding: 5px;
  height: calc(100% - 10px);
}
```

说明：
- `.terminal` 是 xterm.js 注入的容器 class（来自三方库），这里对其加 padding 与高度计算。

---

## 4) Motion（动效）

本 UI 的动效主要来自 overlay：

- 显示时：直接设置 `opacity: 0.75`
- 隐藏时：
  - 先设置 `opacity: 0`
  - 200ms 后从 DOM 移除并复位 opacity
- CSS transition：`opacity 180ms ease-in`

---

## 5) Z-index 层级（简表）

| 层 | z-index | 说明 |
|---|---:|---|
| Modal | 40 | 覆盖全屏（固定定位） |
| Overlay | 未显式设置 | 作为 `terminal.element` 的子节点，依赖 DOM 顺序覆盖终端内容 |

