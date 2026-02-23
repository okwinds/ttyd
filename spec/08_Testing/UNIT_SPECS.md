# 单元测试规格（Unit Specs）

> 本仓库默认不附带自动化测试，但为了“复刻级可验收”，本规格必须定义建议实现的单元测试。  
> 重点：锁定协议常量、字符串处理、参数解析边界、以及已知“实现细节”（例如 PTY pause/resume 标志位不一致）。

---

## 1. `src/utils.c` 单元测试

### 1.1 `endswith(str, suffix)`

测试点（事实源：`src/utils.c:56-60`）：

- Case A：`str="abc.sock"`, `suffix=".sock"` → `true`
- Case B：`str="abc"`, `suffix="abc"` → **false**（注意：实现要求 `str_len > suffix_len` 才返回 true）
- Case C：`str="abc"`, `suffix="bc"` → `true`
- Case D：`suffix=""`（空后缀）：
  - 由于 `suffix_len=0` 且 `str_len>0`，比较 `strcmp(str + str_len, "")` 为 0，因此返回 true  
  - 复刻实现需决定是否保持这一行为；若保持，应加入测试

### 1.2 `uppercase/lowercase`

测试点：
- 输入 `SIGhup`：
  - `uppercase` 应把原字符串原地转为 `SIGHUP` 并返回 **末尾指针**（注意：函数返回值是移动后的 `s`，不是起始地址；`src/utils.c:40-46`），这属于可观察 API 行为
- `lowercase` 类似（返回末尾指针）

### 1.3 `get_sig(sig_name)` 与 `get_sig_name(sig)`

测试点（事实源：`src/utils.c:62-75`）：
- `get_sig("SIGHUP")` 与 `get_sig("HUP")` 返回相同信号
- `get_sig("9")` 返回 `9`（atoi）
- `get_sig_name(SIGHUP)` 生成的字符串必须以 `SIG` 开头且为大写

### 1.4 `open_uri(uri)`

不建议做严格单元测试（会调用系统命令）。可做“mock/隔离”或仅验证命令拼接逻辑（若复刻实现替换为可注入实现）。

---

## 2. `src/server.c` 参数解析单元测试（建议抽取为可测函数）

> 现有实现将解析写在 `main()` 里；复刻实现建议抽象为 `parse_args()` 以便测试，但必须保持行为一致。

### 2.1 `--credential` 格式校验

- 输入：`-c foo`（无 `:`）→ 返回 `-1`，并输出 `ttyd: invalid credential...`
- 输入：`-c user:pass` → `server->credential` 必须是 base64(`user:pass`)

### 2.2 `--index` 校验

- 输入：`-I <不存在路径>` → 返回 `-1`，并输出 `Can not stat index.html...`
- 输入：`-I <目录路径>` → 返回 `-1`，输出 `Invalid index.html path... is it a dir?`
- 输入：`-I ~/a.html`：
  - 仅当 `HOME` 存在时展开
  - 展开规则：`<HOME> + <optarg+1>`

### 2.3 `--base-path` trim 行为

- 输入：`-b "/base///"` → endpoints 应使用 `"/base"` 作为前缀
- 输入：`-b "/"` → trim 后长度为 0，端点不变

### 2.4 `--ping-interval` / `--srv-buf-size` 的负数校验

- `-P -1` → 返回 `-1`
- `-f -1` → 返回 `-1`

---

## 3. `src/pty.c` 的行为锁定测试（建议）

### 3.1 `paused` 标志位不一致（行为测试）

事实源：`src/pty.c:122-133` 与 `pty_spawn` 内设置 `paused=true`。

建议测试（无需真正 fork 子进程，可通过构造假的 `pty_process` + fake `uv_stream` 或在复刻实现中提供接口）：

- 初始化 `process->paused = true`
- 调用 `pty_resume(process)`：
  - 期望：会调用 `uv_read_start(...)`
  - 且 **不会**把 `paused` 置为 false（保持 true）
- 调用 `pty_pause(process)`：
  - 由于 `paused` 仍为 true，函数会直接 return（不会调用 `uv_read_stop`）

> 这是“已知实现细节”。如果复刻实现把它修复，会改变 PAUSE/RESUME 的可观察行为，因此必须通过测试将其锁死（除非明确选择“修复并记录差异”，见 `spec/09_Verification/KNOWN_GAPS.md`）。

