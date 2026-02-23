# CLI 参数规格（复刻级）

> 本文定义 `ttyd` 的命令行接口（CLI）——包括每个参数的语义、默认值、校验规则、以及对系统行为的影响。  
> 事实源：`src/server.c`（`options[]`、`opt_string`、`print_help()`、`main()` 中的 `getopt_long` switch）。

## 1. 基本用法

```
ttyd [options] <command> [<arguments...>]
```

约束：
- `<command>` 必填；若缺失，程序输出错误 `ttyd: missing start command` 并返回 `-1`（见 `src/server.c:529-532`）。
- 当参数只有程序名（`argc==1`）时打印 help 并返回 `0`（见 `src/server.c:304-307`）。

## 2. 选项总表

> 表格中“默认值”是指在没有显式传参时的行为。  
> “副作用”列描述该参数如何影响 HTTP/WS/PTY/退出策略等。

| 短参数 | 长参数 | 需要值 | 默认 | 校验与错误 | 副作用（关键行为） |
|---:|---|:---:|---|---|---|
| `-p` | `--port` | 是 | `7681` | `<0` 时报 `ttyd: invalid port: ...` 并返回 `-1`（`src/server.c:381-387`） | 监听端口；允许 `0`（随机端口）。 |
| `-i` | `--interface` | 是 | `NULL` | 无直接校验 | 绑定网卡名或作为 UNIX socket 路径；若值以 `.sock`/`.socket` 结尾且 lws 支持 UNIX socket，则启用 `LWS_SERVER_OPTION_UNIX_SOCK` 并强制 `port=0`（`src/server.c:546-561`）。 |
| `-U` | `--socket-owner` | 是 | `""` | 无直接校验 | 仅在 UNIX socket 模式时使用：设置 `info.unix_socket_perms = socket_owner`（`src/server.c:553-555`）。|
| `-c` | `--credential` | 是 | unset | 必须包含 `:`，否则报 `ttyd: invalid credential, format: username:password` 并返回 `-1`（`src/server.c:396-400`） | 启用 Basic Auth：将 `username:password` base64 编码后保存在 `server->credential`；影响 HTTP 鉴权 + `/token` + WS 首包 token 校验。 |
| `-H` | `--auth-header` | 是 | unset | 无直接校验 | 启用“反向代理鉴权”模式：HTTP/WS 仅检查该 header 是否存在；WS 会从该 header 复制用户标识到 `pss->user`，最终注入子进程环境变量 `TTYD_USER`。参数会被转换为 **小写 + 末尾追加 `:`**（`src/server.c:581-587`）。 |
| `-u` | `--uid` | 是 | `-1` | `parse_int` 失败则退出（`src/server.c:255-264`） | 传给 lws：监听 socket 的 uid。 |
| `-g` | `--gid` | 是 | `-1` | 同上 | 传给 lws：监听 socket 的 gid。 |
| `-s` | `--signal` | 是 | `SIGHUP` | 若信号名无效则报 `ttyd: invalid signal: ...` 并返回 `-1`（`src/server.c:414-423`） | WS 断开时 kill 子进程使用的信号（Unix：`uv_kill(-pid, sig)`；Windows：`TerminateProcess`）。 |
| `-w` | `--cwd` | 是 | unset | 无直接校验 | 子进程工作目录：Unix child `chdir()`；Windows `CreateProcessW` 的 `cwd`。 |
| `-a` | `--url-arg` | 否 | `false` | 无 | 允许浏览器通过 URL query `arg=` 追加子进程 argv（见 `src/protocol.c:237-245`）。 |
| `-W` | `--writable` | 否 | `false`（只读） | 无 | 允许浏览器输入写入 PTY；否则 `INPUT` 被忽略（`src/protocol.c:308-315`）。 |
| `-t` | `--client-option` | 是 | 空对象 | `key=value` 解析失败则报 `ttyd: invalid client option...` 并返回 `-1`（`src/server.c:508-516`） | 将 `key=value` 写入 `client_prefs` JSON，并在 WS 初始消息 `SET_PREFERENCES` 下发给前端。`value` 会尝试按 JSON 解析，失败则作为字符串。 |
| `-T` | `--terminal-type` | 是 | `xterm-256color` | 无 | 子进程环境变量 `TERM` 的值（见 `src/protocol.c:132-135`）。 |
| `-O` | `--check-origin` | 否 | `false` | 无 | WS 连接时要求 `Origin` 与 `Host` 匹配（`src/protocol.c:223-228`）。 |
| `-m` | `--max-clients` | 是 | `0`（无限制） | `<0` 会被 `parse_int` 退出或后续逻辑异常；实现未显式拒绝负数（`src/server.c:369-371`） | 限制同时在线客户端数；达到上限时 WS 过滤阶段拒绝连接（`src/protocol.c:208-211`）。 |
| `-o` | `--once` | 否 | `false` | 无 | 只允许一个客户端：已有客户端时 WS 过滤阶段拒绝；且当最后一个客户端断开时 `exit(0)`（`src/protocol.c:204-207, 382-387`）。 |
| `-q` | `--exit-no-conn` | 否 | `false` | 无 | 当所有客户端断开时退出进程（`src/protocol.c:382-387`）。 |
| `-B` | `--browser` | 否 | `false` | 无 | 启动后自动打开 `http(s)://localhost:<port>`。 |
| `-I` | `--index` | 是 | unset | `stat` 失败/是目录则报错并返回 `-1`（`src/server.c:436-444`） | 使用自定义 index.html 文件替代内嵌 UI。支持 `~/` 展开（依赖 `HOME`，见 `src/server.c:428-435`）。 |
| `-b` | `--base-path` | 是 | unset | 仅截断末尾 `/`；空串则忽略（`src/server.c:447-451`） | 将 `endpoints.ws/index/token/parent` 统一添加 base-path 前缀（`src/server.c:452-456`）。 |
| `-P` | `--ping-interval` | 是 | `5`（仅 LWS>=4） | `<0` 报错并返回 `-1`（`src/server.c:459-467`） | 配置 lws retry/idle policy：`secs_since_valid_ping` 与 `secs_since_valid_hangup`。 |
| `-f` | `--srv-buf-size` | 是 | `4096` | `<0` 报错并返回 `-1`（`src/server.c:469-476`） | 设置 `info.pt_serv_buf_size`，影响 HTTP 分块发送上限。 |
| `-6` | `--ipv6` | 否 | 关闭 | 仅当编译启用 `LWS_WITH_IPV6` 才会在 help 中出现 | 移除 `LWS_SERVER_OPTION_DISABLE_IPV6`（`src/server.c:477-479`）。 |
| `-S` | `--ssl` | 否 | 关闭 | 仅当编译启用 TLS 支持才可用 | 开启 TLS，并设置证书/私钥路径；额外添加 `ALLOW_NON_SSL_ON_SSL_PORT` 与 `REDIRECT_HTTP_TO_HTTPS`（`src/server.c:563-575`）。 |
| `-C` | `--ssl-cert` | 是 | `""` | 无 | TLS 证书路径（`info.ssl_cert_filepath`）。 |
| `-K` | `--ssl-key` | 是 | `""` | 无 | TLS 私钥路径（`info.ssl_private_key_filepath`）。 |
| `-A` | `--ssl-ca` | 是 | `""` | 无 | TLS 客户端证书校验 CA 路径；若设置则要求客户端证书有效（`src/server.c:570-573`）。 |
| `-d` | `--debug` | 是 | `LLL_ERR|LLL_WARN|LLL_NOTICE` | `parse_int` 失败则退出 | 调用 `lws_set_log_level(debug_level, NULL)` 设置日志等级。 |
| `-v` | `--version` | 否 | - | - | 打印 `ttyd version <TTYD_VERSION>` 并返回 `0`（`src/server.c:354-356`）。 |
| `-h` | `--help` | 否 | - | - | 打印 help 并返回 `0`（`src/server.c:351-354`）。 |

