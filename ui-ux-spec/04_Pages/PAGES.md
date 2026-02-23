# 页面规格（Pages，像素级）

> `ttyd` 前端是一个“单页单视图”的极简应用：整个页面只有一个终端容器（以及条件渲染的 Modal）。  
> 事实源：`html/src/index.tsx`、`html/src/components/app.tsx`、`html/src/components/terminal/index.tsx`、`html/src/style/index.scss`。

---

## Page 1：终端页（唯一页面）

### 1.1 路由

- 无前端路由库；页面路径完全由服务端提供（可能带 base-path 前缀）。
- 前端只使用：
  - `window.location.protocol/host/pathname/search`

### 1.2 顶层渲染位置

事实源：`html/src/index.tsx:9`。

- `render(<App />, document.body)`

这意味着 `body` 的直接子节点就是 `Terminal` 容器。

### 1.3 DOM 结构（不含三方 xterm 内部 DOM）

```html
<body>
  <div id="terminal-container">
    <!-- xterm.js 在 open() 时会挂载自身 DOM -->
    <!-- Modal show=true 时会追加 modal DOM（见组件规格） -->
  </div>
</body>
```

### 1.4 布局与样式（必须一致）

见 `ui-ux-spec/01_Foundation/FOUNDATION.md`：
- `html, body` 100% 高度与禁滚动
- `#terminal-container` 高度 100%
- `.terminal` padding 5px，高度 `calc(100% - 10px)`

### 1.5 页面状态（State）

页面有以下“可观察状态”：

1) **初始加载**：
   - 先拉 token（通常很快）
   - 打开 xterm 并 `fit()`
   - 建立 WS
2) **已连接（Running）**：
   - 终端可交互
3) **断开（Closed）**：
   - overlay 显示 `Connection Closed`
   - 可能自动重连或等待用户 Enter
4) **重连中（Reconnecting）**：
   - overlay 显示 `Reconnecting...`
5) **已重连（Reconnected）**：
   - overlay 显示 `Reconnected`（300ms）
6) **文件选择 Modal 打开**：
   - 覆盖全屏 modal（z-index 40）

页面状态机详见 `ui-ux-spec/03_Patterns/PATTERNS.md` 与 `spec/04_Business_Logic/STATE_MACHINES.md`。

