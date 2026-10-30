---
sidebar_position: 15
title: "Web Dashboard"
description: "基于浏览器的仪表板，用于管理配置、API 密钥、会话、日志、分析、定时任务和技能"
---

# Web Dashboard

本页的 Python 依赖命令使用 [PM 准备的源码环境](../../reference/package-management.md#developer-workflow)。
依赖变更后，请重新激活该 checkout 并重启 Hermes。

Web Dashboard 是一个基于浏览器的 UI，用于管理你的 Hermes Agent 安装。无需编辑 YAML 文件或运行 CLI 命令，即可通过简洁的 Web 界面配置设置、管理 API 密钥并监控会话。

## 快速开始

```bash
hermes dashboard
```

这将启动一个本地 Web 服务器，并在浏览器中打开 `http://127.0.0.1:9119`。Dashboard 完全在你的机器上运行——数据不会离开 localhost。

### 选项

| 标志 | 默认值 | 描述 |
|------|---------|-------------|
| `--port` | `9119` | Web 服务器运行端口 |
| `--host` | `127.0.0.1` | 绑定地址 |
| `--no-open` | — | 不自动打开浏览器 |
| `--insecure` | 关闭 | **已弃用 / 无实际作用。** 此参数曾用于在非回环绑定上绕过鉴权；自 2026 年 6 月安全加固后，公网绑定始终需要用户名/密码或 OAuth 鉴权提供方。若需保持仅本地访问，请绑定 `127.0.0.1` 并通过隧道连接。 |

```bash
# 自定义端口
hermes dashboard --port 8080

# 绑定到所有接口（在共享网络上请谨慎使用）
hermes dashboard --host 0.0.0.0

# 启动时不打开浏览器
hermes dashboard --no-open
```

## 管理多个 profile

Dashboard 是一个**机器级**管理界面：一台服务器管理该机器上的每个
[profile](../profiles.md)。侧边栏中的 profile 切换器（只要存在不止一个 profile 就会显示）
决定管理页面读写的 profile —— Config、API Keys、Skills、
MCP、Models 和 Chat 标签页都跟随它。当选中的不是 dashboard 自己的
profile 时，一条琥珀色横幅会标明被管理的 profile，
这样写入目标永远不会含糊。

选择保存在 URL 中（`?profile=<name>`），因此像
`http://127.0.0.1:9119/skills?profile=worker` 这样的深链会带着预选的
切换器打开，并在刷新后保持。

从 profile 别名启动 dashboard 会路由到机器级
dashboard，而不是再启动第二个服务器：

```bash
worker dashboard
# → 已在运行：在浏览器中打开 ?profile=worker
# → 未运行：启动机器级 dashboard，并预选 "worker"
```

这样启动的 dashboard 会在整个服务器生命周期内把 `worker` 作为回退作用域：
省略 `?profile=` 的深链（例如
`/chat?resume=<id>` 链接）仍会在 `worker` 下打开，因此内嵌聊天
能看到该 profile 的 MCP 服务器、模型和技能。URL 中显式的 `?profile=`
始终优先。

传入 `--isolated` 可退出该行为，运行一个专属于该
profile 的服务器（统一之前的行为 —— 如果你有意用不同的认证
暴露不同 profile 的 dashboard，这会很有用）。

**Chat** 标签页也跟随切换器：限定作用域的聊天会用所选 profile 的
`HERMES_HOME` 派生出它的 PTY 子进程，因此对话会带着该 profile 的模型、
技能、内存和会话历史运行。切换 profile 会启动一个新的终端会话。

Hub 操作（技能安装/更新/卸载、MCP 安装、工具集设置）
使用目标 profile 自己的机密作用域运行 —— 它的 `.env` 和配置的
机密来源 —— 而不是 dashboard 进程的环境；这也包括从机器级 dashboard
针对 `default` profile 的操作。

保持不变、*不会*被切换器吸收的：gateway
进程（通过 `hermes -p <name> gateway …` 管理）、每个 profile 的
会话数据库，以及 cron 调度器（Cron 页面已用自带的过滤器
跨 profile 聚合）。

## 前置条件

FastAPI、Uvicorn 和平台 PTY 辅助库是 Hermes 核心依赖。`web` extra 提供 HTTP 栈的精确约束，`pty` extra 为空。
标准 PM 安装通过 `all` 选择 `web`；消息和语音功能仍需单独请求。
如依赖损坏，请运行 `hermes pm repair` 并重启 Hermes。

在没有依赖项的情况下运行 `hermes dashboard` 时，它会告诉你需要安装什么。如果前端尚未构建且 `npm` 可用，则会在首次启动时自动构建。

Chat 标签页是每次 `hermes dashboard` 启动的一部分——内嵌的浏览器聊天面板（通过 PTY/WebSocket 运行 TUI）始终可用，无需任何额外参数。

## 页面

### Status（状态）

首页显示你的安装的实时概览：

- **Agent 版本**和发布日期
- **Gateway 状态**——运行中/已停止、PID、已连接平台及其状态
- **活跃会话**——过去 5 分钟内活跃的会话数量
- **最近会话**——最近 20 个会话的列表，包含模型、消息数、token 用量和对话预览

状态页每 5 秒自动刷新一次。

### Chat（聊天）

**Chat** 标签页将完整的 Hermes TUI（与 `hermes --tui` 相同的界面）直接嵌入浏览器。你在终端 TUI 中能做的一切——斜杠命令、模型选择器、工具调用卡片、Markdown 流式输出、clarify/sudo/approval 提示、皮肤主题——在这里都完全一致，因为 Dashboard 运行的是真实的 TUI 二进制文件，并通过 [xterm.js](https://xtermjs.org/) 的 WebGL 渲染器以像素级精度渲染其 ANSI 输出。

**工作原理：**

- `/api/pty` 打开一个经 Dashboard 会话 token 认证的 WebSocket
- 服务器在 POSIX 伪终端后面启动 `hermes --tui`
- 按键传输到 PTY；ANSI 输出流式返回浏览器
- xterm.js 的 WebGL 渲染器将每个单元格绘制到整数像素网格；鼠标追踪（SGR 1006）、宽字符（Unicode 11）和方框绘制字形均原生渲染
- 调整浏览器窗口大小会通过 `@xterm/addon-fit` 插件调整 TUI 大小

**恢复已有会话：** 在 **Sessions** 标签页中，点击任意会话旁的播放图标（▶）。这会跳转到 `/chat?resume=<id>` 并以 `--resume` 参数启动 TUI，加载完整历史记录。

**前置条件：**

- Node.js（与 `hermes --tui` 相同的要求；TUI 包在首次启动时构建）
- `ptyprocess`——POSIX 上的核心依赖
- POSIX 内核（Linux、macOS 或 WSL2）。`/chat` 终端面板特别需要 POSIX PTY——原生 Windows Python 没有等效实现，因此在原生 Windows 安装上，Dashboard 的其余部分（sessions、jobs、metrics、config editor）可以正常工作，但 `/chat` 标签页会显示提示，告知你需要使用 WSL2 才能使用该功能。

关闭浏览器标签页后，PTY 会在服务器端被干净地回收。重新打开会启动一个新会话。

要让 [Hermes Desktop](#connecting-hermes-desktop-to-a-remote-backend) 连接另一台机器上运行的 dashboard 而不是它自带的本地后端，请参阅下面的远程后端部分。

### 将 Hermes Desktop 连接到远程后端 {#connecting-hermes-desktop-to-a-remote-backend-pages}

Hermes Desktop 通常启动自己的本地后端，但它也可以通过 **Settings → Gateways → Remote gateway** 连接到远程机器（VM、homelab 机器等）上运行的 dashboard。这是 "Desktop 说后端已就绪但聊天永远不工作" 这类报告最常见的来源，因为 Desktop 的就绪检查验证的内容比实时聊天连接实际需要的少。

:::info 前提：远程主机上必须有 `hermes dashboard` 在运行
Desktop 连接的"远程后端"**就是**远程机器上运行的 `hermes dashboard` 进程 —— 也就是本页记录的这个服务器。它必须在下面任何步骤生效之前就启动并可访问；Desktop 是连接到它，而不是替你启动它。请让它运行在 `systemd`/`tmux`/等之下，以熬过注销和重启。**gateway**（Telegram/Discord/Slack/等）是一个*独立*的长期运行进程 —— 如果你依赖消息渠道，请单独启动它；它不是桌面应用连接的东西。
:::

Desktop 的"远程后端已就绪"探针只访问 `GET /api/status`，这是一个公开端点 —— 只要主机上*任何* dashboard 在运行，它就会应答。实时聊天连接是到一个**独立**的 WebSocket `/api/ws`（以及 `/api/pty`），该 socket 还受状态探针从不触及的两项检查把关：

1. **你必须通过认证。** 当 dashboard 绑定到非回环地址时，它会启用认证门禁。请用用户名和密码保护它（随附的[用户名/密码提供商](#usernamepassword-provider-no-oauth-idp)）；Desktop 登录一次，并通过一次性票据为 WebSocket 复用产生的会话。未配置提供商时，非回环 dashboard 会在启动时**失败即关闭**。
2. **绑定主机必须允许该客户端，并与 Host 头匹配。** 回环绑定（`127.0.0.1`）只接受回环客户端，因此无论凭据如何，远程机器都会在 socket 层被拒绝。请绑定到非回环地址（`--host 0.0.0.0`），让对端 IP 守卫放行远程客户端。你在 Desktop 中输入的远程 URL 必须通过与其绑定相同的主机到达 dashboard —— DNS 重绑定守卫要求 Host 头匹配。

#### 远程 dashboard 配置 {#remote-dashboard-setup}

设置用户名和密码，然后把 dashboard 绑定到可达地址运行。以 `systemd` 服务为例：

```ini
[Service]
EnvironmentFile=%h/.hermes/.env
ExecStart=/path/to/venv/bin/python -m hermes_cli.main dashboard \
    --host 0.0.0.0 --port 9119 --no-open
Restart=always
RestartSec=10
# 退出码 78（EX_CONFIG）是有意的拒绝（"this host is already served by PID … on
# another port"）；停在该状态好过让一个什么都没监听的进程无限重启。
RestartPreventExitStatus=78
```

一个后端服务整台主机，因此当 `hermes dashboard` 发现一个无法用你输入的 `--host`/`--port` 服务的在线后端时，它会以退出码 78 拒绝并指出持有者。请停止那个后端，或用 `--isolated` 给该服务一个专属服务器。Desktop 应用自己的回环后端从不主张主机所有权，因此两者可以共存，无需任何标志。

其中 `~/.hermes/.env` 包含：

```bash
HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=choose-a-strong-password
HERMES_DASHBOARD_BASIC_AUTH_SECRET=<32+ random bytes; openssl rand -base64 32>
```

然后在 Desktop 中输入**远程 URL**（例如 `http://VM_IP:9119`）并用该用户名和密码**登录**。完整配置面见[用户名/密码提供商](#usernamepassword-provider-no-oauth-idp)部分。

:::tip 重试 Desktop 之前先确认门禁已启用
从任意机器检查 dashboard 是否宣告了用户名/密码提供商：

```bash
curl -s http://VM_IP:9119/api/status | jq '.auth_required, .auth_providers'
# true
# ["basic"]
```

- `auth_required: true` 且提供商列表中有 `"basic"` → Desktop 的**登录**流程会成功。
- `auth_required: false` → 绑定是回环，或门禁未启用。请绑定到非回环地址。
- `auth_required: true` 但没有 `"basic"` 提供商 → 用户名/密码环境变量未加载。请先修好这些。
:::

如果 `/api/status` 显示门禁已启用且提供商为 `"basic"`，而 Desktop 登录后*仍然*连接失败，那么问题已超出基础配置 —— 请在同一重试时间窗内抓取一份新的 `desktop.log`（Settings → Gateways → Open logs）以及 dashboard 的日志，并查找 `/api/ws` 的关闭码（4403 = 聊天 WS 被请求守卫拒绝，例如 Host/对端不匹配；4401 = WS 票据未通过认证）。

### Config（配置）

`config.yaml` 的表单式编辑器。所有 150+ 个配置字段均从 `DEFAULT_CONFIG` 自动发现，并按标签页分类组织：

- **model** — 默认模型、提供商、基础 URL、推理设置
- **terminal** — 后端（local/docker/ssh/modal）、超时、Shell 偏好
- **display** — 皮肤、工具进度、恢复显示、spinner 设置
- **agent** — 最大迭代次数、gateway 超时、服务层级
- **delegation** — 子 agent 限制、推理力度
- **memory** — 提供商选择、上下文注入设置
- **approvals** — 危险命令审批模式（smart/manual/off）
- 更多——config.yaml 的每个部分都有对应的表单字段

具有已知有效值的字段（terminal 后端、皮肤、审批模式等）渲染为下拉菜单。布尔值渲染为开关。其余均为文本输入框。

**操作：**

- **Save** — 立即将更改写入 `config.yaml`
- **Reset to defaults** — 将所有字段恢复为默认值（点击 Save 前不会保存）
- **Export** — 将当前配置下载为 JSON
- **Import** — 上传 JSON 配置文件以替换当前值

:::tip
配置更改在下一次 agent 会话或 gateway 重启时生效。Web Dashboard 编辑的是 `hermes config set` 和 gateway 读取的同一个 `config.yaml` 文件。
:::

### API Keys（API 密钥）

管理存储 API 密钥和凭据的 `.env` 文件。密钥按类别分组：

- **LLM Providers** — OpenRouter、Anthropic、OpenAI、DeepSeek 等
- **Tool API Keys** — Browserbase、Firecrawl、Tavily、ElevenLabs 等
- **Messaging Platforms** — Telegram、Discord、Slack bot token 等
- **Agent Settings** — 非敏感环境变量，如 `API_SERVER_ENABLED`

每个密钥显示：
- 是否已设置（带有值的脱敏预览）
- 用途说明
- 提供商注册/密钥页面的链接
- 用于设置或更新值的输入框
- 删除按钮

高级/不常用的密钥默认隐藏，可通过开关显示。

### Sessions（会话）

浏览和检查所有 agent 会话。每行显示会话标题、来源平台图标（CLI、Telegram、Discord、Slack、cron）、模型名称、消息数、工具调用数以及最后活跃时间。实时会话以脉冲徽章标记。

- **Search** — 使用 FTS5 对所有消息内容进行全文搜索。结果显示高亮片段，展开时自动滚动到第一条匹配消息。
- **Expand** — 点击会话以加载完整消息历史。消息按角色（user、assistant、system、tool）用颜色区分，并以带语法高亮的 Markdown 渲染。
- **Tool calls** — 包含工具调用的 assistant 消息显示可折叠块，包含函数名和 JSON 参数。
- **Delete** — 使用垃圾桶图标删除会话及其消息历史。

### Logs（日志）

查看 agent、gateway 和错误日志文件，支持过滤和实时追踪。

- **File** — 在 `agent`、`errors` 和 `gateway` 日志文件之间切换
- **Level** — 按日志级别过滤：ALL、DEBUG、INFO、WARNING 或 ERROR
- **Component** — 按来源组件过滤：all、gateway、agent、tools、cli 或 cron
- **Lines** — 选择显示行数（50、100、200 或 500）
- **Auto-refresh** — 切换实时追踪，每 5 秒轮询新日志行
- **Color-coded** — 日志行按严重程度着色（错误为红色，警告为黄色，debug 为暗色）

### Analytics（分析）

基于会话历史计算的用量和成本分析。选择时间段（7、30 或 90 天）查看：

- **Summary cards** — 总 token 数（输入/输出）、缓存命中率、总估算或实际成本，以及总会话数和日均值
- **Daily token chart** — 堆叠柱状图，显示每日输入和输出 token 用量，悬停提示显示明细和成本
- **Daily breakdown table** — 每日日期、会话数、输入 token、输出 token、缓存命中率和成本
- **Per-model breakdown** — 显示每个使用模型的会话数、token 用量和估算成本的表格

### Cron（定时任务）

创建和管理按定期计划运行 agent prompt 的定时任务。

- **Create** — 填写名称（可选）、prompt、cron 表达式（如 `0 9 * * *`）和投递目标（local、Telegram、Discord、Slack 或 email）
- **Job list** — 每个任务显示其名称、prompt 预览、计划表达式、状态徽章（enabled/paused/error）、投递目标、上次运行时间和下次运行时间
- **Pause / Resume** — 在活跃和暂停状态之间切换任务
- **Trigger now** — 在正常计划之外立即执行任务
- **Delete** — 永久删除定时任务

### Profiles（Profile）

创建和管理 [profile](../profiles.md) —— 拥有各自配置、技能和会话的隔离 Hermes 实例。

- **Profile 卡片** —— 每个显示其模型/提供商、技能数、gateway 状态、描述和徽章（活跃、默认、别名）
- **Create** —— 名称 + 可选的自默认克隆 / 克隆全部 / 不含内置技能，以及描述和模型；专用的 Profile Builder 页面（`/profiles/new`）提供完整流程（模型、MCP、技能）
- **管理技能与工具** —— 跳转到限定到该 profile 的 Skills 页面（同时设置侧边栏的 profile 切换器）
- **Set as active** —— 翻转粘性默认值，**将来的 CLI/gateway 运行**会采用它（等同于 `hermes profile use`）。这*不*改变 dashboard 管理的是哪个 profile —— 那是 profile 切换器的职责
- **编辑模型 / 描述 / SOUL** —— 就地编辑并写入该 profile
- **Rename / Delete** —— 仅限具名 profile

### Skills（技能）

浏览、搜索和切换已安装的技能与工具集，并从 hub 安装新的。技能从 `~/.hermes/skills/` 加载，并按类别分组。

- **Search** —— 按名称、描述或类别过滤已安装的技能和工具集
- **Category filter** —— 点击类别标签缩小列表范围（如 MLOps、MCP、Red Teaming、AI）
- **Toggle** —— 使用开关启用或禁用单个技能。更改在下一次会话时生效。
- **Toolsets** —— 单独的视图显示内置工具集（文件操作、Web 浏览等），包含其活跃/非活跃状态、设置要求和包含的工具列表
- **Browse hub** —— 第三个视图跨所有来源搜索技能 hub（等同于 `hermes skills search`），按标识符安装任意结果并显示实时安装日志，还提供 "Update all" 按钮刷新已安装的技能。

![技能管理页 —— Browse hub 视图：搜索、安装和更新](/img/dashboard/admin-skills-hub.png)

### MCP

无需 CLI 即可管理 [MCP](./mcp) 服务器。与 `hermes mcp` 读取的是 `config.yaml` 中同一个 `mcp_servers` 块。

**你的 MCP 服务器：**

- **Add** —— 注册 HTTP/SSE 服务器（URL）或 stdio 服务器（命令 + 参数），stdio 服务器可带 `KEY=VALUE` 环境变量
- **Enable / disable** —— 启用或停用服务器而不删除它。停用的服务器保留在配置中，以便之后重新启用。在下一次 gateway 重启时生效。
- **Test** —— 连接服务器、列出其工具并断开 —— 在 agent 依赖它之前验证连接
- **Remove** —— 从配置中删除服务器
- 机密形态的环境变量值在列表视图中会被打码

**目录：** 浏览 Nous 批准的 MCP 服务器（随附的 `optional-mcps/`
目录），一键安装任意一个。需要 API 密钥的条目
会就地提示输入；值会写入 `.env`。这与
`hermes mcp catalog` / `hermes mcp install` 使用的是同一个目录。

![MCP 管理页 —— 带启用/停用开关的服务器列表，以及安装目录](/img/dashboard/admin-mcp.png)

### Webhooks

管理动态 [webhook 订阅](../messaging/webhooks.md)。必须先在消息设置中
启用 webhook 平台；未启用时该页面会显示提示。

- **Create** —— 名称、描述、事件过滤器、投递目标、可选的直投模式，以及 agent prompt。创建后页面会显示路由 URL 和一次性 HMAC 机密供复制。
- **Enable / disable** —— 启用或停用订阅。停用的路由保留在订阅文件中，但 gateway 会拒绝它们传入的事件（403）。gateway 会热重载该文件，因此更改在下一个事件时生效 —— 无需重启。
- **List** —— 每个订阅显示其 URL、事件和投递目标
- **Delete** —— 删除订阅

![Webhooks 管理页 —— 带启用/停用开关的订阅](/img/dashboard/admin-webhooks.png)

### Pairing

无需 CLI 即可批准和撤销消息用户 —— 远程管理员
可以用它把 Telegram/Discord/等用户接入已配对的 gateway。与
`hermes pairing` 完全对等。

- **待处理请求** —— 每个显示平台、代码、用户和时长，并带 Approve 按钮
- **已批准用户** —— 每个显示平台和用户，并带 Revoke 按钮
- **Clear pending** —— 丢弃所有未完成的配对码

![Pairing 管理页](/img/dashboard/admin-pairing.png)

### Channels

从浏览器把 Hermes 连接到任意消息平台 —— 与
`hermes setup gateway` 完全对等。该页面列出所有支持的渠道（Telegram、
Discord、Slack、Matrix、Mattermost、WhatsApp、Signal、BlueBubbles/iMessage、
Email、SMS/Twilio、DingTalk、Feishu/Lark、WeCom、WeChat、QQ Bot、Yuanbao，以及
API server 和 webhook 端点）及其实时连接状态。

- **Configure** —— 打开按平台定制的表单，只包含该渠道需要的字段（bot token、app token、服务器 URL、允许列表等）。机密渲染为密码输入框并以打码形式保存；留空字段会保留现有值。必填字段会被标记并校验。"Setup guide" 链接指向该平台的凭据文档。
- **Enable / disable** —— 启用或停用渠道。凭据仍保留在磁盘上，只是活跃状态改变。
- **Test** —— 检查该渠道是否已配置、已启用，并报告来自 gateway 的实时连接。
- **Restart gateway** —— 凭据写入 `~/.hermes/.env`，启用标志写入 `config.yaml`；gateway 会在下一次重启时连接每个已启用的渠道，你可以直接从页面触发该重启。在系统级安装（`hermes gateway install --system`）上，dashboard 以 `sudo -n` 执行重启，因此 dashboard 用户需要免密 sudo；没有它，请求会立即失败，而不是报告一个随后被 CLI 拒绝的重启。

![Channels 管理页 —— 每个消息平台的状态、启用开关和按平台配置表单](/img/dashboard/admin-channels.png)

### System

面向整机安装操作的统一管理面板：

- **Host** —— 实时系统统计：OS / 内核、架构、主机名、Python 与 Hermes 版本、CPU 核心数 + 利用率、内存、Hermes home 的磁盘占用、运行时长和平均负载。（安装 `psutil` 时 CPU/内存/磁盘来自它；身份字段始终显示。）Hermes 版本会显示**更新状态徽章**（已是最新 / 落后 N 个提交）和一个 **Check for updates** 按钮。git 安装有可用更新时，**Update now** 按钮会先打开确认对话框 —— 显示你将拉取多少提交 —— 然后在后台运行 `hermes update`。在 Docker/Nix 安装上，dashboard 无法就地应用更新，因此会改为显示正确的带外命令。
- **Nous Portal** —— 登录状态、当前推理提供商，以及 Tool Gateway 路由表（哪些工具经由 Portal、哪些本地运行），并附管理订阅的链接。`hermes portal` 的只读镜像。
- **Skill curator** —— 后台技能维护状态（活跃 / 已暂停、间隔、上次运行），带暂停/恢复和立即运行按钮。镜像 `hermes curator`。
- **Gateway** —— 启动、停止和重启消息 gateway，并显示实时状态（运行中/已停止、PID、状态）
- **Memory** —— 选择外部内存提供商（或仅内置），并重置内置的 `MEMORY.md` / `USER.md` 存储
- **Credential pool** —— 添加和移除 agent 轮换使用的 API 密钥（按提供商）。密钥在列表中被 打码；原始值只会到达 agent。
- **Operations** —— 运行 `doctor`、安全审计、创建备份、从备份归档恢复、更新技能、显示系统提示体积明细、生成支持转储，或为退役设置迁移配置。每项都会派发一个后台操作，其实时日志会流式显示在页面中。
- **Checkpoints** —— 查看 `/rollback` 影子存储的大小并修剪它
- **Shell hooks** —— 列出已配置的钩子及其同意状态 + 可执行状态，**创建**钩子（事件、命令、匹配器、超时，并附主动授予的同意），以及移除钩子。钩子会运行任意命令，因此创建表单带有安全警告，且钩子只有在授予同意后才会触发。

![System 管理页 —— 主机统计与 Nous Portal 状态](/img/dashboard/admin-system-top.png)

![System 管理页 —— 技能维护、gateway、内存和凭据池](/img/dashboard/admin-system-curator.png)

![System 管理页 —— 操作、检查点和 shell 钩子](/img/dashboard/admin-system-ops.png)

创建 shell 钩子（注意同意复选框和运行任意命令的警告）：

![新建 shell 钩子对话框](/img/dashboard/admin-hook-create.png)

:::warning 安全
Web Dashboard 会读写包含 API 密钥和机密的 `.env` 文件。它默认绑定到 `127.0.0.1` —— 只能从本机访问，无需登录。绑定到任何非回环地址（包括 `0.0.0.0`）都会启用[认证门禁](#authentication-gated-mode)：在配置认证提供商（用户名/密码或 OAuth）之前，服务器会拒绝启动。
:::


## `/reload` 斜杠命令

Dashboard 还为交互式 CLI 添加了 `/reload` 斜杠命令。通过 Web Dashboard（或直接编辑 `.env`）更改 API 密钥后，在活跃的 CLI 会话中使用 `/reload` 即可获取更改，无需重启：

```
You → /reload
  Reloaded .env (3 var(s) updated)
```

这会将 `~/.hermes/.env` 重新读取到运行中进程的环境中。当你通过 Dashboard 添加了新的提供商密钥并希望立即使用时非常有用。

## REST API

Web Dashboard 暴露了一个供前端使用的 REST API。你也可以直接调用这些端点进行自动化操作：

### GET /api/status

返回 agent 版本、gateway 状态、平台状态和活跃会话数。

### GET /api/chat/workspaces

新的 Chat 标签页会话可以作为起始目录的目录列表：该 profile 的项目（含文件夹）以及发现的 git 仓库（`root`、`label`、`sessions`、`last_active`），外加 `default_cwd`（未选择任何项时聊天落在哪里）和 `home`。`?scan=1` 会先在主机上重新扫描 `desktop.repo_scan_roots`。与 `/api/pty?cwd=<path>` 配合使用，后者在目录缺失时失败即关闭。

### GET /api/sessions

返回最近 20 个会话的元数据（模型、token 数、时间戳、预览）。

### GET /api/config

以 JSON 格式返回当前 `config.yaml` 内容。

### GET /api/config/defaults

返回默认配置值。

### GET /api/config/schema

返回描述每个配置字段的 schema——类型、描述、类别，以及适用时的选项。前端使用此 schema 为每个字段渲染正确的输入控件。

### PUT /api/config

保存新配置。请求体：`{"config": {...}}`。

### GET /api/env

返回所有已知环境变量，包含其设置/未设置状态、脱敏值、描述和类别。

### PUT /api/env

设置环境变量。请求体：`{"key": "VAR_NAME", "value": "secret"}`。

### DELETE /api/env

删除环境变量。请求体：`{"key": "VAR_NAME"}`。

### GET /api/sessions/\{session_id\}

返回单个会话的元数据。

### GET /api/sessions/\{session_id\}/messages

返回有上限的会话消息页，包含工具调用和时间戳。默认按时间升序返回最近 500 条；可用 `limit`（最大 500）、`offset` 和 `order=oldest|latest` 显式分页。

### GET /api/sessions/search

对消息内容进行全文搜索。查询参数：`q`。返回匹配的会话 ID 和高亮片段。

### DELETE /api/sessions/\{session_id\}

删除会话及其消息历史。

### 当 gateway 拥有存储时进行批量维护

`POST /api/sessions/bulk-delete`、`DELETE /api/sessions/empty`、`POST /api/sessions/prune`
（非 dry-run）和 `POST /api/sessions/owner-backfill` 会一次重写许多行，
并绕过按会话的修订围栏，因此它们是**离线维护**：会获取该 profile 的
排他 `gateway.lock`。当该 profile 的 gateway 正在运行时，它们会返回
`409` 并说明 `Exclusive maintenance refused: Gateway runtime already owns profile <home>.
Drain and stop the gateway, then retry.`，且不做任何更改。请停止 gateway
（`hermes gateway stop`），运行维护调用，然后再启动它。`dry_run` 的 prune
和所有读取端点可对运行中的 gateway 继续工作。Desktop 的一次性旧版
owner 回填会把该 `409` 视为已连接 gateway 的稳态（就像没有该路由的后端），
并且不会在每次侧边栏刷新时重试。

### GET /api/logs

返回日志行。查询参数：`file`（agent/errors/gateway）、`lines`（数量）、`level`、`component`。

### GET /api/analytics/usage

返回 token 用量、成本和会话分析。查询参数：`days`（默认 30）。响应包含每日明细和按模型聚合数据。

### GET /api/cron/jobs

返回所有已配置的定时任务，包含其状态、计划和运行历史。

### POST /api/cron/jobs

创建新定时任务。请求体：`{"prompt": "...", "schedule": "0 9 * * *", "name": "...", "deliver": "local"}`。

### POST /api/cron/jobs/\{job_id\}/pause

暂停定时任务。

### POST /api/cron/jobs/\{job_id\}/resume

恢复已暂停的定时任务。

### POST /api/cron/jobs/\{job_id\}/trigger

在计划之外立即触发定时任务。

### DELETE /api/cron/jobs/\{job_id\}

删除定时任务。

### GET /api/skills

返回所有技能，包含其名称、描述、类别和启用状态。

### PUT /api/skills/toggle

启用或禁用技能。请求体：`{"name": "skill-name", "enabled": true}`。

### GET /api/tools/toolsets

返回所有工具集，包含其标签、描述、工具列表以及活跃/已配置状态。

### 管理端点 {#admin-endpoints}

这些端点驱动 MCP、Channels、Webhooks、Pairing 和 System 页面。它们都与 `/api/` 的其余部分处于同一认证门禁之后。

| 方法与路径 | 用途 |
|---------------|---------|
| `GET /api/mcp/servers` | 列出已配置的 MCP 服务器（环境变量值打码） |
| `POST /api/mcp/servers` | 添加服务器。Body：`{name, url?, command?, args?, env?, auth?}` |
| `POST /api/mcp/servers/{name}/test` | 连接、列出工具、断开 |
| `PUT /api/mcp/servers/{name}/enabled` | 启用 / 停用服务器 |
| `DELETE /api/mcp/servers/{name}` | 移除服务器 |
| `GET /api/mcp/catalog` | 浏览 Nous 批准的 MCP 目录 |
| `POST /api/mcp/catalog/install` | 安装目录条目（包含所需 env） |
| `GET /api/messaging/platforms` | 列出所有消息渠道及其状态 + 按平台设置字段 |
| `PUT /api/messaging/platforms/{id}` | 配置渠道。Body：`{enabled?, env?, clear_env?}`（env 写入 `.env`，enabled 写入 `config.yaml`） |
| `POST /api/messaging/platforms/{id}/test` | 报告渠道是否已配置、已启用并已连接 |
| `GET /api/pairing` | 列出待处理 + 已批准的消息用户 |
| `POST /api/pairing/approve` | 批准配对码。Body：`{platform, code}` |
| `POST /api/pairing/revoke` | 撤销用户。Body：`{platform, user_id}` |
| `POST /api/pairing/clear-pending` | 丢弃所有待处理配对码 |
| `GET /api/webhooks` | 列出订阅 + 平台启用状态 |
| `POST /api/webhooks` | 创建订阅（返回一次性机密） |
| `DELETE /api/webhooks/{name}` | 移除订阅 |
| `GET /api/credentials/pool` | 列出轮换池中的密钥（打码） |
| `POST /api/credentials/pool` | 添加密钥。Body：`{provider, api_key, label?}` |
| `DELETE /api/credentials/pool/{provider}/{index}` | 移除密钥（从 1 开始的索引） |
| `GET /api/memory` | 活跃提供商 + 可用提供商 + 内置文件大小 |
| `PUT /api/memory/provider` | 选择提供商（空 = 仅内置） |
| `POST /api/memory/reset` | 重置内置内存。Body：`{target: all\|memory\|user}` |
| `POST /api/gateway/start` · `/stop` · `/restart` | gateway 生命周期（后台执行） |
| `POST /api/ops/doctor` · `/security-audit` · `/backup` · `/import` | 诊断与维护（后台执行；通过 `/api/actions/{name}/status` 跟踪） |
| `GET /api/ops/hooks` | 已配置的 shell 钩子 + 允许列表状态 |
| `GET /api/ops/checkpoints` · `POST .../prune` | 查看 / 修剪 `/rollback` 存储 |
| `POST /api/ops/hooks` · `DELETE /api/ops/hooks` | 创建 / 移除 shell 钩子（需授予同意） |
| `GET /api/system/stats` | 主机统计 —— OS、CPU、内存、磁盘、运行时长 |
| `GET /api/hermes/update/check` | 报告更新可用性（落后提交数、安装方式）但不应用。对落后的 git 安装，还会返回变更的 `commits` 列表（`sha`、`summary`、`author`、`at`）。`?force=1` 会绕过 24 小时缓存（检查走 GitHub API，从不 `git fetch`） |
| `GET /api/curator` · `PUT .../paused` · `POST .../run` | 技能维护状态 + 暂停/恢复 + 运行 |
| `GET /api/portal` | Nous Portal 认证 + Tool Gateway 路由（只读） |
| `POST /api/ops/prompt-size` · `/dump` · `/config-migrate` | 诊断（后台执行） |
| `PUT /api/webhooks/{name}/enabled` | 启用 / 停用 webhook 路由 |
| `POST /api/skills/hub/install` · `/uninstall` · `/update` | 技能 hub 操作（后台执行） |
| `GET /api/skills/hub/search` | 跨所有来源搜索技能 hub |
| `GET /api/sessions/stats` | 会话存储统计 |
| `PATCH /api/sessions/{id}` | 重命名 / 归档会话 |
| `GET /api/sessions/{id}/export` | 将会话（元数据 + 消息）导出为 JSON |
| `POST /api/sessions/prune` | 删除早于 N 天的已结束会话 |
| `PUT /api/cron/jobs/{id}` | 编辑 cron 任务的 prompt / 计划 / 名称 / 投递 |

## 身份验证（门控模式） {#authentication-gated-mode}

当 Dashboard 绑定到公网地址或非回环地址 —— 即 `127.0.0.1` / `localhost` 之外的任何地址 —— 时，Hermes Agent 会启用认证门禁。每个请求都必须携带经过验证的会话 cookie，否则会被退回登录页。内置三种提供商：

- **[用户名/密码](#usernamepassword-provider-no-oauth-idp)** —— 为自托管 / 本地部署 / homelab 的 Dashboard 加上认证的最简单方式。不需要外部身份提供商。**仅在可信网络或 VPN 之后使用 —— 不要用于直接暴露到公网。**
- **[OAuth（Nous Portal）](#default-provider-nous-research)** —— 适用于托管部署，以及任何可通过公网访问的 Dashboard，也是[远程 Hermes Desktop 连接](#connecting-hermes-desktop-to-a-remote-backend)推荐的路径。每次登录都会针对你的 Nous 账户验证，因此这是适合对公网开放的提供商。
- **[自托管 OIDC](#self-hosted-oidc-provider)** —— 通过标准 OpenID Connect 接入你自己的身份提供商（Keycloak、Auth0、Okta、Google、经 OIDC 桥接的 GitHub 等）。不涉及 Nous Portal；在符合规范的 OIDC 服务器前面时，适合对公网开放。

运营方自有、绑定到回环地址的 Dashboard 不受影响 —— 无需认证，也没有登录页。

### 门禁何时启用

| 参数 | 认证门禁 | 使用场景 |
|-------|-----------|----------|
| `hermes dashboard`（默认 —— 绑定 `127.0.0.1`） | 关闭 | 本地开发 |
| `hermes dashboard --host 0.0.0.0` | **开启** | 远程 / 生产 —— 用用户名/密码提供商或 OAuth 保护 |

当且仅当绑定主机不是 `127.0.0.1`、`::1` 或 `localhost` 时，门禁才会开启。绑定到 `0.0.0.0`（或任何 RFC1918 / LAN 地址）都会启门禁。遗留的 `--insecure` 参数**不再能关闭它** —— 它仅为向后兼容而接受，会被忽略并给出警告。

:::danger `--insecure` 是空操作 —— 它不会禁用认证
自 2026 年 6 月的加固之后，`--insecure` 不再绕过 Dashboard 认证：非回环绑定始终需要一个认证提供商（用户名/密码提供商或 OAuth）。如果你想要无认证的 Dashboard，请绑定到 `127.0.0.1`，并通过 SSH 隧道或 Tailscale 访问。
:::

### 失败即关闭（fail-closed）语义

如果门禁会启用，但**没有**注册任何 `DashboardAuthProvider`（既没有 Nous 插件，也没有自定义插件），`hermes dashboard` 会拒绝绑定并给出明确的错误信息。不存在“默认拒绝但接受一切”的回退 —— 配置错误的门控 Dashboard 永远不会启动。

当你**交互式**地（真实终端）运行 `hermes dashboard --host 0.0.0.0` 且尚未配置提供商时，Hermes 不只是失败 —— 它会当场提议设置一个：选择**用户名与密码**（把 `dashboard.basic_auth` 写入 `config.yaml`，几秒内即可运行）或 **OAuth**（引导你执行 `hermes dashboard register`）。非交互式调用方 —— Docker/s6、CI、管道运行 —— 会跳过该提示，直接遇到上面的失败即关闭错误，因此无人值守的部署在没有认证时同样永远不会启动。

### 默认提供商：Nous Research {#default-provider-nous-research}

内置的 `plugins/dashboard_auth/nous` 插件**始终安装**并自动加载。当配置了 client ID 时，它会自动注册一个名为 `nous` 的 `DashboardAuthProvider`。

由于每次登录都会针对 Nous Portal 验证并受你的 Nous 账户保护，**Nous 提供商是适合把 Dashboard 暴露到公网的哪一个。**

#### 注册 Dashboard {#registering-a-dashboard}

要使用 Nous 提供商，你需要一个 OAuth client ID（形如 `agent:{id}`）。获取方式有两种：

- **CLI —— `hermes dashboard register`。** 在 Dashboard 所在的主机上运行它。它会解析你现有的 Nous 登录（如果尚未登录，请先运行 `hermes setup`），向 Portal 注册一个自托管 OAuth 客户端，并把 `HERMES_DASHBOARD_OAUTH_CLIENT_ID` 写进 `~/.hermes/.env`。可选参数：`--name`（人类可读的标签，否则自动生成）和 `--redirect-uri`（面向公网主机的公开 HTTPS 回调 URL）。

  ```bash
  hermes dashboard register
  # ✓ Registered dashboard "swift_falcon"
  # …writes HERMES_DASHBOARD_OAUTH_CLIENT_ID to ~/.hermes/.env
  ```

- **GUI —— Local Dashboards 页面。** 在 Nous Portal 中打开 [`/local-dashboards`](https://portal.nousresearch.com/local-dashboards)，即可从浏览器注册、命名、管理并吊销自托管 Dashboard。把生成的 `agent:{id}` client ID 复制到 `HERMES_DASHBOARD_OAUTH_CLIENT_ID`（环境变量）或 `dashboard.oauth.client_id`（config.yaml）。通过 CLI 注册的 Dashboard 也在这里吊销。

#### 配置

该插件从两个来源读取配置，环境变量在非空时优先：

**`config.yaml`** —— 规范来源：

```yaml
dashboard:
  oauth:
    client_id: agent:01HXYZ…             # required to engage the gate
    token_leeway: 60                     # optional; seconds of clock-skew tolerance
```

**环境变量** —— 运营方覆盖：

| 环境变量 | 覆盖 | 格式 | 由谁写入 |
|---------|-----------|--------|----------------|
| `HERMES_DASHBOARD_OAUTH_CLIENT_ID` | `dashboard.oauth.client_id` | `agent:{instance_id}` | `hermes dashboard register` |

按照 Hermes Agent 的惯例（`~/.hermes/.env` 只放 API 密钥 / 机密），**`config.yaml` 是为本地开发、本地部署以及任何你直接控制的部署设置这些值的推荐位置**。环境变量这条路是为了让托管平台的密钥注入能在每次部署时推送 `client_id`，而不必有人进镜像里修改 `config.yaml` —— 这才是它的主要用途。

空的环境变量值会被视为未设置，因此某个已注入但未填值的平台密钥不会意外遮蔽 `config.yaml` 中的有效条目。

`dashboard.oauth.token_leeway`（仅 config.yaml）是时钟偏差容差 —— 单位为秒 —— 在 JWT 校验时应用于 Portal 访问令牌的 `exp`/`nbf`/`iat` 声明。默认值为 `60`（RFC 7519 §4.1.4-4.1.6），这样时钟落后于 Portal 的主机不会登录失败；`0` 恢复严格校验，而无效值（无法解析 / 负数 / 非有限值）会失败即关闭地回落为 `0`。自托管 OIDC 提供商有等效的 `dashboard.oauth.self_hosted.id_token_leeway` 旋钮。

如果两个来源都没有提供 client_id，插件会报告具体原因，而 Dashboard 的失败即关闭绑定错误会准确告诉你该修什么：

```
Refusing to bind dashboard to 0.0.0.0 — the auth gate engages on
non-loopback binds, but no auth providers are registered.

Bundled providers reported these issues:
  • nous: HERMES_DASHBOARD_OAUTH_CLIENT_ID is not set (and
    dashboard.oauth.client_id in config.yaml is empty). …

Configure an auth provider before exposing the dashboard:
  • Password: set dashboard.basic_auth.username + password_hash in config.yaml
  • OAuth: run `hermes dashboard register` (Nous Portal) or install a
    DashboardAuthProvider plugin.
There is no unauthenticated public-bind option — to keep it local, bind
127.0.0.1 and tunnel in (SSH / Tailscale).
```

#### 完整示例：Nous Research

从已登录的 Hermes 安装到受 Nous 门控的 Dashboard，共三步。

**1. 登录并注册 Dashboard。** `hermes dashboard register` 使用你现有的 Nous 登录来配置一个 OAuth 客户端，并把 `HERMES_DASHBOARD_OAUTH_CLIENT_ID` 写入 `~/.hermes/.env`：

```bash
hermes setup            # if you're not already logged into Nous Portal
hermes dashboard register
# ✓ Registered dashboard "swift_falcon"
# …writes HERMES_DASHBOARD_OAUTH_CLIENT_ID to ~/.hermes/.env
```

**2. 在可达地址上运行 Dashboard。** 非回环绑定会启用 OAuth 门禁，刚刚写入的 `client_id` 会激活 `nous` 提供商：

```bash
hermes dashboard --host 0.0.0.0 --port 9119 --no-open
```

**3. 登录。** 打开 `http://<host>:9119/`，你会被跳到 `/login`。点击 **Sign in with Nous Research** → 在 Portal 完成认证 → 回到已认证的 Dashboard。可从任意机器验证门禁：

```bash
curl -s http://<host>:9119/api/status | jq '.auth_required, .auth_providers'
# true
# ["nous"]
```

随后 `GET /api/auth/me` 会返回已验证的会话（`provider: nous`）。对于面向公网的主机，请用 `--redirect-uri https://hermes.example.com/auth/callback` 注册并设置 `HERMES_DASHBOARD_PUBLIC_URL`，使 OAuth 回调解析到你的公开 URL（参见[公共 URL 覆盖](#public-url-override)）。

### 用户名/密码提供商（无 OAuth IDP） {#usernamepassword-provider-no-oauth-idp}

如果你不想接入 OAuth 身份提供商 —— 一种自托管的“只给我的 Dashboard 加个密码”式部署 —— 内置的 `plugins/dashboard_auth/basic` 插件会注册一个名为 `basic` 的 `DashboardAuthProvider`，它用**用户名和密码**认证，而不是 OAuth 重定向。

它接入与 OAuth 提供商相同的门禁：门禁在非回环绑定时启用，登录页会为该提供商渲染一个凭据表单（而不是“使用 X 登录”按钮），而登录之后的一切 —— 会话 cookie、透明刷新、WS ticket、注销、审计日志 —— 都与 OAuth 路径相同。会话是该提供商自己签发的无状态 HMAC 签名令牌，因此**没有数据库，也没有外部 IDP**。密码哈希使用标准库 `scrypt`（无第三方依赖）。

:::warning 仅在可信网络使用 —— 不要用于公网
用户名/密码提供商面向**可信网络**上的自托管 / 本地部署 / homelab Dashboard，或仅能通过 **VPN** 访问的场景。它保护的是单个共享凭据，背后没有外部身份提供商、MFA 或按用户区分的账户，因此**不适合**把 Dashboard 直接暴露到公网。对于面向公网的 Dashboard，请改用 [Nous Research 提供商](#default-provider-nous-research)（或你自己的[自托管 OIDC](#self-hosted-oidc-provider) / [自定义 OAuth](#custom-providers) 提供商）。
:::

#### 配置

与 Nous 提供商一样，它从 `config.yaml`（规范来源）读取，环境变量在非空时优先。只有当配置了 `username` 加上 `password_hash`（首选）或 `password` 之一时它才会激活 —— 否则它是空操作，因此 OAuth 用户和回环运营方不受影响。

**`config.yaml`：**

```yaml
dashboard:
  basic_auth:
    username: admin
    # Preferred — no plaintext at rest. Compute with:
    #   python -c "from plugins.dashboard_auth.basic import hash_password; print(hash_password('PW'))"
    password_hash: "scrypt$16384$8$1$…$…"
    # ...or a plaintext password (hashed in-memory at load; less safe at rest):
    # password: "s3cret"
    secret: "<32+ random bytes, base64 or hex>"  # token-signing key
    session_ttl_seconds: 43200                    # optional; access-token lifetime (default 12h)
```

**环境变量覆盖：**

| 环境变量 | 覆盖 | 说明 |
|---------|-----------|-------|
| `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` | `dashboard.basic_auth.username` | 激活所必需 |
| `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH` | `dashboard.basic_auth.password_hash` | 首选（不落明文） |
| `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD` | `dashboard.basic_auth.password` | 明文；**优先于配置中的 `password_hash`**，便于通过环境变量轮换 |
| `HERMES_DASHBOARD_BASIC_AUTH_SECRET` | `dashboard.basic_auth.secret` | 令牌签名密钥 |
| `HERMES_DASHBOARD_BASIC_AUTH_TTL_SECONDS` | `dashboard.basic_auth.session_ttl_seconds` | 访问令牌有效期 |

:::caution 为稳定的会话显式设置 `secret`
`secret` 为空时，会为每个进程生成随机签名密钥。对单个进程这没问题，但这意味着**每次重启都会使所有会话失效**，会话也**无法跨多个 worker**。对于需要重启后存活 / 多 worker 的部署，请显式设置 `secret`。
:::

`/auth/password-login` 端点按客户端 IP 限流（默认每分钟 10 次 → HTTP 429），并且对未知用户和错误密码都返回同一个通用 `401 Invalid credentials`，因此无法被用作用户名枚举的判定。

#### 完整示例：用户名/密码

从零到可信网络上的密码门控 Dashboard，共三步。

**1. 在 `~/.hermes/.env` 中设置凭据。** 对密码做哈希，避免明文落盘，并设置一个稳定的签名密钥，使会话在重启后仍然有效：

```bash
# Compute a scrypt hash of your chosen password:
HASH=$(python -c "from plugins.dashboard_auth.basic import hash_password; print(hash_password('choose-a-strong-password'))")

cat >> ~/.hermes/.env <<EOF
HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH=$HASH
HERMES_DASHBOARD_BASIC_AUTH_SECRET=$(openssl rand -base64 32)
EOF
chmod 600 ~/.hermes/.env
```

**2. 在可达地址上运行 Dashboard。** 非回环绑定会启用门禁，用户名加哈希会激活 `basic` 提供商：

```bash
hermes dashboard --host 0.0.0.0 --port 9119 --no-open
```

**3. 登录。** 打开 `http://<host>:9119/`，你会被跳到 `/login` —— 显示的是**凭据表单**（而不是“Sign in with X”按钮）。输入 `admin` / 你的密码 → 进入已认证的 Dashboard。可从任意机器验证门禁：

```bash
curl -s http://<host>:9119/api/status | jq '.auth_required, .auth_providers'
# true
# ["basic"]
```

随后 `GET /api/auth/me` 会返回已验证的会话（`provider: basic`）。请把它放在 VPN 之后 —— 见上面的警告；对于公网主机，请改用 [Nous Research](#default-provider-nous-research) 或[自托管 OIDC](#self-hosted-oidc-provider) 提供商。

#### 编写你自己的密码提供商

`basic` 只是某个扩展点的一种实现。任何插件都可以注册密码提供商：在你的 `DashboardAuthProvider` 子类上设置 `supports_password = True`，并实现 `complete_password_login(*, username, password) -> Session`（拒绝时抛出 `InvalidCredentialsError`，后端存储不可用时抛出 `ProviderError`）。纯密码提供商可以把 OAuth 的 `start_login` / `complete_login` 方法留作 `NotImplementedError` 桩。这是 LDAP 绑定、凭据数据库或任何其他非重定向认证方案的路径 —— 框架会替你处理表单、路由、cookie 和刷新。

### 部署会话令牌提供商（仅令牌部署） {#deployment-session-token-provider}

内置的 `plugins/dashboard_auth/session_token` 插件让**部署会话令牌**（`HERMES_DASHBOARD_SESSION_TOKEN`）成为门控模式的一等凭据。这正是密钥托管部署已经在提供的形式 —— 一个共享令牌（nix/agenix 以解密后的文件交给它；Nix 模块根据 `backend.sessionTokenFile` 设置 `HERMES_DASHBOARD_SESSION_TOKEN=file:<path>`）——由 Hermes Desktop 远程 gateway 的 **“Session token”** 字段提交。没有这个提供商时，非回环绑定只配了一个已提供的令牌，就会撞上失败即关闭的门禁，把运营方推向部署从未要求的 basic-auth 或 OAuth 凭据。

环境变量被提供时，该提供商会自动注册，`verify_session` 会以常量时间比较提交的凭据与它。令牌模式客户端使用的两条腿都走这个接缝 —— REST `Authorization:` 路径和 `/api/ws` 的 `?token=` 升级凭据 —— 因此一个已提供的令牌即可同时授权两者。这里**没有浏览器登录流程**：`/login` 会渲染一段说明该凭据渠道的提示，而不是登录按钮，自动 SSO 重定向也永远不会提供该提供商。

:::warning 把令牌当作密码对待
已提供的令牌会赋予持有者完整的 Dashboard 访问权限。请从密钥管理器提供它（绝不要放在 Nix store 路径），通过重新部署密钥来轮换 —— 轮换会使每个已保存的客户端副本失效，表现为 Desktop 中“HTTP 可达但 WebSocket 失败”，直到粘贴新令牌为止。它背后没有按用户区分的身份：对于共享或面向公网的部署，优先使用 [Nous Research](#default-provider-nous-research) 或[自托管 OIDC](#self-hosted-oidc-provider) 提供商。
:::

**激活与退出。** 注册由凭据本身驱动：在设置 `HERMES_DASHBOARD_SESSION_TOKEN` 之前该提供商是空操作（环境变量优先；`file:<path>` 在启动时读取，缺失/为空时会明确报错）。若要让令牌仅限内部使用（回环 SPA 使用、Desktop 启动的后端），而改用 basic-auth/OAuth 认证门禁，请禁用该提供商：

```yaml
dashboard:
  session_token_auth:
    enabled: false
```

可从任意机器验证令牌门控的部署：

```bash
curl -s http://<host>:9119/api/status | jq '.auth_required, .auth_providers'
# true
# ["session-token"]

curl -s -o /dev/null -w '%{http_code}\n' \
  -H "Authorization: Bearer ***" http://<host>:9119/api/auth/me
# 200
```

### 自托管 OIDC 提供商 {#self-hosted-oidc-provider}

如果你运行自己的身份提供商，内置的 `plugins/dashboard_auth/self_hosted` 插件会用**标准 OpenID Connect** 针对它认证 Dashboard —— 无需按 IDP 写代码，也不涉及 Nous Portal。它已针对任何符合规范的 OIDC 服务器验证过，并可与之配合工作：

> **Authentik · Keycloak · Zitadel · Authelia · Auth0 · Okta · Google · …**

与 Nous 提供商一样，它自动加载，并且只会在配置好之后注册自身，因此对回环 Dashboard 是空操作。

#### 配置

配置一个 **issuer** 和一个 **client_id**（公开 PKCE 客户端 —— 无 client secret）。插件会从 `{issuer}/.well-known/openid-configuration` 获取 IDP 的 `authorization_endpoint`、`token_endpoint` 和 `jwks_uri`，因此你永远不必硬编码端点 URL。

**`config.yaml`** —— 规范来源：

```yaml
dashboard:
  oauth:
    provider: self-hosted
    self_hosted:
      issuer: https://auth.example.com/application/o/hermes/   # required
      client_id: hermes-dashboard                              # required
      scopes: "openid profile email"                           # optional (this is the default)
      id_token_leeway: 60                                      # optional; seconds of clock-skew tolerance
```

**环境变量** —— 运营方覆盖（非空时环境变量优先于 `config.yaml`；空值视为未设置）：

| 环境变量 | 覆盖 | 说明 |
|---------|-----------|-------|
| `HERMES_DASHBOARD_OIDC_ISSUER` | `dashboard.oauth.self_hosted.issuer` | OIDC issuer URL —— 必需 |
| `HERMES_DASHBOARD_OIDC_CLIENT_ID` | `dashboard.oauth.self_hosted.client_id` | 公开 client id —— 必需 |
| `HERMES_DASHBOARD_OIDC_SCOPES` | `dashboard.oauth.self_hosted.scopes` | 默认 `openid profile email` |

在你的 IDP 中注册一个**公开**应用/客户端，使用授权码 + PKCE（S256）授权类型，并把 Dashboard 的回调添加为允许的重定向 URI。回调是 `<dashboard public URL>/auth/callback`（关于 Dashboard 在代理后面如何推导自己的公开 URL，参见[公共 URL 覆盖](#public-url-override)）。

#### 它校验什么

每个能登录的账户都代表你 —— 机器所有者 —— 行事。使用自托管登录提供商时，请在提供商中把应用限制为你自己的账户（例如 Okta 的应用分配、Microsoft Entra 的 “Assignment required”，或限定给你的 Google Workspace 应用）；否则，凡是你的提供商放行的人都会获得完整访问权限，与以前一样。

该提供商针对发现的 `jwks_uri` 校验 OpenID Connect **ID token**（RS256/ES256），并把 `iss` 和 `aud` 声明固定为你配置的 `issuer` 和 `client_id`。时间声明（`exp`/`nbf`/`iat`）以 60 秒的时钟偏差容差校验（RFC 7519 §4.1.4-4.1.6），因此时钟落后于 IDP 的 Dashboard 主机不会登录失败；可在 `config.yaml` 中用 `dashboard.oauth.self_hosted.id_token_leeway` 调整（低于 `60` 的任何值都会收紧它，`0` 恢复严格校验，无效值会失败即关闭地回落为 `0`）。标准 OIDC 声明会映射到 Dashboard 会话：

| 会话字段 | 声明 |
|---------------|----------|
| `user_id` | `sub`（必需） |
| `email` | `email` |
| `display_name` | `name` → `preferred_username` → `nickname` → `email` |
| `org_id` | `org_id` / `organization`，否则用拼接的 `groups` |

建立身份的是 ID token —— 访问令牌被视为不透明（OIDC 规范并不要求它是 JWT）。端点 URL 必须是 HTTPS（回环 `http://` 允许用于本地开发 IDP），并且发现文档声明的 `issuer` 必须与你配置的一致（尾随斜杠的差异可以容忍）。当 IDP 签发刷新令牌时，会用标准 `refresh_token` 授权进行静默重新认证；注销时会调用 IDP 的 RFC 7009 `revocation_endpoint`（若已声明）。

> **机密客户端**（带 `client_secret` 的）暂不支持 —— 请配置公开 + PKCE 客户端，这也是面向浏览器的 Dashboard 的典型选择。

#### 完整示例：Keycloak

[Keycloak](https://www.keycloak.org/) 是最容易为本地测试搭起来的自托管 OIDC 服务器之一 —— 它以开发模式（内存数据库）作为单个容器运行，并提供教科书式的 OIDC 发现。本演练能在几分钟内让你从零搭到可用的 Dashboard 登录。

**1. 用预配置的 realm 运行 Keycloak。** 把这个 realm 导出保存为 `realm-hermes.json` —— 它定义了一个 `hermes` realm、一个**公开 PKCE 客户端**（`hermes-dashboard`）和一个测试用户，启动时全部导入，因此无需在管理界面里点任何东西：

```json
{
  "realm": "hermes",
  "enabled": true,
  "clients": [
    {
      "clientId": "hermes-dashboard",
      "name": "Hermes Agent Dashboard",
      "enabled": true,
      "publicClient": true,
      "standardFlowEnabled": true,
      "protocol": "openid-connect",
      "redirectUris": ["http://localhost:9119/auth/callback"],
      "webOrigins": ["http://localhost:9119"],
      "attributes": { "pkce.code.challenge.method": "S256" }
    }
  ],
  "users": [
    {
      "username": "testuser",
      "enabled": true,
      "emailVerified": true,
      "email": "testuser@example.com",
      "firstName": "Test",
      "lastName": "User",
      "credentials": [
        { "type": "password", "value": "testpassword", "temporary": false }
      ]
    }
  ]
}
```

启动它（Keycloak 26+），把该文件挂载到导入目录：

```bash
docker run --rm -p 8080:8080 \
  -e KC_BOOTSTRAP_ADMIN_USERNAME=admin \
  -e KC_BOOTSTRAP_ADMIN_PASSWORD=admin \
  -v "$PWD/realm-hermes.json:/opt/keycloak/data/import/realm-hermes.json:ro" \
  quay.io/keycloak/keycloak:26.0 \
  start-dev --import-realm
```

启动后，该 realm 会在
`http://localhost:8080/realms/hermes/.well-known/openid-configuration` 声明标准 OIDC 发现信息（issuer 为
`http://localhost:8080/realms/hermes`）。管理控制台位于
`http://localhost:8080/`（`admin` / `admin`）。

**2. 让 Dashboard 指向它。** 自托管插件允许回环 `http://` issuer（任何非回环 issuer 都要求 HTTPS），因此本地 Keycloak 可以直接使用：

```bash
export HERMES_DASHBOARD_OIDC_ISSUER="http://localhost:8080/realms/hermes"
export HERMES_DASHBOARD_OIDC_CLIENT_ID="hermes-dashboard"
export HERMES_DASHBOARD_PUBLIC_URL="http://localhost:9119"
hermes dashboard --host 0.0.0.0 --port 9119 --no-open
```

`HERMES_DASHBOARD_PUBLIC_URL` 告诉 Dashboard 它的 OAuth 回调是
`http://localhost:9119/auth/callback` —— 也就是上面 realm 注册的重定向 URI。绑定到
`0.0.0.0`（非回环绑定）正是启用 OAuth 门禁的原因。

**3. 登录。** 打开 `http://localhost:9119/`，你会被跳到 `/login`。点击 **Sign in with Self-Hosted OIDC** → 以 `testuser` / `testpassword` 在 Keycloak 完成认证 → 回到已认证的 Dashboard。侧边栏会显示 `Logged in as Test User via self-hosted`，而 `GET /api/auth/me` 返回已验证的会话（`provider: self-hosted`、`email: testuser@example.com`）。

> 如果你绑定或浏览的是其他主机/端口，请把该来源的
> `…/auth/callback` 添加到 Keycloak 管理控制台中客户端的 **Valid redirect URIs**
> （Clients → hermes-dashboard → Settings）。同样的模式也适用于
> Authentik、Zitadel、Authelia 以及其他 OIDC 服务器 —— 只有 issuer
> URL 和客户端注册界面不同。

### 公共 URL 覆盖 {#public-url-override}

默认情况下，Dashboard 会从请求重建 OAuth 回调 URL —— `X-Forwarded-Host` + `X-Forwarded-Proto` + `X-Forwarded-Prefix`（前提是 uvicorn 配置了 `proxy_headers=True`，门禁下 `start_server` 会启用它）。在正确设置这三个头的反向代理后面，这一点开箱即用。

对于反向代理不能可靠转发这些头的部署（手工 nginx 配置、本地 ingress、带部分代理链的自定义域名部署），请把 `dashboard.public_url`（或 `HERMES_DASHBOARD_PUBLIC_URL`）设置为 Dashboard 实际访问所用的**完整公开 URL**：

```yaml
dashboard:
  public_url: "https://dashboard.example.com/hermes"
  trusted_proxies:
    - "172.20.0.5"
```

设置后，OAuth 回调 URL 就逐字变成 `<public_url>/auth/callback` —— 在这条代码路径上 `X-Forwarded-Prefix` 会被忽略，因为运营方已明确声明了公开 URL。这是有意为之：在其上再叠加前缀，会让常见情况下（前缀已经写进 `public_url`）出现双前缀。

`public_url` 中的主机名也会被接受为**精确匹配**的 HTTP `Host` 和
WebSocket `Origin` 值。这支持那种保留浏览器侧主机名、同时转发到绑定在
`127.0.0.1` 上的 Dashboard 的反向代理。不允许通配符和前缀匹配，因此像
`dashboard.example.com.evil.test` 这样的攻击者主机仍会被 DNS 重绑定
防护拒绝。

声明非回环的 `public_url` 总会启用 Dashboard 认证门禁，
即使后端绑定到回环地址也一样。请先配置密码或 OAuth 提供商；没有的话，Hermes 会在启动时
失败即关闭。这可以防止本地 SPA 会话令牌通过代理变成远程认证机制。
这种模式下 uvicorn 也会启用代理头处理。回环
代理会被自动信任。如果 TLS 终结器从另一个
容器或主机连接，请把其精确 IP 地址加入 `dashboard.trusted_proxies`，或者在地址
动态时，为专用代理网络添加一个有界 CIDR：

```yaml
dashboard:
  public_url: "https://dashboard.example.com/hermes"
  trusted_proxies:
    - "172.20.0.0/24"
```

只有列出的对端才能提供 `X-Forwarded-Proto` 和 `X-Forwarded-For`。
Hermes 始终保留回环信任，并拒绝 `*`、`0.0.0.0/0` 和
`::/0`。信任一个网络意味着该网络上每个容器或机器都能提供
转发元数据，因此优先使用精确的代理 IP 或专用的
仅代理网络。没有可信代理条目时，该代理后面的客户端
会共享它的密码登录限流和原生登录上限，认证审计
事件记录的也是代理的地址。

```bash
# Backend remains reachable only on this machine.
hermes dashboard --host 127.0.0.1 --port 9119 --no-open
```

把 TLS 反向代理指向 `http://127.0.0.1:9119`，并在
`dashboard.public_url` 中使用相同的外部来源。

Tailscale Serve 是这种部署形态的一个例子：它可以在
`https://<machine>.<tailnet>.ts.net` 主机名上终结仅 tailnet 的 HTTPS，同时
代理到回环 Dashboard。请把那个确切的 HTTPS 来源用作
`dashboard.public_url`。它仍会被当作非回环的浏览器侧
来源，因此需要一个 Dashboard 认证提供商；这并不要求
让服务可从公网访问。

与其他 Dashboard 设置相同的优先级 —— 环境变量优先于 `config.yaml`：

| 途径 | 覆盖路径 | 何时使用 |
|---------|---------------|-------------|
| `config.yaml` 中的 `dashboard.public_url` | `HERMES_DASHBOARD_PUBLIC_URL` | 本地开发 / 本地部署（规范） |
| `HERMES_DASHBOARD_PUBLIC_URL` 环境变量 | — | 托管平台密钥 / CI |
| （未设置） | — | 默认 —— 从 `X-Forwarded-*` 头重建 |

校验会拒绝没有 `http://` / `https://` scheme、没有主机，或包含引号 / 尖括号 / 空白 / 控制字符的值。格式错误的值会静默回落到按头重建，使登录流程继续工作，而不是把用户送到恶意 URL。

> **注意：** `public_url` 只覆盖 OAuth 回调 URL。`Secure` cookie 标志仍由 `request.url.scheme` 控制，仅当连接对端是回环或在 `trusted_proxies` 中列出时才使用 `X-Forwarded-Proto`。当代理不在回环上时，请为 HTTPS 的 `public_url` 配上 TLS 终结和一个有界的可信代理条目。

### OAuth 流程

该提供商实现了 [Nous Portal OAuth contract v1](https://github.com/NousResearch/nous-account-service/blob/main/docs/agent-dashboard-oauth-contract.md) —— 使用 PKCE（S256）的授权码授权：

1. 用户在没有会话 cookie 的情况下访问 `/` → 门禁重定向到 `/login`。
2. 登录页显示 “Continue with Nous Research” 按钮 → `/auth/login?provider=nous`。
3. 服务器把 PKCE 状态存入一个短期 cookie，并把用户重定向到 `https://portal.nousresearch.com/oauth/authorize?…`。
4. 用户在 Portal 完成认证，落到 `/auth/callback?code=…&state=…`。
5. 服务器在 `POST /api/oauth/token` 用授权码换取访问令牌，针对 Portal 的 JWKS（`/.well-known/jwks.json`）校验 JWT 签名，并设置 `hermes_session_at` cookie。
6. 用户被重定向到 `/`（或通过 `next=` 查询参数回到原来的深链路径）。

访问令牌有效期为 15 分钟。**contract v1 中没有刷新令牌** —— 令牌过期时，SPA 的 fetch 包装会识别 401 响应封装并整页跳回 `/login` 重新走流程。

### 设置的 cookie

| 名称 | 有效期 | 说明 |
|------|----------|-------|
| `hermes_session_at` | 令牌 TTL（15 分钟） | HttpOnly、SameSite=Lax、HTTPS 时 Secure |
| `hermes_session_pkce` | 10 分钟 | HttpOnly；在往返过程中保存 PKCE verifier + 提供商提示。HTTPS 下为 SameSite=None + Secure（必须挺过跨站 IDP 重定向链 —— Chromium 会丢弃跨站链中 302 上设置的 SameSite=Lax cookie）；回环 HTTP 下为 SameSite=Lax |
| `hermes_session_rt` | v1 中未使用 | 为向前兼容保留；`refresh_token` 为空时不写入 |

三者路径都是 `Path=/`。会话 cookie 是 `SameSite=Lax`；PKCE cookie 在 HTTPS 下设置时为 `SameSite=None`（见表格）。当 Dashboard 通过 HTTPS 访问时会设置 `Secure` 标志（通过请求 URL scheme 检测 —— 在 `proxy_headers=True` 下会遵循上游 TLS 终结器的 `X-Forwarded-Proto`）。

### 注销

侧边栏组件显示 `Logged in as <user_id…> via nous` 和一个注销图标。点击它会 POST `/auth/logout`，清除所有 dashboard-auth cookie 并重定向回 `/login`。

### 审计日志

每次登录开始、成功、失败和会话校验失败都会以 JSON 行写入 `$HERMES_HOME/logs/dashboard-auth.log`。敏感字段（`access_token`、`refresh_token`、`code`、`code_verifier`、`state`、`Authorization` 头）在写入日志前会被脱敏。

### 自定义提供商 {#custom-providers}

要接入非 Nous 的 OAuth 提供商（例如 Google、GitHub、自定义 OIDC），请创建一个注册 `DashboardAuthProvider` 的插件：

```python
# ~/.hermes/plugins/dashboard-auth-myidp/__init__.py
from hermes_cli.dashboard_auth import DashboardAuthProvider, Session, LoginStart

class MyIdPProvider(DashboardAuthProvider):
    name = "myidp"
    display_name = "My Identity Provider"

    def start_login(self, *, redirect_uri): ...
    def complete_login(self, *, code, state, code_verifier, redirect_uri): ...
    def verify_session(self, *, access_token): ...
    def refresh_session(self, *, refresh_token): ...
    def revoke_session(self, *, refresh_token): ...

def register(ctx):
    ctx.register_dashboard_auth_provider(MyIdPProvider())
```

登录页会列出所有已注册的提供商；多个提供商可以叠加，用户在 `/login` 选择其一。

### 非交互式（bearer 令牌）认证

除了交互式人工登录（会话 cookie + 刷新）之外，`DashboardAuthProvider` ABC 还通过 `supports_token = True` + `verify_token(token=...)` 支持**非交互式的服务间**能力。当提供商选择加入时，入站的 `Authorization: Bearer` 令牌会被校验，成功后一个 `TokenPrincipal` 会被附加到请求上（`request.state.token_principal`），供该提供商标记为可用令牌认证的端点使用 —— 没有 cookie、没有重定向、没有刷新。

内置的第一个使用者是 **drain** 提供商（`plugins/dashboard_auth/drain`）：`nous-account-service` 通过 `HERMES_DASHBOARD_DRAIN_SECRET` 提供一个按 agent 区分的密钥，该提供商以常量时间比较来校验入站 bearer 令牌，并把 `/api/gateway/drain` 注册为可用令牌认证。它会**失败即关闭** —— 弱/过短的密钥（< 256 位）在注册时就被拒绝，端点保持禁用；环境变量未设置时它是空操作。行为旋钮（`scope`、`min_secret_chars`）位于 `config.yaml` 的 `dashboard.drain_auth` 下。

自定义提供商可以用同样方式实现 `supports_token`/`verify_token`，以暴露自己可被机器认证的端点。

### 验证门禁已启用

```bash
# Quick env-var path.
HERMES_DASHBOARD_OAUTH_CLIENT_ID=agent:test \
  hermes dashboard --host 0.0.0.0

# Or the equivalent via config.yaml (recommended for local dev / on-prem):
#
#   dashboard:
#     oauth:
#       client_id: agent:test
#
# then just:
hermes dashboard --host 0.0.0.0

# Hit /api/status to see the gate state:
curl -s http://127.0.0.1:9119/api/status | jq '.auth_required, .auth_providers'
# true
# ["nous"]
```

Dashboard 的 React StatusPage 会在 “Web server” 下显示相同字段。登录后，侧边栏的 AuthWidget 会显示当前身份。

## 将 Hermes Desktop 连接到远程后端 {#connecting-hermes-desktop-to-a-remote-backend}

Hermes Desktop 可以驱动运行在另一台机器上的 Hermes 后端（VPS、家用服务器、Tailscale 后的 Mini）。在应用中这位于 **Settings → Gateways → Remote gateway**，它会要求一个 **Remote URL** 和一种 **Sign in** 方式。（关于桌面应用本身 —— 安装、设置、聊天 —— 参见 [Hermes Desktop](../desktop.md) 页面。）

你用内置认证提供商之一保护远程 Dashboard，桌面应用会针对后端所声明的提供商进行登录。对于超出你自己机器范围的后端 —— VPS、公网主机、任何面向公网的地址 —— 推荐的提供商是 **OAuth（Nous Portal）**（用 [`hermes dashboard register`](#registering-a-dashboard) 注册，然后用 *Sign in with Nous Research* 登录）。当后端位于可信 LAN 或只能通过 VPN 访问时，内置的[用户名/密码提供商](#usernamepassword-provider-no-oauth-idp)是最快的选择，但**不适合直接暴露到公网**。把 Dashboard 绑定到非回环地址会启用它的认证门禁；登录后，Desktop 会自动复用该会话用于聊天的 WebSocket —— 没有令牌需要复制或粘贴。

下面的配方使用用户名/密码路径，因为它是在可信网络上最快搭起来的；OAuth 路径请参见[默认提供商：Nous Research](#default-provider-nous-research)。

### 在后端（远程机器）上

```bash
# 1. Set the dashboard login credentials in ~/.hermes/.env (secrets file, 0600).
cat >> ~/.hermes/.env <<'EOF'
HERMES_DASHBOARD_BASIC_AUTH_USERNAME=admin
HERMES_DASHBOARD_BASIC_AUTH_PASSWORD=choose-a-strong-password
EOF
# Recommended: a stable signing secret so sessions survive restarts.
# This runs openssl, so the secret is unique to you.
echo "HERMES_DASHBOARD_BASIC_AUTH_SECRET=$(openssl rand -base64 32)" >> ~/.hermes/.env
chmod 600 ~/.hermes/.env

# 2. Run the dashboard bound to a reachable address. The non-loopback bind
#    engages the auth gate; the username/password provider handles login.
hermes dashboard --no-open --host 0.0.0.0 --port 9119
```

该配方的早期版本把 secret 行写在了带引号的 heredoc 内部，那样存入的是文本 `$(openssl rand -base64 32)` 而不是随机值。如果 `grep BASIC_AUTH_SECRET ~/.hermes/.env` 显示的是那段文本，请把该行替换为新生成的密钥并重启 Dashboard。

不想落明文？改用 `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD_HASH` 配一个 scrypt 哈希 —— 完整表面见[用户名/密码提供商](#usernamepassword-provider-no-oauth-idp)。

如果你把 Dashboard 作为 systemd 服务运行，当单元带有 `EnvironmentFile=%h/.hermes/.env` 时，`~/.hermes/.env` 会被自动读取，因此凭据在启动时就在环境中。

:::warning
Dashboard 会读写你的 `.env`（API 密钥、机密）并能运行 agent 命令。这里展示的**用户名/密码**设置适用于可信网络 —— 绝不要把密码保护的 Dashboard 直接暴露到开放互联网。请把它放在 VPN 之后。[Tailscale](https://tailscale.com/) 是干净的选择：绑定到机器的 tailscale IP（`--host <tailscale-ip>`），并用 `http://<tailscale-ip>:9119` 作为 Remote URL。只有你 tailnet 上的设备才能访问它。要通过公网访问后端，请改用 **OAuth（Nous Portal）** 提供商。
:::

### 在 Hermes Desktop 中

**Settings → Gateways → Remote gateway：**

- **Remote URL** —— `http://<backend-host>:9119`（如果用反向代理在前面，支持 `/hermes` 这类路径前缀）
- **Sign in** —— 应用会检测到用户名/密码 gateway 并显示 **Sign in** 按钮；点击它并输入第 1 步的凭据
- **Save and reconnect** —— 把桌面外壳切换到远程后端

会话会自动刷新，并且当后端设置了 `HERMES_DASHBOARD_BASIC_AUTH_SECRET` 时能挺过重启。

### 环境变量覆盖

除了应用内设置，你也可以在启动桌面应用前用环境变量指向某个后端。设置了 `HERMES_DESKTOP_REMOTE_URL` 时，它会覆盖已保存的应用内 URL（Gateway 设置面板会显示 “env override” 徽章并禁用编辑）；你仍然要用面板中的用户名和密码 **Sign in**。

| 环境变量 | 值 |
|---------|-------|
| `HERMES_DESKTOP_REMOTE_URL` | `http://<backend-host>:9119` |

### 排障

- **“Remote gateway incomplete”** —— 你还没有输入远程 URL。
- **登录失败并返回 401 / “Invalid credentials”** —— 用户名或密码与后端的 `HERMES_DASHBOARD_BASIC_AUTH_USERNAME` / `HERMES_DASHBOARD_BASIC_AUTH_PASSWORD` 不匹配。后端对未知用户和错误密码返回相同的通用错误，因此两者都要检查。用 `curl -s http://<host>:9119/api/status | jq '.auth_required, .auth_providers'` 确认门禁 —— 它应报告 `true` 并包含 `"basic"`。
- **没有 “Sign in” 按钮 —— 而是要求会话令牌** —— 用户名/密码提供商未激活（`/api/status` 不会列出 `"basic"`）。请确认已设置用户名和一个密码（或密码哈希），并且 Dashboard 进程已加载它们。
- **每次重启都被登出** —— 把 `HERMES_DASHBOARD_BASIC_AUTH_SECRET` 设为稳定值；否则签名密钥会在每次启动时重新生成。
- **连接被拒绝 / 超时** —— 后端绑定到了 `127.0.0.1`（默认值）而不是可达地址，或者防火墙/VPN 挡住了端口。绑定到 `0.0.0.0` 或 tailscale IP，并向可信网络开放该端口。

## CORS

Web 服务器将 CORS 限制为仅 localhost 来源：

- `http://localhost:9119` / `http://127.0.0.1:9119`（生产环境）
- `http://localhost:3000` / `http://127.0.0.1:3000`
- `http://localhost:5173` / `http://127.0.0.1:5173`（Vite 开发服务器）

如果你在自定义端口上运行服务器，该来源会自动添加。

## 开发

如果你要为 Web Dashboard 前端做贡献：

```bash
# 终端 1：启动后端 API
hermes dashboard --no-open

# 终端 2：启动带 HMR 的 Vite 开发服务器
cd web/
npm install
npm run dev
```

`http://localhost:5173` 上的 Vite 开发服务器会将 `/api` 请求代理到 `http://127.0.0.1:9119` 上的 FastAPI 后端。

前端使用 React 19、TypeScript、Tailwind CSS v4 和 shadcn/ui 风格组件构建。生产构建输出到 `hermes_cli/web_dist/`，由 FastAPI 服务器作为静态 SPA 提供服务。

## 更新时自动构建

运行 `hermes update` 时，如果 `npm` 可用，Web 前端会自动重新构建。这使 Dashboard 与代码更新保持同步。如果未安装 `npm`，更新会跳过前端构建，`hermes dashboard` 将在首次启动时构建。

## 主题与插件

Dashboard 内置六个主题，并可通过用户自定义主题、插件标签页和后端 API 路由进行扩展——全部即插即用，无需克隆仓库。

**实时切换主题**：点击顶部栏语言切换器旁的调色板图标。选择会持久化到 `config.yaml` 的 `dashboard.theme` 下，并在页面加载时恢复。

内置主题：

| 主题 | 特点 |
|-------|-----------|
| **Hermes Teal** (`default`) | 深青色 + 奶油色，系统字体，舒适间距 |
| **Hermes Teal (Large)** (`default-large`) | 与 default 相同，但使用 18px 文字和更宽松的间距 |
| **Midnight** (`midnight`) | 深蓝紫色，Inter + JetBrains Mono |
| **Ember** (`ember`) | 暖深红 + 古铜色，Spectral 衬线体 + IBM Plex Mono |
| **Mono** (`mono`) | 灰度，IBM Plex，紧凑 |
| **Cyberpunk** (`cyberpunk`) | 黑底霓虹绿，Share Tech Mono |
| **Rosé** (`rose`) | 粉色 + 象牙色，Fraunces 衬线体，宽松 |

如需构建自定义主题、添加插件标签页、注入 shell 插槽或暴露插件专属 REST 端点，请参阅 **[扩展 Dashboard](./extending-the-dashboard)**——完整指南涵盖：

- 主题 YAML schema——调色板、排版、布局、资源、componentStyles、colorOverrides、customCSS
- 布局变体——`standard`、`cockpit`、`tiled`
- 插件 manifest、SDK、shell 插槽、页面级插槽（在不覆盖内置页面的情况下注入控件）、后端 FastAPI 路由
- 完整的主题加插件综合演示（Strike Freedom cockpit 示例）
- 发现、重载和故障排查
