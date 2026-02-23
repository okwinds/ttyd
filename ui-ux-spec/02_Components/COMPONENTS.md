# 组件目录（Components，像素级）

> 本文以“可复刻实现”为目标，逐个组件写明：用途、DOM 结构、样式（含固定 px 值）、状态、交互、微文案。  
> 事实源：`html/src/components/**`、`html/src/style/**`、以及 overlay/zmodem addon 源码。

---

## 0) 组件清单（Inventory）

本仓库 UI 的“本地组件”（不含三方 xterm 内部 DOM）：

1) `App`（入口容器，负责 URL 组装与默认 options）
2) `Terminal`（容器组件：创建 Xterm 实例 + Modal）
3) `Modal`（通用弹窗壳）
4) `OverlayAddon`（终端 overlay 提示层）
5) `ZmodemAddon`（文件传输 addon，含 trzsz drag&drop）
6) `Xterm`（本地封装类：把 xterm.js 与 WS 协议、prefs 应用、重连、文案等聚合在一起）

> 注意：xterm.js 本身会生成一套 DOM 结构与 class（例如 `.terminal`），这些来自三方库；像素级复刻需要锁定其版本与 CSS（见 `ui-ux-spec/07_Engineering_Constraints/ENGINEERING.md`）。

---

## 1) `App`（入口容器）

### 用途
负责：
- 计算 `wsUrl` 与 `tokenUrl`
- 声明前端默认 options（clientOptions、termOptions、flowControl）
- 渲染 `<Terminal />`

### 使用位置
- `html/src/index.tsx`：`render(<App />, document.body)`

### 结构（组件树）

```txt
App
└── Terminal(id="terminal-container", wsUrl, tokenUrl, clientOptions, termOptions, flowControl)
```

### 关键视觉/内容
无直接 DOM 文案输出；仅提供配置。

### URL 组装（必须一致）
事实源：`html/src/components/app.tsx:8-11`：

- `protocol = (window.location.protocol === 'https:') ? 'wss:' : 'ws:'`
- `path = window.location.pathname.replace(/[/]+$/, '')`（去掉末尾 `/`）
- `wsUrl = protocol + '//' + host + path + '/ws' + search`
- `tokenUrl = window.location.protocol + '//' + host + path + '/token'`

---

## 2) `Terminal`（终端容器组件）

### 用途
负责：
- 管理 `Xterm` 实例的生命周期；
- 在 mount 时先拉 token，再打开终端并建立 WS；
- 提供“发送文件”的 UI（Modal + `<input type=file multiple>`）。

### 使用位置
- `App` 渲染唯一实例。

### DOM 结构（未打开 Modal）
事实源：`html/src/components/terminal/index.tsx:35-46`。

```html
<div id="terminal-container">
  <!-- xterm.js 会在 open() 时把自己的 DOM 挂到这个 div 内 -->
  <!-- Modal show=false 时不渲染任何 modal DOM -->
</div>
```

### DOM 结构（Modal 打开时）

当 `state.modal === true` 时，`Terminal.render()` 结构为：

```html
<div id="terminal-container">
  <div class="modal">
    <div class="modal-background"></div>
    <div class="modal-content">
      <div class="box">
        <label class="file-label">
          <input class="file-input" type="file" multiple>
          <span class="file-cta">Choose files…</span>
        </label>
      </div>
    </div>
  </div>
</div>
```

微文案（必须逐字一致）：
- `Choose files…`（U+2026 省略号）

### 交互（Interaction）

#### 生命周期
事实源：`html/src/components/terminal/index.tsx:25-33`。

- `componentDidMount`：
  1) `await xterm.refreshToken()`
  2) `xterm.open(container)`
  3) `xterm.connect()`
- `componentWillUnmount`：`xterm.dispose()`

#### 打开 Modal
事实源：`html/src/components/terminal/index.tsx:48-51`。