## 3. `--client-option`（`-t`）的 JSON 语义

事实源：`src/server.c:503-519`。

- 输入格式：`key=value`
- `value` 的解析：
  - 先 `json_tokener_parse(value)`
  - 若返回非 `NULL`，则写入 JSON 值（可为 boolean/number/object/array/string/null）
  - 若返回 `NULL`，则写入 JSON string（原样字符串）

约束/边界：
- 该机制使前端偏好设置可以由服务端强制下发并覆盖默认值（详见 `spec/03_API/ENDPOINTS.md` 与 `spec/04_Business_Logic/RULES.md`）。
- 解析实现中存在一个可观察细节：`case 't'` 内部 for-loop 复用 `optarg`，因此形如 `-t a=b c=d`（无重复 `-t`）的写法不会按预期解析第二项；复刻时应保持与现有实现一致（即不支持无 `-t` 前缀的多项）。  

## 4. base-path 重写规则（`--base-path`）

事实源：`src/server.c:446-457`。

- 输入 `path` 会先复制到长度 128 的局部 buffer，并**去掉末尾所有 `/`**
- 若去掉后长度为 `0`，则不做任何事
- 否则对四个端点字段做统一前缀化：
  - `endpoints.ws`
  - `endpoints.index`
  - `endpoints.token`
  - `endpoints.parent`
- 拼接规则为：`<base-path><old-endpoint>`
  - 例如 base-path `/mounted/here`，默认 `endpoints.ws="/ws"` → `/mounted/here/ws`

