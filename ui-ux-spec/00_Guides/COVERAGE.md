# 覆盖矩阵（Coverage Matrix，像素级）

> 本表用于防止“漏写规格”。Replica lint 通过是必要条件，但不等价于覆盖完整。  
> 状态含义：`Missing | Draft | Implementable | Replica`

---

## 1) Pages 覆盖

| Page | Spec path | Pixel structure | Interaction/keyboard | A11y | Fixtures | Regression notes | Status |
|---|---|---|---|---|---|---|---|
| 终端页（唯一页面） | `ui-ux-spec/04_Pages/PAGES.md` | ✅ | ✅ | ✅（按现状） | N/A（终端内容为动态） | 依赖 xterm.css 与字体渲染 | Replica |

---

## 2) Components 覆盖

| Component | Spec path | DOM/Slots | Styles/pixels | State machine | Keyboard | A11y | Fixtures | Status |
|---|---|---|---|---|---|---|---|---|
| App | `ui-ux-spec/02_Components/COMPONENTS.md` | ✅ | N/A | ✅ | N/A | N/A | N/A | Replica |
| Terminal | `ui-ux-spec/02_Components/COMPONENTS.md` | ✅ | ✅ | ✅ | ✅ | ✅（按现状） | N/A | Replica |
| Modal | `ui-ux-spec/02_Components/COMPONENTS.md` | ✅ | ✅ | ✅ | ✅（有限） | ✅（按现状） | N/A | Replica |
| OverlayAddon | `ui-ux-spec/02_Components/COMPONENTS.md` | ✅ | ✅ | ✅ | N/A | ✅（按现状） | N/A | Replica |
| Xterm（封装类） | `ui-ux-spec/02_Components/COMPONENTS.md` | ✅（行为） | 部分（依赖三方 CSS） | ✅ | ✅ | ✅（按现状） | N/A | Replica |
| ZmodemAddon | `ui-ux-spec/02_Components/COMPONENTS.md` | ✅（行为） | N/A | ✅ | ✅ | ✅（按现状） | N/A | Implementable |

> 注：ZmodemAddon 的像素级 UI 很大程度呈现在终端文本与 Modal 上；其主要验收在交互与微文案而非像素布局。

---

## 3) 高风险细节清单（已覆盖）

- [x] 高度链：`html/body` 100% + overflow hidden + `#terminal-container` 100% + `.terminal height calc(...)`
- [x] Modal：fixed 全屏 + z-index 40 + 遮罩颜色 `#4a4a4acc`
- [x] Focus/keyboard：Enter 触发重连、beforeunload 提示
- [x] 微文案：重连/关闭/剪刀/文件选择
- [x] rendererType webgl/canvas/dom 的降级策略

---

## 4) 已知缺口（若追求更严格像素级）

- [ ] 将 `@xterm/xterm/css/xterm.css` 的关键布局/字体规则摘要化（不建议全文复制；可提取必要片段并锁版本）