- `showModal()`：`setState({ modal: true })`
- 该回调由 `Xterm` 的 `sendCb` 触发（通常在 Zmodem detect send 时调用）

#### 发送文件
事实源：`html/src/components/terminal/index.tsx:53-58`。

- `<input type=file multiple onChange={sendFile}>`
- `sendFile(event)`：
  - `setState({ modal:false })`
  - 读取 `event.target.files`
  - 若存在则 `xterm.sendFile(files)`

### 样式（Styles）

- 容器样式来自 `ui-ux-spec/01_Foundation/FOUNDATION.md` 中的 `#terminal-container` 规则
- Modal 样式详见下一个组件 `Modal`

---

## 3) `Modal`（弹窗壳）

### 用途
提供一个全屏遮罩 + 居中内容容器，用于展示子内容（这里用于文件选择）。

### 使用位置
- `Terminal` 内部：`<Modal show={modal}>{...file input...}</Modal>`

### DOM 结构
事实源：`html/src/components/modal/index.tsx:15-25`。

当 `show=true` 时输出：

```html
<div class="modal">
  <div class="modal-background"></div>
  <div class="modal-content">
    <div class="box">
      <!-- children -->
    </div>
  </div>
</div>
```

当 `show=false` 时：不渲染任何 DOM（返回 `false`）。

### 样式（必须逐条一致）
事实源：`html/src/components/modal/modal.scss`。

#### `.modal`
- `position: fixed; top:0; right:0; bottom:0; left:0;`
- `display: flex; align-items: center;`
- `overflow: hidden;`
- `z-index: 40;`

#### `.modal-background`
- `position: absolute; top:0; right:0; bottom:0; left:0;`
- `background-color: #4a4a4acc;`

#### `.modal-content`
- `margin: 0 20px;`
- `max-height: calc(100vh - 160px);`
- `overflow: auto;`
- `position: relative;`
- `width: 100%;`

内部 `.box`：
- `background-color: #fff;`
- `color: #4a4a4a;`
- `display: block;`
- `padding: 1.25rem;`

媒体查询（min-width 769px）：
- `.modal-content { margin: 0 auto; max-height: calc(100vh - 40px); width: 640px; }`

#### 文件选择相关 class

`.file-input`（隐藏 input）：
- `position: absolute; top:0; left:0;`
- `width: .01em; height: .01em;`
- `outline: none;`

`.file-cta`（可点击按钮）：
- `cursor: pointer;`
- `background-color: #f5f5f5;`
- `color: #6200ee;`
- `display: inline-flex; align-items: center; justify-content: flex-start;`
- `height: 2.25em;`
- `line-height: 1.5;`
- `border-color: #dbdbdb; border-radius: 3px;`
- `font-size: 1em; font-weight: 500;`
- `padding: calc(.375em - 1px) 1em;`
- `white-space: nowrap;`

### 交互与可访问性（现状复刻）
- 无点击遮罩关闭、无 ESC 关闭、无 focus trap（见 `ui-ux-spec/05_A11y/A11Y.md`）

---

## 4) `OverlayAddon`（终端提示层）

### 用途
在终端区域中央显示临时提示（连接状态、resize、复制剪刀等）。

### 使用位置
- 在 `Xterm.open()` 中加载 addon：`terminal.loadAddon(overlayAddon)`。

### DOM 结构
事实源：`html/src/components/terminal/xterm/addons/overlay.ts`。

- addon 构造函数创建一个 `div`（记为 `overlayNode`）
- 当调用 `showOverlay(msg)` 且 `terminal.element` 存在：
  - 若 `overlayNode` 尚未挂载，则 append 到 `terminal.element`
  - 设置 `overlayNode.textContent = msg`
  - 计算并设置 `overlayNode.style.top/left`，使其居中

### 样式（Inline CSS，必须一致）
详见 `ui-ux-spec/01_Foundation/FOUNDATION.md` Overlay token。

