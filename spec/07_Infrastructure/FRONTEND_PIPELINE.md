# 前端构建与内嵌资源生成规格（`html/` → `src/html.h`）

> 本文定义如何将前端 SPA 构建为后端可内嵌的 C 头文件 `src/html.h`。  
> 目标：复刻实现不要求 `src/html.h` 字节级一致，但必须满足同等语义与运行行为：后端 `GET /` 能返回可工作的 UI（可 gzip 或解压）。  
> 事实源：`html/webpack.config.js`、`html/gulpfile.js`、`html/src/template.html`、`src/http.c`、`src/html.h`。

---

## 1. 前端工程概览

- webpack entry：`html/src/index.tsx`（`html/webpack.config.js:14-16`）
- 输出目录：`html/dist/`
- HTML 模板：`html/src/template.html`（使用 HtmlWebpackPlugin，`inject:false`，并在模板中用 `<script inline ...>` 标记）

---

## 2. webpack 构建规格

事实源：`html/webpack.config.js`。

### 2.1 dev/prod 模式判定
`devMode = process.env.NODE_ENV !== 'production'`。

### 2.2 输出文件命名
- dev：`[name].js`、`[name].css`
- prod：`[name].[contenthash].js`、`[name].[contenthash].css`

### 2.3 loader 规则

- `ts-loader`：处理 `.ts/.tsx`
- `sass-loader` + `css-loader` +（dev `style-loader` / prod `MiniCssExtractPlugin.loader`）处理 `.scss/.sass/.css`

### 2.4 HtmlWebpackPlugin 模板参数
- `title: "ttyd - Terminal"`
- `template: "./template.html"`
- `inject: false`
- `minify: { removeComments: true, collapseWhitespace: true }`

### 2.5 开发服务器（devServer）

事实源：`html/webpack.config.js:64-91`。

- 端口：`9000`
- `proxy`：
  - 将 `/token` 与 `/ws` 代理到 `http://localhost:7681`，并支持 ws 代理（`ws:true`）
- websocketServer：`sockjs`，path `/sockjs-node`

> 复刻实现应保留该 devServer 行为，以便本地调试前端时可直接连接本地后端 ttyd。

---

## 3. gulp 管线规格（生成 `src/html.h`）

事实源：`html/gulpfile.js`。

### 3.1 产物与关键中间件

产物：
- `html/dist/index.html`（webpack 产物）
- `html/dist/inline.html`（把 CSS/JS 等资源内联后的 HTML）
- `src/html.h`（将 `inline.html` gzip 后转为 C 数组 + 元信息）

关键依赖：
- `gulp-inline-source`：将模板中带 `inline` 属性的 `<link>`/`<script>` 内联进 HTML
- `gulp-gzip`：对 `inline.html` gzip
- `through2`：读取 buffer 并生成 C header 内容

### 3.2 生成的 C header 结构（语义）

`genHeader(size, buf, len)` 会生成三段：

1) `unsigned char index_html[] = { 0x.., 0x.., ... };`
2) `unsigned int index_html_len = <len>;`
3) `unsigned int index_html_size = <size>;`

其中：
- `<size>`：未压缩 `inline.html` 的字节长度（通过 `file.contents.length` 记录）
- `<buf>`：gzip 后的 bytes
- `<len>`：gzip bytes 的长度

### 3.3 与后端 HTTP 的契约

事实源：`src/http.c`。

- 默认情况下后端会把 `index_html[]` 作为 gzip bytes 使用：
  - 若客户端 `Accept-Encoding` 支持 gzip，则直接返回 `index_html` 并添加 `Content-Encoding: gzip`
  - 否则解压得到 HTML（长度为 `index_html_size`）并返回
- 解压缓存：
  - 首次解压会用 `inflateInit2(16+15)` 解 gzip，输出 buffer 长度预分配为 `index_html_size`
  - 解压结果缓存到 `html_cache`，后续复用

因此复刻实现的前端生成链路必须满足：
- `index_html` 必须是有效 gzip stream；
- 解压后得到的 HTML 必须可在浏览器中运行（含所需 JS/CSS 内联）；
- `index_html_size` 应等于解压后的字节长度（否则后端预分配长度不匹配可能导致解压失败）。

---

## 4. `yarn` scripts 约定（复刻仓库应保留）

事实源：`html/package.json:scripts`。

- `yarn run build`：
  - `NODE_ENV=production webpack && gulp`
  - 其中 gulp 默认任务依赖 `inline`，再 gzip + 生成 `../src/html.h`
- `yarn run inline`：
  - `NODE_ENV=production webpack && gulp inline`（只生成 inline.html，不生成 html.h）
- `yarn run start`：
  - `webpack serve`（dev server）

---

## 5. “不要求字节一致”的一致性验收建议

复刻实现不要求 `src/html.h` 的字节序列一致，但应保证以下验收通过：

1) 后端 `GET /` 返回的 HTML 能渲染出终端页面；
2) 页面能正确计算 `wsUrl/tokenUrl` 并建立连接；
3) 样式与微文案符合 `ui-ux-spec/**`（像素级复刻目标）；
4) gzip/解压路径均可工作：
   - 带 `Accept-Encoding: gzip` 的请求返回 gzip（或等价策略）
   - 不带 gzip 时能返回可用的未压缩 HTML。

