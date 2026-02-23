# 工程约束（Engineering Constraints）

> UI 像素级复刻不仅取决于本仓库源码，也取决于依赖版本（xterm.js、Preact、CSS loader 等）。  
> 本文锁定“必须一致”的工程约束，以保证 UI 行为与视觉可复刻。

事实源：`html/package.json`、`html/yarn.lock`、`html/webpack.config.js`、`html/gulpfile.js`。

---

## 1) 运行时依赖（必须按版本范围安装）

来自 `html/package.json:dependencies`：

- `preact`：`^10.19.6`
- `@xterm/xterm`：`^5.5.0`
- xterm addons：
  - `@xterm/addon-fit`：`^0.10.0`
  - `@xterm/addon-webgl`：`^0.18.0`
  - `@xterm/addon-canvas`：`^0.7.0`
  - `@xterm/addon-web-links`：`^0.11.0`
  - `@xterm/addon-clipboard`：`^0.1.0`
  - `@xterm/addon-unicode11`：`^0.8.0`
  - `@xterm/addon-image`：`^0.8.0`（用于 Sixel）
- 文件传输：
  - `zmodem.js`：`^0.1.10`（并有 yarn patch）
  - `trzsz`：`^1.1.5`
  - `file-saver`：`^2.0.5`
- 其他：
  - `whatwg-fetch`：`^3.6.20`
  - `decko`：`^1.2.0`

关键点（必须复刻）：
- **必须引入** `@xterm/xterm/css/xterm.css`（代码中显式 import）。
- 由于 UI 大部分 DOM/CSS 由 xterm.js 决定，像素级复刻需要保持 xterm 版本一致（建议锁到小版本/通过 yarn.lock 固定）。

---

## 2) 构建工具链（必须可重建）

来自 `html/package.json:devDependencies`：

- TypeScript：`^5.3.3`
- webpack：`^5.90.3`，webpack-dev-server：`^5.0.2`
- loaders：`ts-loader`、`sass-loader`、`css-loader`、`style-loader`
- CSS 抽取：`mini-css-extract-plugin`
- 压缩：`terser-webpack-plugin`、`css-minimizer-webpack-plugin`
- gulp：`^4.0.2`，以及 `gulp-inline-source`、`gulp-gzip`、`through2`、`gulp-rename`、`gulp-clean`
- lint/格式：`gts`、eslint（由 workflow 使用 `yarn run check`）

---

## 3) yarn patch（zmodem.js）

事实源：`html/package.json:resolutions`。

仓库声明：

```json
"resolutions": {
  "zmodem.js@^0.1.10": "patch:zmodem.js@npm%3A0.1.10#./.yarn/patches/zmodem.js-npm-0.1.10-e5537fa2ed.patch"
}
```

约束：
- 复刻仓库必须保留该 patch 机制（或在行为上等价），否则 zmodem 行为可能发生差异。

---

## 4) devServer 反向代理约束

事实源：`html/webpack.config.js:66-82`。

开发模式（`NODE_ENV!=production`）下：
- devServer 端口为 `9000`
- 将 `/token` 与 `/ws` 代理到 `http://localhost:7681`，并启用 ws proxy

该约束影响本地开发体验，建议复刻时保持一致。

---

## 5) 生成物约束：`src/html.h`

事实源：`html/gulpfile.js` 与 `spec/07_Infrastructure/FRONTEND_PIPELINE.md`。

复刻实现不要求字节一致，但必须保证：
- 后端能在 gzip 与非 gzip 的 Accept-Encoding 情况下返回可运行 UI；
- `index_html_size` 与解压后的 HTML 长度一致（否则后端 inflate 预分配长度会不匹配）。