### 交互
- `mousedown` 捕获阶段阻止默认与冒泡（用于避免点击 overlay 影响终端）
- timeout 行为：
  - 若传入 `timeout`：在 timeout 后开始 fade-out 并移除
  - 若不传入 `timeout`：一直显示（直到被下一次 overlay 覆盖或手动移除）

### 微文案（来自调用方）
调用方文案定义在 `Xterm` 中（见 `ui-ux-spec/03_Patterns/PATTERNS.md`）。

---

## 5) `Xterm`（本地 xterm + WS 协议封装类）

> 虽然它不是 Preact 组件，但它是 UI 行为的核心，必须按“组件”方式规格化。

### 用途
负责：
- 创建 xterm.js `Terminal` 实例并挂载到 DOM
- 维护 WebSocket 连接与消息协议（hello/prefix frames）
- 应用服务端下发 preferences，并切换渲染器/启用 addon
- 管理重连与 overlay 文案
- 管理 beforeunload 提示与复制行为

### 使用位置
- `Terminal` 构造函数中：`this.xterm = new Xterm(props, this.showModal);`

### 关键内部状态（用于复刻）
事实源：`html/src/components/terminal/xterm/index.ts`：

- `opened: boolean`：首次连接标志
- `title?: string`：服务端设置的窗口标题
- `titleFixed?: string`：固定标题（若设置则不再跟随 title change）
- `resizeOverlay: boolean`：默认 true；`disableResizeOverlay` 会置 false
- `reconnect/doReconnect: boolean`：控制自动重连
- `closeOnDisconnect: boolean`：断开后 `window.close()`
- 流控：`written/pending`（见 patterns）

### 关键用户可见文本（必须逐字一致）
（更多见 patterns）
- beforeunload：`Close terminal? this will also terminate the command.`
- overlay：
  - `Reconnected`
  - `Connection Closed`
  - `Reconnecting...`
  - `Press ⏎ to Reconnect`
  - 复制：`✂`

---

## 6) `ZmodemAddon`（文件传输）

### 用途
支持两条文件传输线路：
- zmodem.js：基于终端内容检测 zmodem session，并调用 send/receive
- trzsz：支持拖拽上传（drop/dragover），并与 zmodem 检测共存

### 使用位置
- `Xterm.applyPreferences()`：当 `enableZmodem || enableTrzsz` 时构造并 `terminal.loadAddon(zmodemAddon)`

### UI/交互（可观察行为）

#### 发送文件（send）
事实源：`html/src/components/terminal/xterm/addons/zmodem.ts:115-138`。

- 当检测到 session `type === 'send'`：
  - 调用 `options.onSend()` → 触发 `Terminal.showModal()` → 打开文件选择 Modal
- 用户选择文件后：
  - `Zmodem.Browser.send_files(session, files, { on_progress })`
  - 完成后 `session.close()`；失败则 `reset()`

#### 接收文件（receive）
事实源：`html/src/components/terminal/xterm/addons/zmodem.ts:141-156`。

- session 监听 `offer`：
  - `offer.accept().then(payloads => saveAs(blob, filename))`
- `session.start()`

#### 进度输出（终端内）
事实源：`writeProgress()`（`html/src/components/terminal/xterm/addons/zmodem.ts:159-168`）。

在终端里写入一行（带回车 `\r`）：
```
${name} ${percent}% ${bytesHuman(offset, 2)}/${bytesHuman(size, 2)}\r
```

> 注意：这是写入终端内容，不是 overlay 文案。percent 保留 2 位小数（`toFixed(2)`）。

#### trzsz 拖拽上传
事实源：`trzszInit()`（`html/src/components/terminal/xterm/addons/zmodem.ts:65-91`）。

- 在 `terminal.element` 上：
  - `dragover`：`preventDefault()`
  - `drop`：`preventDefault()` 后调用 `uploadFiles(dataTransfer.items)`
- 终端 resize 时更新 trzsz columns
