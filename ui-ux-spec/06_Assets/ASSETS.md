# 资源清单（Assets）

> 本文列出 UI 直接引用的静态资源，以及它们在构建后如何进入最终 HTML。  
> 事实源：`html/src/template.html`、`html/webpack.config.js`、`html/src/favicon.png`。

---

## 1) favicon

- 源文件：`html/src/favicon.png`
- 构建行为：
  - `CopyWebpackPlugin` 将其复制到 `dist/` 根目录（`html/webpack.config.js:42-44`）
- HTML 引用：
  - 模板中：`<link inline rel="icon" type="image/png" href="favicon.png">`（`html/src/template.html:8`）
  - `inline` 属性意味着在 gulp inline 阶段可能被内联（取决于 gulp-inline-source 对 icon link 的处理）；无论是否内联，运行时必须能正确显示 favicon。

---

## 2) 终端样式依赖（非仓库文件）

终端视觉很大一部分来自三方依赖的 CSS：
- `@xterm/xterm/css/xterm.css`（在 `html/src/components/terminal/index.tsx` 与 `html/src/components/terminal/xterm/index.ts` 中被 import）

该 CSS 不在本仓库中，因此“资产内容”无法直接在仓库里枚举；复刻实现必须锁定 xterm 版本并引入对应 CSS（见 `ui-ux-spec/07_Engineering_Constraints/ENGINEERING.md`）。

