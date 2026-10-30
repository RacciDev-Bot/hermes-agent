---
sidebar_position: 2
title: "配置"
description: "配置 Hermes Agent — config.yaml、providers、模型、API 密钥等"
---

# 配置

本页的 Python 依赖命令使用 [PM 准备的源码环境](../reference/package-management.md#developer-workflow)。
依赖变更后，请重新激活该 checkout 并重启 Hermes。

所有设置均存储在 `~/.hermes/` 目录中，便于访问。

## 目录结构

```text
~/.hermes/
├── config.yaml     # 设置（模型、终端、TTS、压缩等）
├── .env            # API 密钥和机密
├── auth.json       # OAuth provider 凭据（Nous Portal 等）
├── SOUL.md         # 主要 agent 身份（系统提示词第 #1 槽位）
├── memories/       # 持久记忆（MEMORY.md、USER.md）
├── skills/         # Agent 创建的技能（通过 skill_manage 工具管理）
├── cron/           # 定时任务
├── sessions/       # Gateway 会话
└── logs/           # 日志（errors.log、gateway.log — 机密自动脱敏）
```

## 管理配置

```bash
hermes config              # 查看当前配置
hermes config edit         # 在编辑器中打开 config.yaml
hermes config get KEY      # 打印解析后的值
hermes config set KEY VAL  # 设置特定值
hermes config unset KEY    # 移除用户设置的值
hermes config check        # 检查缺失选项和过时的已保存选择
hermes config migrate      # 交互式添加缺失选项

# 示例：
hermes config get model
hermes config set model anthropic/claude-opus-4
hermes config set terminal.backend docker
hermes config unset terminal.backend
hermes config set OPENROUTER_API_KEY sk-or-...  # 保存到 .env
```

:::tip
`hermes config set` 命令会自动将值路由到正确的文件 —— 每个 `UPPER_SNAKE` 形式的名称（`OPENROUTER_API_KEY`、`DISCORD_HOME_CHANNEL`、`TELEGRAM_GROUP_ALLOWED_USERS`、`HERMES_TIMEZONE` 等）都是环境变量，会保存到 `.env`，绝不会写入 `config.yaml`；带点的设置则进入 `config.yaml`。其他任何 `UPPER_SNAKE` 名称都会原样保存到 `.env`（并导出到进程环境供插件和技能使用）；位于环境变量写入器禁止列表上的名称（`HERMES_YOLO_MODE`、`PATH` 等）会被拒绝。已知键写在错误前缀下（`gateway.discord.foo`，而 `discord.foo` 本身是已知键）会在写入任何内容之前被拒绝并给出"你是不是想写"提示；加 `--force` 可以照样写入。已知段下的其他未知路径（例如 `agent.max_turnz` 这类拼写错误，或没有预置默认值的运行时读取键）会在写入的同时给出提示，因为仅凭 schema 无法区分两者。对这类路径执行 `hermes config get` 会打印你文件中的值，并在 stderr 提示 Hermes 可能不会读取它，因此遗留键无法悄悄冒充生效中的设置。
:::

## 配置优先级

设置按以下顺序解析（优先级从高到低）：

1. **CLI 参数** —— 例如 `hermes chat --model anthropic/claude-sonnet-4`（单次调用覆盖）
2. **`~/.hermes/config.yaml`** —— 所有非机密设置的主配置文件
3. **`~/.hermes/.env`** —— 环境变量的回退；机密（API 密钥、token、密码）**必须**放这里
4. **内置默认值** —— 未设置任何内容时的硬编码安全默认值

:::info 经验法则
机密（API 密钥、bot token、密码）放入 `.env`。其他所有内容（模型、终端后端、压缩设置、内存限制、工具集）放入 `config.yaml`。当两者都设置时，`config.yaml` 对非机密设置优先。
:::

## 运行时限制 {#runtime-limits}

长时间运行的 Hermes 服务器面（包括 gateway 和
`hermes serve --isolated`）在操作系统支持时，会在启动期间应用配置的
`RLIMIT_NOFILE` 软限制：

```yaml
runtime:
  nofile_soft_limit: 4096
```

默认值为 `4096`。Hermes 会把目标值钳制到操作系统的硬限制，并且绝不会降低一个
已经拥有更高软限制的进程。将值设为 `0`、`false` 或 `null` 可禁用该调整。在 Windows 上以及
无法修改该限制的沙箱中，启动会继续进行而不改动该限制。

## 数据库设置 {#database-settings}

`database:` 段控制 Hermes 如何打开它的 SQLite 状态数据库
（`state.db`），该数据库存储会话、消息和 gateway 路由：

```yaml
database:
  # state.db 的 journal 模式：wal（默认）或 delete。
  # 在 WAL 不安全的文件系统（网络挂载）上使用 delete。在
  # virtiofs/9p 绑定挂载（Docker Desktop、macOS 上的 Podman、OrbStack）上，
  # Hermes 会检测该挂载并以 delete 模式新建数据库。
  # 注意：已存在的磁盘 WAL 数据库永远不会被
  # 在线降级 —— Hermes 会保持 WAL 并记录错误，告诉你配置的
  # delete 未生效（或 WAL 数据库位于跨 VM 挂载上）。要转换
  # 已有数据库，请停止所有使用它的进程并运行
  # `hermes sessions set-journal-mode delete`（见 Sessions）。
  journal_mode: wal

  # 每个 state.db 连接的持久性级别：OFF、NORMAL、FULL、
  # EXTRA（或 0-3）。不设置则沿用 SQLite 的编译期默认值，
  # 不同解释器构建可能不同。在 macOS 上这是下限而非
  # 固定值：低于 FULL 的值会被拒绝，以防 Darwin 的 fsync
  # 重排序；EXTRA 会被接受。
  # synchronous: FULL

  # 可选的 WAL 尺寸 pragma（整数）。不设置 = SQLite 默认值。
  # wal_autocheckpoint: 1000     # 自动检查点之间的页数
  # journal_size_limit: 67108864 # WAL/journal 大小上限（字节）
```

当现有数据库的磁盘 journal 模式在打开时被静默切换为 WAL 时，Hermes 也会发出警告
（每个进程每个数据库一次）——例如运维人员曾手动将其转换为 `delete` 的数据库——
并指出 `database.journal_mode` 是让该选择固定下来的设置。
反向操作永远不会自动发生：已经处于 WAL 模式的数据库在你设置
`journal_mode: delete` 时不会被在线降级（在有打开连接的情况下降级
可能损坏它）。`hermes doctor` 会警告
`<db> is in WAL mode despite database.journal_mode=delete`，直到你停止
该 profile 的所有 Hermes 进程并运行
`hermes sessions set-journal-mode delete`（当仍有东西持有该文件时它会拒绝，
并验证转换后的文件头）。在该警告下，它会列出当前持有数据库的
进程（`<db> is held by PID <n> (<command>)`），
让你知道该停止什么；当持有者扫描不完整或不可用时，它会说
`cannot prove the database is quiet`，而不是给出"一切正常"的结论。

## 环境变量替换

可以在 `config.yaml` 中使用 `${VAR_NAME}` 语法引用环境变量：

```yaml
auxiliary:
  vision:
    api_key: ${GOOGLE_API_KEY}
    base_url: ${CUSTOM_VISION_URL}

delegation:
  api_key: ${DELEGATION_KEY}
```

单个值中可以有多个引用：`url: "${HOST}:${PORT}"`。如果引用的变量未设置，占位符将保持原样（`${UNDEFINED_VAR}` 保持不变）。仅支持 `${VAR}` 语法 —— 裸 `$VAR` 不会被展开。

关于 AI provider 设置（OpenRouter、Anthropic、自定义端点、自托管 LLM、回退模型等），请参阅 [AI Providers](../integrations/providers.md)。

### Provider 超时

可以为 provider 设置 `providers.<id>.request_timeout_seconds` 作为全局请求超时，以及 `providers.<id>.models.<model>.timeout_seconds` 作为特定模型的覆盖值。适用于每种传输方式（OpenAI-wire、原生 Anthropic、Anthropic 兼容）上的主轮次客户端、回退链、凭据轮换后的重建，以及（对于 OpenAI-wire）每请求超时 kwarg —— 因此配置值优先于旧版 `HERMES_API_TIMEOUT` 环境变量。

还可以设置 `providers.<id>.stale_timeout_seconds` 用于非流式陈旧调用检测器，以及 `providers.<id>.models.<model>.stale_timeout_seconds` 作为特定模型的覆盖值。此值优先于旧版 `HERMES_API_CALL_STALE_TIMEOUT` 环境变量。

不设置这些值将保持旧版默认值（`HERMES_API_TIMEOUT=1800`s、`HERMES_API_CALL_STALE_TIMEOUT=90`s、原生 Anthropic 900s）。隐式的非流式 stale 检测会在本地端点上自动禁用，并且会在超大上下文下自动放宽。在 AWS Bedrock（`providers.bedrock`）上，解析后的请求超时是 boto3 Converse 客户端（`bedrock_converse`）与 AnthropicBedrock SDK 客户端的 socket 读取超时，按请求为当前模型选定，因此 `/model` 切换或回退会使用该模型的值。连接超时保持固定（botocore 在 Converse 上为 60s，在 AnthropicBedrock 上为 10s），读取超时约束的是每次 socket 读取，而非整个调用。不设置则保持 botocore 在 Converse 上的 60s 读取超时和 AnthropicBedrock 上的 900s。请参阅 [`cli-config.yaml.example`](https://github.com/NousResearch/hermes-agent/blob/main/cli-config.yaml.example) 中的注释示例。

## 更新行为 {#update-behavior}

### 后台检查 {#background-checks}

被动更新检查（CLI 横幅、TUI 徽章、dashboard、桌面应用）会向
GitHub REST API 询问 `main` 的末端提交，当它与你的检出不同时，再询问
compare 端点以获取精确的提交数和变更日志。它们从不运行
`git fetch` ——在部分克隆（`--filter=blob:none`）中，它们也不会
从 promisor 远程下载缺失的对象（Git 2.44 或更新版本）——并且
每次安装最多**每 24 小时查询一次**（检查失败会在 1 小时后
重试）。应用更新（`hermes update`，或桌面端的
Update 按钮）总是重新抓取并使缓存的答案失效。显式
检查 —— `hermes update --check`、桌面端的 "Check for Updates…" 菜单项、
Settings → About → "Check now" —— 会绕过缓存。

### SSH 认证 {#ssh-authentication}

启动时的更新检查使用与其网络调用相同的隔离 Git
配置来读取 origin URL。因此全局的 `url.*.insteadOf` 重写
无法向公共 HTTPS 路径隐藏一个官方 SSH 远程。

Hermes 的隔离内部 Git 命令默认使用 `ssh -o BatchMode=yes`：
未知的主机密钥、密码，以及需要口令的加密密钥会失败
而不是打开终端提示。拥有可用密钥或
SSH agent 的可信主机仍可完成认证。这不会改变你磁盘上的 Git 或 SSH
配置，也不会改变你在终端工具中运行的命令。

该内部默认值会覆盖仓库的 `core.sshCommand` 设置。显式的
`GIT_SSH_COMMAND` 环境变量仍然优先，因此
自定义身份或传输命令可以保留在那里。如果这类覆盖必须保持非交互，
请在其中包含 `-o BatchMode=yes`；
允许提示的覆盖仍可能打断后台检查。

`hermes update` 的设置位于 `config.yaml` 的 `updates` 下：

```yaml
updates:
  pre_update_backup: quick       # quick（状态快照，默认） | full（快照 + HERMES_HOME zip） | off
  backup_keep: 5                 # 保留多少个完整的更新前备份 zip
  non_interactive_local_changes: stash  # stash | discard
  auto_switch_parked_branch: true       # 自动将一个干净且已完全合并的 parked 分支切回 main
```

`pre_update_backup` 是唯一的更新前安全开关：`quick`（默认）将关键状态文件（配对数据、cron 任务、配置、认证；超过 1 GiB 的文件会跳过）快照到 `state-snapshots/`；`full` 还会把整个 `HERMES_HOME` 打包到 `backups/`，在大型 home 上可能多花几分钟；`off` 两者都禁用。旧版布尔值仍被接受（`true` → `full`，`false` → `off`）。

`config.yaml` 自身的时点副本（在 `hermes setup` 重写它之前、在 `hermes migrate` 编辑它之前、每次文件解析成功时，以及解析失败时）会放入 `backups/config/config.yaml.<reason>.<timestamp>`。相同的重复副本会被跳过，每个原因只保留最新的五份，因此它们不会在 `config.yaml` 旁边堆积。如果 `config.yaml` 已损坏，Hermes 会提供最新的 `good` 副本而不是内置默认值，并在每次启动时警告，直到 YAML 被修好；损坏的文件永远不会被修改。

对于 git 安装，Hermes 在检出更新分支或拉取之前，会自动 stash 已跟踪的脏文件和未跟踪文件。交互式终端更新会在恢复该 stash 前提示。非交互式更新（桌面/聊天应用、gateway 或 `--yes`）使用 `updates.non_interactive_local_changes`：`stash` 在成功拉取后恢复本地源码改动，而 `discard` 在成功拉取后丢弃更新产生的 stash。仅在本地源码改动从不需要持久化的托管安装上使用 `discard`。

在 stash 步骤之前，Hermes 还会恢复由 npm 安装/构建产生的已跟踪 `package-lock.json` 差异。更新前请提交或手动 stash 有意的 lockfile 修改。

## 终端后端配置

Hermes 支持七种终端后端。每种后端决定 agent 的 shell 命令实际在哪里执行 —— 本地机器、Docker 容器、通过 SSH 的远程服务器、Modal 云沙箱（直接或通过 Nous 托管的 gateway）、Daytona 工作区、Vercel Sandbox，或 Singularity/Apptainer 容器。

```yaml
terminal:
  backend: local    # local | docker | ssh | modal | daytona | vercel_sandbox | singularity
  cwd: "."          # Gateway/cron 工作目录（CLI 始终使用启动目录）
  timeout: 180      # 每条命令的超时时间（秒）
  env_passthrough: []  # 转发到沙箱执行的环境变量名（terminal + execute_code）
  singularity_image: "docker://nikolaik/python-nodejs:python3.11-nodejs20"  # Singularity 后端的容器镜像
  modal_image: "nikolaik/python-nodejs:python3.11-nodejs20"                 # Modal 后端的容器镜像
  daytona_image: "nikolaik/python-nodejs:python3.11-nodejs20"               # Daytona 后端的容器镜像
```

对于 Modal、Daytona 和 Vercel Sandbox 等云沙箱，`container_persistent: true` 表示 Hermes 将尝试在沙箱重建后保留文件系统状态。这并不保证相同的活跃沙箱、PID 空间或后台进程之后仍在运行。

### 后端概览

| 后端 | 命令运行位置 | 隔离性 | 最适合 |
|---------|-------------------|-----------|----------|
| **local** | 直接在您的机器上 | 无 | 开发、个人使用 |
| **docker** | 单个持久 Docker 容器（跨会话、`/new`、子 agent 共享） | 完全（命名空间、cap-drop） | 安全沙箱、CI/CD |
| **ssh** | 通过 SSH 的远程服务器 | 网络边界 | 远程开发、强大硬件 |
| **modal** | Modal 云沙箱 | 完全（云 VM） | 临时云计算、评估 |
| **daytona** | Daytona 工作区 | 完全（云容器） | 托管云开发环境 |
| **vercel_sandbox** | Vercel Sandbox | 完全（云 microVM） | 带快照文件系统持久化的云执行 |
| **singularity** | Singularity/Apptainer 容器 | 命名空间（--containall） | HPC 集群、共享机器 |

### Local 后端

默认后端。命令直接在您的机器上运行，无隔离。无需特殊设置。

```yaml
terminal:
  backend: local
```

:::warning
Agent 拥有与您的用户账户相同的文件系统访问权限。使用 `hermes tools` 禁用不需要的工具，或切换到 Docker 进行沙箱隔离。
:::

### Docker 后端

在具有安全加固的 Docker 容器内运行命令（所有权限已删除、无权限提升、PID 限制）。

**单个持久容器，而非每条命令一个容器。** Hermes 在首次使用时启动一个长期运行的容器，并通过 `docker exec` 将每个终端、文件和 `execute_code` 调用路由到同一容器中 —— 跨会话、`/new`、`/reset` 和 `delegate_task` 子 agent，贯穿 Hermes 进程的整个生命周期。工作目录更改、已安装的包以及 `/workspace` 中的文件会从一次工具调用延续到下一次，就像本地 shell 一样。容器在关闭时停止并删除。详情请参阅下方的**容器生命周期**。

```yaml
terminal:
  backend: docker
  docker_image: "nikolaik/python-nodejs:python3.11-nodejs20"
  docker_mount_cwd_to_workspace: false  # 将启动目录挂载到 /workspace
  docker_run_as_host_user: false   # 参见下方"以宿主用户身份运行容器"
  docker_forward_env:              # 转发到容器的环境变量
    - "GITHUB_TOKEN"
  docker_volumes:                  # 宿主目录挂载
    - "/home/user/projects:/workspace/projects"
    - "/home/user/data:/data:ro"   # :ro 表示只读
  docker_extra_args:               # 附加到 `docker run` 的额外标志
    - "--gpus=all"
    - "--network=host"

  # 资源限制
  container_cpu: 1                 # CPU 核心数（0 = 不限制）
  container_memory: 5120           # MB（0 = 不限制）
  container_disk: 51200            # MB（需要 XFS+pquota 上的 overlay2）
  container_persistent: true       # 跨会话持久化 /workspace 和 /root
```

**`terminal.docker_extra_args`**（也可通过 `TERMINAL_DOCKER_EXTRA_ARGS='["--gpus=all"]'` 覆盖）允许传递 Hermes 未作为一级键公开的任意 `docker run` 标志 —— `--gpus`、`--network`、`--add-host`、替代 `--security-opt` 覆盖等。每个条目必须是字符串；该列表最后附加到组装好的 `docker run` 调用中，因此可以在需要时覆盖 Hermes 的默认值。请谨慎使用 —— 与沙箱加固（权限删除、`--user`、workspace 绑定挂载）冲突的标志将悄然削弱隔离性。

**要求：** 已安装并运行 Docker Desktop 或 Docker Engine。Hermes 会探测 `$PATH` 以及常见的 macOS 安装位置（`/usr/local/bin/docker`、`/opt/homebrew/bin/docker`、Docker Desktop 应用包）。开箱即用支持 Podman：设置 `HERMES_DOCKER_BINARY=podman`（或完整路径）以在两者都安装时强制使用它。

**容器生命周期：** Hermes 为每个终端和文件工具调用重用单个长期运行的容器（`docker run -d ... sleep 2h`），跨会话、`/new`、`/reset` 和 `delegate_task` 子 agent，贯穿 Hermes 进程的整个生命周期。命令通过带登录 shell 的 `docker exec` 运行，因此工作目录更改、已安装的包以及 `/workspace` 中的文件都会从一次工具调用延续到下一次。容器在 Hermes 关闭时（或空闲清理回收时）停止并删除。

通过 `delegate_task(tasks=[...])` 生成的并行子 agent 共享这一个容器 —— 并发的 `cd`、环境变量修改以及对同一路径的写入会发生冲突。如果子 agent 需要隔离的沙箱，必须通过 `register_task_env_overrides()` 注册每任务镜像覆盖，RL 和基准测试环境（TerminalBench2、HermesSweEnv 等）会自动为其每任务 Docker 镜像执行此操作。

**安全加固：**
- `--cap-drop ALL`，仅添加回 `DAC_OVERRIDE`、`CHOWN`、`FOWNER`
- `--security-opt no-new-privileges`
- `--pids-limit 256`
- `/tmp`（512MB）、`/var/tmp`（256MB）、`/run`（64MB）的大小限制 tmpfs

**凭据转发：** `docker_forward_env` 中列出的环境变量首先从您的 shell 环境解析，然后回退到 `~/.hermes/.env`。技能也可以声明 `required_environment_variables`，这些变量会自动合并。

### SSH 后端

通过 SSH 在远程服务器上运行命令。使用 ControlMaster 进行连接复用（5 分钟空闲保活）。默认启用持久 shell —— 状态（cwd、环境变量）在命令之间保持。

```yaml
terminal:
  backend: ssh
  persistent_shell: true           # 保持长期运行的 bash 会话（默认：true）
```

**必需的环境变量：**

```bash
TERMINAL_SSH_HOST=my-server.example.com
TERMINAL_SSH_USER=ubuntu
```

**可选：**

| 变量 | 默认值 | 描述 |
|----------|---------|-------------|
| `TERMINAL_SSH_PORT` | `22` | SSH 端口 |
| `TERMINAL_SSH_KEY` | （系统默认） | SSH 私钥路径 |
| `TERMINAL_SSH_PERSISTENT` | `true` | 启用持久 shell |

**工作原理：** 使用 `BatchMode=yes` 和 `StrictHostKeyChecking=accept-new` 在初始化时连接。持久 shell 在远程主机上保持单个 `bash -l` 进程存活，通过临时文件进行通信。需要 `stdin_data` 或 `sudo` 的命令会自动回退到单次模式。

### Modal 后端

在 [Modal](https://modal.com) 云沙箱中运行命令。每个任务获得一个具有可配置 CPU、内存和磁盘的隔离 VM。文件系统可以跨会话快照/恢复。

```yaml
terminal:
  backend: modal
  container_cpu: 1                 # CPU 核心数
  container_memory: 5120           # MB（5GB）
  container_disk: 51200            # MB（50GB）
  container_persistent: true       # 快照/恢复文件系统
```

**必需：** `MODAL_TOKEN_ID` + `MODAL_TOKEN_SECRET` 环境变量，或 `~/.modal.toml` 配置文件。

**持久化：** 启用后，沙箱文件系统在清理时快照，并在下次会话时恢复。快照在 `~/.hermes/modal_snapshots.json` 中跟踪。这保留文件系统状态，而非活跃进程、PID 空间或后台任务。

**凭据文件：** 自动从 `~/.hermes/` 挂载（OAuth token 等），并在每条命令前同步。

### Daytona 后端

在 [Daytona](https://daytona.io) 托管工作区中运行命令。支持停止/恢复以实现持久化。

```yaml
terminal:
  backend: daytona
  container_cpu: 1                 # CPU 核心数
  container_memory: 5120           # MB → 转换为 GiB
  container_disk: 10240            # MB → 转换为 GiB（最大 10 GiB）
  container_persistent: true       # 停止/恢复而非删除
```

**必需：** `DAYTONA_API_KEY` 环境变量。

**持久化：** 启用后，沙箱在清理时停止（而非删除），并在下次会话时恢复。沙箱名称遵循 `hermes-{task_id}` 模式。

**磁盘限制：** Daytona 强制执行 10 GiB 最大值。超过此值的请求将被截断并发出警告。

### Vercel Sandbox 后端

在 [Vercel Sandbox](https://vercel.com/docs/vercel-sandbox) 云 microVM 中运行命令。Hermes 使用普通的终端和文件工具接口；没有 Vercel 特定的面向模型的工具。

```yaml
terminal:
  backend: vercel_sandbox
  vercel_runtime: node24          # node24 | node22 | python3.13
  cwd: /vercel/sandbox            # 默认工作区根目录
  container_persistent: true      # 快照/恢复文件系统
  container_disk: 51200           # 仅共享默认值；不支持自定义磁盘
```

**必需安装：** 安装可选 SDK 扩展：

```bash
python -c "import pm; pm.sync_venv(['vercel'], explicit=True)"
```

**必需认证：** 使用 `VERCEL_TOKEN`、`VERCEL_PROJECT_ID` 和 `VERCEL_TEAM_ID` 三者全部配置访问令牌认证。这是在 Render、Railway、Docker 及类似宿主上部署和正常长期运行 Hermes 进程的受支持设置。

对于一次性本地开发，Hermes 也接受短期 Vercel OIDC token：

```bash
VERCEL_OIDC_TOKEN="$(vc project token <project-name>)" hermes chat
```

在已链接的 Vercel 项目目录中，可以省略项目名称：

```bash
VERCEL_OIDC_TOKEN="$(vc project token)" hermes chat
```

OIDC token 是短期的，不应作为文档化的部署路径使用。

**运行时：** `terminal.vercel_runtime` 支持 `node24`、`node22` 和 `python3.13`。未设置时，Hermes 默认使用 `node24`。

**持久化：** 当 `container_persistent: true` 时，Hermes 在清理期间对沙箱文件系统进行快照，并从该快照为同一任务恢复后续沙箱。快照内容可以包括复制到沙箱中的 Hermes 同步凭据、技能和缓存文件。这仅保留文件系统状态；不保留活跃沙箱身份、PID 空间、shell 状态或正在运行的后台进程。

**后台命令：** `terminal(background=true)` 使用 Hermes 的通用非本地后台进程流程。您可以在沙箱存活期间通过普通进程工具生成、轮询、等待、查看日志和终止进程。Hermes 不提供清理或重启后的原生 Vercel 分离进程恢复。

**磁盘大小：** Vercel Sandbox 目前不支持 Hermes 的 `container_disk` 资源旋钮。将 `container_disk` 保持未设置或使用共享默认值 `51200`；非默认值会导致诊断和后端创建失败，而不是被静默忽略。

### Singularity/Apptainer 后端

在 [Singularity/Apptainer](https://apptainer.org) 容器中运行命令。专为 Docker 不可用的 HPC 集群和共享机器设计。

```yaml
terminal:
  backend: singularity
  singularity_image: "docker://nikolaik/python-nodejs:python3.11-nodejs20"
  container_cpu: 1                 # CPU 核心数
  container_memory: 5120           # MB
  container_persistent: true       # 可写覆盖层跨会话持久化
```

**要求：** `$PATH` 中有 `apptainer` 或 `singularity` 二进制文件。

**镜像处理：** Docker URL（`docker://...`）自动转换为 SIF 文件并缓存。现有 `.sif` 文件直接使用。

**临时目录：** 按顺序解析：`TERMINAL_SCRATCH_DIR` → `TERMINAL_SANDBOX_DIR/singularity` → `/scratch/$USER/hermes-agent`（HPC 惯例）→ `~/.hermes/sandboxes/singularity`。

**隔离：** 使用 `--containall --no-home` 实现完全命名空间隔离，不挂载宿主 home 目录。

### 常见终端后端问题

如果终端命令立即失败或终端工具报告为已禁用：

- **Local** —— 无特殊要求。入门时最安全的默认选项。
- **Docker** —— 运行 `docker version` 验证 Docker 是否正常工作。如果失败，修复 Docker 或执行 `hermes config set terminal.backend local`。
- **SSH** —— `TERMINAL_SSH_HOST` 和 `TERMINAL_SSH_USER` 都必须设置。如果缺少任一项，Hermes 会记录清晰的错误。
- **Modal** —— 需要 `MODAL_TOKEN_ID` 环境变量或 `~/.modal.toml`。运行 `hermes doctor` 检查。
- **Daytona** —— 需要 `DAYTONA_API_KEY`。Daytona SDK 处理服务器 URL 配置。
- **Singularity** —— 需要 `$PATH` 中有 `apptainer` 或 `singularity`。HPC 集群上常见。

如有疑问，将 `terminal.backend` 设回 `local` 并首先验证命令在那里运行。

### 拆卸时远程到宿主文件同步

对于 **SSH**、**Modal** 和 **Daytona** 后端（agent 的工作树位于与运行 Hermes 的宿主不同的机器上），Hermes 跟踪 agent 在远程沙箱中触及的文件，并在会话拆卸/沙箱清理时，将修改的文件**同步回宿主**，存放在 `~/.hermes/cache/remote-syncs/<session-id>/` 下。

- 触发时机：会话关闭、`/new`、`/reset`、gateway 消息超时、子 agent 使用远程后端时 `delegate_task` 子 agent 完成。
- 覆盖 agent 修改的整个树，而不仅仅是它明确打开的文件。添加、编辑和删除都会被捕获。
- 远程沙箱可能在您查找时已被拆除；本地 `~/.hermes/cache/remote-syncs/…` 副本是 agent 更改内容的权威记录。
- 大型二进制输出（模型检查点、原始数据集）按大小限制 —— 同步跳过超过 `file_sync_max_mb`（默认 `100`）的文件。如果您期望更大的工件返回，请调高该值。

```yaml
terminal:
  file_sync_max_mb: 100     # 默认 —— 同步最大 100 MB 的文件
  file_sync_enabled: true   # 默认 —— 设为 false 可完全跳过同步
```

这是从会话结束后被销毁的临时云沙箱中恢复结果的方式，无需告诉 agent 显式地 `scp` 或 `modal volume put` 每个工件。

### Docker 卷挂载

使用 Docker 后端时，`docker_volumes` 允许您与容器共享宿主目录。每个条目使用标准 Docker `-v` 语法：`host_path:container_path[:options]`。

```yaml
terminal:
  backend: docker
  docker_volumes:
    - "/home/user/projects:/workspace/projects"   # 读写（默认）
    - "/home/user/datasets:/data:ro"              # 只读
    - "/home/user/.hermes/cache/documents:/output" # Gateway 可见的导出
```

适用于：
- **向 agent 提供文件**（数据集、配置、参考代码）
- **从 agent 接收文件**（生成的代码、报告、导出）
- **共享工作区**，您和 agent 都访问相同的文件

如果您使用消息 gateway 并希望 agent 通过 `MEDIA:/...` 发送生成的文件，建议使用专用的宿主可见导出挂载，例如 `/home/user/.hermes/cache/documents:/output`。

- 在 Docker 中将文件写入 `/output/...`
- 在 `MEDIA:` 中发出**宿主路径**，例如：`MEDIA:/home/user/.hermes/cache/documents/report.txt`
- **不要**发出 `/workspace/...` 或 `/output/...`，除非该确切路径在宿主上对 gateway 进程也存在

:::warning
YAML 重复键会静默覆盖之前的键。如果您已有 `docker_volumes:` 块，请将新挂载合并到同一列表中，而不是在文件后面再添加一个 `docker_volumes:` 键。
:::

也可以通过环境变量设置：`TERMINAL_DOCKER_VOLUMES='["/host:/container"]'`（JSON 数组）。

### Docker 凭据转发

默认情况下，Docker 终端会话不继承任意宿主凭据。如果您需要在容器内使用特定 token，请将其添加到 `terminal.docker_forward_env`。

```yaml
terminal:
  backend: docker
  docker_forward_env:
    - "GITHUB_TOKEN"
    - "NPM_TOKEN"
```

Hermes 首先从您当前的 shell 解析每个列出的变量，然后回退到通过 `hermes config set` 保存的 `~/.hermes/.env`。

:::warning
`docker_forward_env` 中列出的任何内容都会对容器内运行的命令可见。只转发您愿意暴露给终端会话的凭据。
:::

### 以宿主用户身份运行容器

默认情况下，Docker 容器以 `root`（UID 0）身份运行。在 `/workspace` 或其他绑定挂载中创建的文件在宿主上归 root 所有，因此会话结束后您必须 `sudo chown` 才能从宿主编辑器编辑它们。`terminal.docker_run_as_host_user` 标志解决了这个问题：

```yaml
terminal:
  backend: docker
  docker_run_as_host_user: true   # 默认：false
```

启用后，Hermes 将 `--user $(id -u):$(id -g)` 附加到 `docker run` 命令，使写入绑定挂载目录（`/workspace`、`/root`、`docker_volumes` 中的任何内容）的文件归您的宿主用户所有，而非 root。权衡：容器将无法再 `apt install` 或写入 `/root/.npm` 等 root 拥有的路径 —— 如果您同时需要这两者，请使用 `HOME` 归非 root 用户所有的基础镜像（或在镜像构建时添加所需工具）。

保持 `false`（默认）以获得向后兼容的行为。当您的工作流主要是"编辑挂载的宿主文件"且厌倦了 `sudo chown -R` 时，请开启此选项。

### Snap 打包的 Docker（AppArmor） {#snap-packaged-docker-apparmor}

在 Docker 以 snap 方式安装的主机上（Ubuntu 云镜像很常见，例如 Azure VM），snap 的 AppArmor 约束会拒绝沙箱的两个加固参数，容器在启动时就会死掉：

```
exec /sbin/docker-init: operation not permitted     # --init
exec /usr/bin/sleep: operation not permitted        # --security-opt no-new-privileges
```

这是 snapd 的限制（[LP#1908448](https://bugs.launchpad.net/snapd/+bug/1908448)），不是 Hermes 能绕过的。要么改用 Docker 官方 apt 仓库安装 Docker（首选 —— 所有加固都保留），要么显式选择加入：

```yaml
terminal:
  docker_snap_compat: true   # 去掉 --init 和 no-new-privileges；cap-drop、tmpfs、PID 限制保留
```

开启后，沙箱内的僵尸进程不再由 init 回收，容器内的 setuid 二进制可以重新获得特权；容器启动时会记录一条警告。

### 可选：将启动目录挂载到 `/workspace`

Docker 沙箱默认保持隔离。Hermes **不会**将您当前的宿主工作目录传入容器，除非您明确选择加入。

在 `config.yaml` 中启用：

```yaml
terminal:
  backend: docker
  docker_mount_cwd_to_workspace: true
```

启用后：
- 如果您从 `~/projects/my-app` 启动 Hermes，该宿主目录将绑定挂载到 `/workspace`
- Docker 后端从 `/workspace` 开始
- 文件工具和终端命令都能看到相同的挂载项目

禁用时，`/workspace` 保持沙箱所有，除非您通过 `docker_volumes` 显式挂载内容。

安全权衡：
- `false` 保留沙箱边界
- `true` 使沙箱直接访问您启动 Hermes 的目录

仅在您有意希望容器处理实时宿主文件时才选择加入。

### 持久 Shell

默认情况下，每条终端命令在其自己的子进程中运行 —— 工作目录、环境变量和 shell 变量在命令之间重置。启用**持久 shell** 后，单个长期运行的 bash 进程在 `execute()` 调用之间保持存活，使状态在命令之间保持。

这对 **SSH 后端**最有用，它还消除了每条命令的连接开销。持久 shell **对 SSH 默认启用**，对本地后端禁用。

```yaml
terminal:
  persistent_shell: true   # 默认 —— 为 SSH 启用持久 shell
```

禁用：

```bash
hermes config set terminal.persistent_shell false
```

**跨命令保持的内容：**
- 工作目录（`cd /tmp` 对下一条命令生效）
- 导出的环境变量（`export FOO=bar`）
- Shell 变量（`MY_VAR=hello`）

**优先级：**

| 级别 | 变量 | 默认值 |
|-------|----------|---------|
| 配置 | `terminal.persistent_shell` | `true` |
| SSH 覆盖 | `TERMINAL_SSH_PERSISTENT` | 遵循配置 |
| Local 覆盖 | `TERMINAL_LOCAL_PERSISTENT` | `false` |

每个后端的环境变量具有最高优先级。如果您也想在本地后端使用持久 shell：

```bash
export TERMINAL_LOCAL_PERSISTENT=true
```

:::note
需要 `stdin_data` 或 sudo 的命令会自动回退到单次模式，因为持久 shell 的 stdin 已被 IPC 协议占用。
:::

有关每个后端的详细信息，请参阅[代码执行](features/code-execution.md)和 [README 的终端部分](features/tools.md)。

## 技能设置

技能可以通过其 SKILL.md frontmatter 声明自己的配置设置。这些是非机密值（路径、偏好、域设置），存储在 `config.yaml` 的 `skills.config` 命名空间下。

```yaml
skills:
  config:
    myplugin:
      path: ~/myplugin-data   # 示例 —— 每个技能定义自己的键
```

**技能设置的工作原理：**

- `hermes config migrate` 扫描所有已启用的技能，找到未配置的设置，并提供提示
- `hermes config show` 在"技能设置"下显示所有技能设置及其所属技能
- 技能加载时，其解析的配置值会自动注入到技能上下文中

**手动设置值：**

```bash
hermes config set skills.config.myplugin.path ~/myplugin-data
```

有关在您自己的技能中声明配置设置的详细信息，请参阅[创建技能 — 配置设置](../developer-guide/creating-skills.md#config-settings-configyaml)。

### 每个会话自动加载技能 {#auto-loading-skills-every-session}

固定技能，使其在每个新会话开始时、在每个界面上都完整加载：

```yaml
skills:
  auto_load:
    - my-workflow
    - github-pr-workflow
```

在首次构建系统提示时每会话解析一次（因此提示保持缓存稳定；修改在下一会话生效）。缺失或已禁用的技能会给出警告并被跳过；`--ignore-rules` / `HERMES_IGNORE_RULES=1` 会抑制该列表。按 profile 作用域。参见 [CLI —— 通过配置持久自动加载](./cli.md)。

### Agent 创建技能写入的守卫

当 agent 使用 `skill_manage` 创建、编辑、修补或删除技能时，Hermes 可以选择扫描新/更新的内容以查找危险关键字模式（凭据收集、明显的 prompt 注入、数据外泄指令）。扫描器**默认关闭** —— 合法触及 `~/.ssh/` 或提及 `$OPENAI_API_KEY` 的真实 agent 工作流触发启发式规则过于频繁。如果您希望扫描器在 agent 的技能写入落地前提示您，请重新开启：

```yaml
skills:
  guard_agent_created: true   # 默认：false
```

开启后，任何被标记的 `skill_manage` 写入都会以审批提示的形式出现，并附带扫描器的理由。接受的写入落地；拒绝的写入向 agent 返回解释性错误。

### 技能写入的写入审批 {#write-approval-for-skill-writes}

独立于上面的内容扫描器，`skills.write_approval` 会把**每一次** agent 技能写入（创建 / 编辑 / 修补 / 删除 / 支持文件）挡在你的显式审批之后 —— 与危险命令相同的批准/拒绝机制：

```yaml
skills:
  write_approval: false   # false = 自由写入（默认）| true = 每次写入都暂存待审
```

开启后，技能写入会暂存到 `~/.hermes/pending/skills/`，并通过 `/skills pending`、`/skills diff <id>`、`/skills approve <id>`、`/skills reject <id>` 审阅 —— 可在 CLI 或任何消息平台使用。运行时用 `/skills approval on|off` 切换。内存有相同的门禁（`memory.write_approval`，见下文）。完整演练：[门控 agent 技能写入](./features/skills.md)。

## 内存配置

```yaml
memory:
  memory_enabled: true
  user_profile_enabled: true
  memory_char_limit: 2200   # ~800 tokens
  user_char_limit: 1375     # ~500 tokens
```

## 上下文文件截断 {#context-file-truncation}

控制 Hermes 在应用头/尾截断之前从每个自动上下文文件加载多少内容。这适用于注入系统提示的文件，例如 `SOUL.md`、`.hermes.md`、`AGENTS.md`、`CLAUDE.md` 和 `.cursorrules`。它**不**影响 `read_file` 工具。

```yaml
context_file_max_chars: null  # 默认 —— 随模型上下文窗口缩放的动态上限（下限 20K，上限 500K 字符）
```

设置一个正整数即可将上限固定下来，而不是使用动态行为：

```yaml
context_file_max_chars: 25000
```

每次上下文文件读取也受 `context_file_read_timeout`（秒，默认 `5.0`）限制。读取时间超过该值的文件——通常出现在 iCloud Drive、OneDrive 或 NFS 这类网络文件系统上——会被跳过并给出警告，这样系统提示的其余部分仍能正常加载：

```yaml
context_file_read_timeout: 5.0
```

## 文件读取安全

控制单次 `read_file` 调用可以返回多少内容。超过限制的读取将被拒绝，并向 agent 返回错误，提示使用 `offset` 和 `limit` 读取较小范围。这可以防止单次读取压缩的 JS 包或大型数据文件时淹没上下文窗口。

```yaml
file_read_max_chars: 100000  # 默认 —— ~25-35K tokens
```

如果您使用具有大上下文窗口的模型并经常读取大文件，请调高此值。对于小上下文模型，请降低以保持读取高效：

```yaml
# 大上下文模型（200K+）
file_read_max_chars: 200000

# 小型本地模型（16K 上下文）
file_read_max_chars: 30000
```

Agent 还会自动去重文件读取 —— 如果同一文件区域被读取两次且文件未更改，则返回轻量级存根而不是重新发送内容。这在上下文压缩后重置，以便 agent 在内容被摘要后可以重新读取文件。

## 工具输出截断限制

三个相关的上限控制工具在 Hermes 截断之前可以返回多少原始输出：

```yaml
tool_output:
  max_bytes: 50000        # 终端输出上限（字符）
  max_lines: 2000         # read_file 分页上限
  max_line_length: 2000   # read_file 行号视图中的每行上限
```

- **`max_bytes`** —— 当 `terminal` 命令产生超过此字符数的合并 stdout/stderr 时，Hermes 保留前 40% 和后 60%，并在中间插入 `[OUTPUT TRUNCATED]` 通知。默认 `50000`（典型分词器约 12-15K tokens）。
- **`max_lines`** —— 单次 `read_file` 调用的 `limit` 参数上限。超过此值的请求将被截断，以防单次读取淹没上下文窗口。默认 `2000`。
- **`max_line_length`** —— `read_file` 发出行号视图时应用的每行上限。超过此长度的行将被截断为此字符数，后跟 `... [truncated]`。默认 `2000`。

对于具有大上下文窗口且每次调用可以承受更多原始输出的模型，请调高限制。对于小上下文模型，请降低以保持工具结果紧凑：

```yaml
# 大上下文模型（200K+）
tool_output:
  max_bytes: 150000
  max_lines: 5000

# 小型本地模型（16K 上下文）
tool_output:
  max_bytes: 20000
  max_lines: 500
```

### 工具结果溢出预算 {#tool-result-spillover-budget}

与截断分开，超大的工具*结果*不会被切掉，而是溢出到磁盘：完整输出保存到 `$HERMES_HOME/cache/spillover/` 下，上下文内的内容替换为一段预览加上已保存文件的路径（可用 `read_file` 配合 `offset`/`limit` 读取，或用 `execute_code` 处理）。通用的每结果溢出阈值是 100,000 字符，并会为小上下文模型自动调低。只要任何会话的转写仍指向某个溢出文件，该文件就会保留，并在该会话被删除或修剪时随之离开；gateway 的每小时内务处理与每个进程的首次溢出只清理早于会话保留窗口（`sessions.retention_days`，默认 90 天，绝不少于 24 小时）的无引用文件，因此稍后恢复的会话仍能找到其转写所指的内容。

MCP 工具结果（名为 `mcp_*` 的工具）使用更紧的 **50,000 字符**默认阈值：MCP 服务器经常返回大型未分页载荷（工具发现目录、批量执行），否则它们会落在通用阈值之下，在之后每一轮都撑大上下文。什么都不会丢 —— 完整结果保留在磁盘上。可通过以下方式覆盖阈值：

```yaml
tool_budget:
  mcp_result_size_chars: 50000   # mcp_* 工具的每结果溢出阈值
```

MCP 阈值始终封顶于（可能按上下文缩放的）通用每结果阈值，因此调高它也无法超过活跃模型窗口允许的范围。

Hermes 还会标记**provider 侧省略**：当 MCP 或 web 工具结果内嵌了自己的截断标记（`...N more items`、`"has_more": true`、"saved to sandbox" 说明）时，会在结果后附加一行提示，警告可见数据不完整，在把任何枚举当作完整之前应先分页/抓取。

## 全局工具集禁用

要在 CLI 和每个 gateway 平台上统一禁用特定工具集，请在 `agent.disabled_toolsets` 下列出其名称：

```yaml
agent:
  disabled_toolsets:
    - memory       # 隐藏内存工具 + MEMORY_GUIDANCE 注入
    - web          # 任何地方都不使用 web_search / web_extract
```

这在每个平台的工具配置（由 `hermes tools` 写入的 `platform_toolsets`）**之后**应用，因此此处列出的工具集始终被删除 —— 即使平台的已保存配置仍然列出它。当您希望有一个"到处关闭 X"的单一开关而不是编辑 `hermes tools` UI 中的 15+ 个平台行时，请使用此选项。

留空列表或省略键不会产生任何效果。

## Git Worktree 隔离

启用隔离的 git worktree，以便在同一仓库上并行运行多个 agent：

```yaml
worktree: true    # 始终创建 worktree（与 hermes -w 相同）
# worktree: false # 默认 —— 仅在传递 -w 标志时
```

启用后，每个 CLI 会话在 `.worktrees/` 下创建一个带有自己分支的新 worktree。Agent 可以编辑文件、提交、推送和创建 PR，而不会相互干扰。干净的 worktree 在退出时删除；脏的 worktree 保留以供手动恢复。

您还可以通过仓库根目录中的 `.worktreeinclude` 列出要复制到 worktree 的 gitignore 文件：

```
# .worktreeinclude
.env
.venv/
node_modules/
```

## 上下文压缩 {#context-compression}

Hermes 自动压缩长对话以保持在模型的上下文窗口内。压缩摘要器是一个单独的 LLM 调用 —— 您可以将其指向任何 provider 或端点。

所有压缩设置都在 `config.yaml` 中（无环境变量）。

### 完整参考

```yaml
compression:
  enabled: true                                     # 开启/关闭压缩
  threshold: 0.50                                   # 在上下文限制的此百分比时压缩
  target_ratio: 0.20                                # 保留为最近尾部的阈值分数
  tail_mode: legacy                                 # 尾部保留策略："legacy"（0.20×窗口的逐字尾部）或 "lean"（截取 2.5% 窗口、10K-25K 上下限的精简尾部，摘要中附带分块摘录、锚点索引与 session_search 恢复指引 — 压缩后保留 token 约减少 3 倍）
  protect_last_n: 20                                # 保持未压缩的最少最近消息数
  hygiene_hard_message_limit: 5000                  # Gateway 安全阀 —— 见下文
  context_timeout_seconds: 120                      # Agent 侧 compress_context 无进展超时（秒）—— 见下文
  context_total_ceiling_seconds: 600                # Agent 侧 compress_context 预提交等待上限（秒；已开始的 SessionDB 提交不会被放弃，超限会记录日志并告警）

# 摘要模型/provider 在 auxiliary: 下配置：
auxiliary:
  compression:
    model: ""                                       # 空 = 使用主聊天模型。覆盖为例如 "google/gemini-3-flash-preview" 以获得更便宜/更快的压缩。
    provider: "auto"                                # Provider："auto"、"openrouter"、"nous"、"codex"、"main" 等
    base_url: null                                  # 自定义 OpenAI 兼容端点（覆盖 provider）
```

:::info 旧版配置迁移
带有 `compression.summary_model`、`compression.summary_provider` 和 `compression.summary_base_url` 的旧版配置在首次加载时自动迁移到 `auxiliary.compression.*`（配置版本 17）。无需手动操作。
:::

`hygiene_hard_message_limit` 是仅限 gateway 的**预压缩安全阀**。它的存在是为了打破一个死循环：当超大会话的 API 调用持续断开时，gateway 永远收不到 token 使用数据，基于 token 的阈值因此无法触发，于是 transcript 持续增长、断开愈发严重。这个基于消息数的下限仅凭消息数量触发（无论 API 是否失败，消息数始终已知），强制压缩以恢复会话。默认 `5000` —— 远高于任何正常会话，包括做数千次短轮次的大上下文（1M+）模型，它们早就在 token 阈值处压缩了。对于异常平台可调得更高；要强制更积极的压缩则调低。在运行中的 gateway 上编辑此值将在下一条消息时生效（见下文）。

`context_timeout_seconds`（默认 `120`）是 agent 侧 `compress_context`（对话循环、预检压缩、手动 `/compress`）的**无进展超时**，语义与 gateway 会话预压缩（session hygiene）的 inactivity 预算相同：摘要模型仍在流式出 token 时会延长等待；仅当完全无输出时才跳过压缩并保留原消息。设为 `0` 可关闭。Gateway 会话预压缩仍使用自己的 `hygiene_timeout_seconds`，不会被双重包装。

`context_total_ceiling_seconds`（默认 `600`）限制即使仍有 token 推进时的 agent 侧**预提交**等待时间（摘要 / 流式阶段），并会被钳制为至少等于 `context_timeout_seconds`。确切保证：**摘要阶段受该上限约束；提交阶段若超出上限则记录日志并向用户告警。**一旦 worker 已进入 compression commit fence 且 SessionDB 变更正在进行，提交绝不会被中途放弃（那会导致 transcript 分叉），但等待不再是静默的：若提交超过上限，Hermes 会记录超时（WARNING，重复时升级为 ERROR），通过用户可见的警告通道发送一次性提醒，并以有界增量继续等待直到提交完成。

:::tip Gateway 热重载压缩和上下文长度
从最近的版本开始，在运行中的 gateway 上编辑 `config.yaml` 中的 `model.context_length` 或任何 `compression.*` 键将在下一条消息时生效 —— 无需 gateway 重启、`/reset` 或会话轮换。缓存的 agent 签名包含这些键，因此 gateway 在检测到更改时会透明地重建 agent。API 密钥和工具/技能配置仍需要通常的重载路径。
:::

### 常见设置

**默认（自动检测）—— 无需配置：**
```yaml
compression:
  enabled: true
  threshold: 0.50
```
使用您的主 provider 和主模型。如果您希望在比主聊天模型更便宜的模型上进行压缩，请覆盖每任务（例如 `auxiliary.compression.provider: openrouter` + `model: google/gemini-2.5-flash`）。

**强制特定 provider**（基于 OAuth 或 API 密钥）：
```yaml
auxiliary:
  compression:
    provider: nous
    model: gemini-3-flash
```
适用于任何 provider：`nous`、`openrouter`、`codex`、`anthropic`、`main` 等。

**自定义端点**（自托管、Ollama、zai、DeepSeek 等）：
```yaml
auxiliary:
  compression:
    model: glm-4.7
    base_url: https://api.z.ai/api/coding/paas/v4
```
指向自定义 OpenAI 兼容端点。使用 `OPENAI_API_KEY` 进行认证。

### 三个旋钮的交互方式

| `auxiliary.compression.provider` | `auxiliary.compression.base_url` | 结果 |
|---------------------|---------------------|--------|
| `auto`（默认） | 未设置 | 自动检测最佳可用 provider |
| `nous` / `openrouter` / 等 | 未设置 | 强制使用该 provider，使用其认证 |
| 任意 | 已设置 | 直接使用自定义端点（忽略 provider） |

### 流式进展超时 {#stream-progress-timeout}

当摘要以流式方式返回时（压缩在 chat-completions 和 Responses 路由上始终流式，例如 `openai-codex` 或自定义 OpenAI 兼容端点），有两个超时生效，且相互独立：

- `auxiliary.compression.timeout` —— 整个请求的预算（默认 120s）。
- `auxiliary.compression.no_progress_timeout` —— 流在没有**实质性**事件（文本/推理增量或完成的输出项）的情况下可以持续多久，之后该尝试中止（`... stream stalled: no new output for Ns`），由常规的重试/回退链接管。未设置时默认 **60s**。在 chat-completions 流上，窗口从第一个 token 开始。在此之前，远程端点适用主循环的陈旧容忍度（设置了 provider 的 `providers.<id>.stale_timeout_seconds` 时用它，否则 180s，大提示会提高，o3 或 DeepSeek R1 这类推理模型提高到 600s；上限为 `timeout`），因此静默思考的模型不会被切断。在本地端点（localhost、LAN、Tailscale）上，静默预填充只受 `timeout` 约束。同一规则适用于每个流式辅助任务，包括 MoA 参考调用。保活与生命周期帧（`response.in_progress`、ping）不计为进展；每个实质性事件都会重新武装窗口，因此缓慢但仍在推进的摘要永远不会被它切断。

单独调高 `timeout` **不会**拓宽进展窗口 —— 一个配置为 600s 的请求在 60s 间隙后仍会中止。请设置 `no_progress_timeout` 来改变该间隙；有效窗口封顶于 `timeout`，宿主的硬截止/取消仍然优先。宿主自身的不活动预算在这里是外层上限：agent 内压缩在摘要器静默 `compression.context_timeout_seconds`（默认 120s，下限为有效的 `auxiliary.compression.timeout`，后者本身至少 300s）后放弃，gateway 内务处理在 `compression.hygiene_timeout_seconds`（默认 30s）后放弃，因此大于适用宿主预算的 `no_progress_timeout` 会被它静默截短。该键是按任务设置的（`auxiliary.<task>.no_progress_timeout`），因此为压缩放宽它不会改变其他辅助任务。不是正数的值会被忽略，并在日志中给出警告，然后应用 60s 默认值。

```yaml
auxiliary:
  compression:
    provider: openai-codex
    timeout: 600
    no_progress_timeout: 180   # 在长时间推理摘要上容忍 3 分钟的静默间隙
```

:::warning 摘要模型上下文长度要求
摘要模型**必须**具有至少与您的主 agent 模型一样大的上下文窗口。压缩器将对话的完整中间部分发送给摘要模型 —— 如果该模型的上下文窗口小于主模型的，摘要调用将因上下文长度错误而失败。发生这种情况时，中间轮次将**在没有摘要的情况下被丢弃**，静默丢失对话上下文。如果您覆盖模型，请验证其上下文长度满足或超过您的主模型。
:::

## Gateway 轮次租约超时 {#gateway-turn-lease-timeout}

gateway 按解析后的会话 ID 串行化轮次，使两个路由键无法并发加载和写入同一份转写记录。可以独立于常规的 agent 非活动超时来配置最大租约等待时间：

```yaml
agent:
  gateway_turn_lease_timeout: 5
```

如果该预算到期时另一个轮次仍持有会话租约，Hermes 会失败即关闭（fail closed）：它不会加载转写记录，也不会为等待中的消息运行模型。用户会收到拒绝通知并必须重发。Hermes 不会自动重排该消息，因为在没有持久化排序和幂等性的情况下这样做可能导致它被处理两次。非正值使用 5 秒默认值。

## Gateway 重启排空 {#gateway-restart-drain}

当 gateway 停止或重启时（SIGTERM、`hermes update`、`hermes gateway stop`，或 `hermes gateway restart` / `/restart` 的最后阶段），`agent.restart_drain_timeout` 决定它在中断进行中的聊天轮次之前允许多少秒继续运行：

```yaml
agent:
  restart_drain_timeout: 0          # 默认：立即中断进行中的聊天轮次
  cron_drain_timeout: 30            # cron 任务与 api_server（/v1）运行的排空下限
  restart_after_turn_timeout: 1800  # /restart 与 `gateway restart`：在停止之前等待轮次
```

使用默认值 `0` 时，重启会立即中断正在运行的轮次。该轮次以 interrupted 状态结算且不会再次运行；TUI 和桌面端会在该轮次上显示 "Gateway restarting"（或 "Gateway shutting down"）提示，你在 Hermes 重新连接后重发消息即可。排在它后面的消息会被保留，并按顺序在新 gateway 上运行。

如果你更希望让轮次跑完，请设置一个正值：

```yaml
agent:
  restart_drain_timeout: 60
```

更长的排空会让每次重启和更新最多等待同样长的时间。`HERMES_RESTART_DRAIN_TIMEOUT` 会覆盖配置值。

:::caution systemd 的 `TimeoutStopSec`
生成的 systemd unit 会依据该值设定 `TimeoutStopSec`：取 `restart_drain_timeout` 与 `cron_drain_timeout` + 10 s 中的较大者，再加上 30 s 余量，且至少 60 s（`gateway/restart.py`、`resolve_systemd_timeout_stop_sec`）。如果你调高 `restart_drain_timeout`，请重新运行 `hermes gateway install`，以便该 unit 采用新的预算。否则 systemd 会在 gateway 排空的过程中用 SIGKILL 干掉它。launchd 把停止超时限制在 60 s，因此在 macOS 上排空受限于这段时间内能完成的部分。
:::

## 会话卡死监视器（Session Stall Watchdog）

Gateway 运行一个仅通知的卡死监视器（`agent.session_stall_timeout`，默认 `300` 秒，`0` = 禁用）。当一个忙碌的会话存在**待处理的入站后续消息**，且 agent 的共享活动时钟空闲达到该时长时，gateway 会记录一条 WARNING 日志并向用户发送一次性通知：

```
⚠️ Agent session appears stalled (last activity N min ago). Try /new to reset.
```

语义：

- **仅通知。** 监视器绝不会终止当前轮次 —— 对比 `agent.gateway_timeout`（长时间无活动后取消运行）。卡死通知只是告诉您 agent 看起来卡住了，由您决定（`/new`、`/stop` 或继续等待）。
- **每个卡死周期只通知一次。** 待处理入站消息被消化或活动恢复时闩锁清除，因此恢复后再次卡死的会话会再次通知。
- 进度仅来自共享活动快照（工具调用、API 流式进度、压缩心跳）。待处理入站消息是通知门槛，不是进度时钟。

```yaml
agent:
  session_stall_timeout: 300   # 秒；0 禁用监视器
```

## 重连关注升级 {#reconnect-attention-escalation}

当某个平台适配器连接失败时（网络中断、bot token 被撤销、sidecar 损坏），gateway 会以有上限的指数退避无限重试——重试永不停止，因此瞬时中断总能自行恢复，无需运维介入。代价是*永久性*失败（Telegram token 被撤销、缺少 Discord privileged intents）看起来和一次抖动完全一样：永远显示 "retrying"。

有两种机制让永久性失败可见：

- **终态分类。** 异常*类型*证明永远无法自愈的失败——被拒绝/撤销的 token（`telegram_auth_error`、`discord_auth_error`、`email_auth_error`）、缺少 privileged intents（`discord_intents_required`）、依赖无法安装（`SIDECAR_DEPS_MISSING`）或 node 二进制缺失（`SIDECAR_NODE_MISSING`）的 Photon sidecar——会被标记为致命，而不是进入重试队列。分类严格基于类型；有歧义的错误一律继续重试。
- **需要关注升级。** 某个平台持续留在重试队列中超过 `agent.reconnect_attention_after`（默认 `7200` 秒 = 2 小时，`0` 禁用）后，会在 gateway 运行时状态（`hermes status`）中得到 `needs_attention: true` 和 `retrying_since` 时间戳，并记录一条 WARNING 日志。重试行为不变——这是一个信号，不是熔断器。该标志在成功重连后清除。

```yaml
agent:
  reconnect_attention_after: 7200   # 秒；0 禁用该升级标志
```

## Gateway Agent 缓存 {#gateway-agent-cache}

gateway 为每个会话保留一个 agent，使对话复用其缓存的提示前缀，而不是每轮重建系统提示。该缓存的 agent 还持有会话的完整转写——包括工具输出，在一个有上百次工具调用的会话上可达数十 MB。因此在繁忙的多平台 gateway 上，该缓存是进程中内存占用的最大单一来源。

```yaml
agent:
  agent_cache:
    max_size: 128            # LRU 条目上限
    idle_ttl_secs: 3600      # 淘汰空闲这么久的 agent
    memory_high_mb: auto     # 匿名 RSS 预算；数字、"auto" 或 0/off
    max_evictions_per_pass: 16
    protect_recent: 8
```

`max_size` 和 `idle_ttl_secs` 分别按数量和按时间限制缓存。两者都不知道自己占了多少字节，因此 `memory_high_mb` 增加了第三重限制：一旦匿名内存越过预算，就丢弃最近最少使用的转写，这些转写会在下一轮从已存储的会话中重新加载。如果 gateway 正在与其他服务争抢内存，就调低它；如果你更愿意让所有前缀保持热态，就调高它（或设为 `0` 关闭该轮清理）。

`auto` 从 gateway 实际运行所遵循的内存限制推导预算——容器或 systemd unit 的 cgroup 限制，否则为总 RAM——因此 unit 上的 `MemoryMax`/`MemoryHigh` 会被尊重，无需再维护第二个数字。在这种限制下，测量也以同样方式限定范围：cgroup 自身的匿名内存计费（`memory.stat` 的 `anon`），其中包含 `execute_code` 内核和终端命令这类计入该 unit 限制的子进程。没有上限时，测量的是 gateway 自身的匿名 RSS。

进行中的轮次、`protect_recent` 个最近使用的会话，以及任何转写尚未写完落盘的会话永不会被丢弃。淘汰会以 WARNING 级别记录测得的 RSS 和被丢弃的会话：

```
Agent cache pressure: anon RSS 6802MB over budget 6656MB — evicting 5 LRU session(s): ...
```

## 上下文引擎

上下文引擎控制在接近模型 token 限制时如何管理对话。内置的 `compressor` 引擎使用有损摘要（参见[上下文压缩](../developer-guide/context-compression-and-caching.md)）。插件引擎可以用替代策略替换它。

```yaml
context:
  engine: "compressor"    # 默认 —— 内置有损摘要
```

使用插件引擎（例如，用于无损上下文管理的 LCM）：

```yaml
context:
  engine: "lcm"          # 必须与插件名称匹配
```

插件引擎**永远不会自动激活** —— 您必须将 `context.engine` 显式设置为插件名称。可用引擎可以通过 `hermes plugins` → Provider Plugins → Context Engine 浏览和选择。

有关内存插件的类似单选系统，请参阅[内存 Providers](./features/memory-providers.md)。

## 迭代预算

当 agent 在处理具有许多工具调用的复杂任务时，它可能会耗尽其迭代预算（默认：500 轮）。Hermes **不会**在任务中途注入压力警告 —— 早期版本会在预算达到 70%/90% 时警告模型，这会导致模型过早放弃复杂任务，该机制已于 2026 年 4 月移除。

取而代之的是，当预算真正耗尽（500/500）时，Hermes 注入一条消息要求模型收尾，并允许一次**宽限调用**以便其给出最终响应。如果该宽限调用仍未产生文本，则会要求 agent 总结已完成的工作。

```yaml
agent:
  max_turns: 500               # 每次对话轮次的最大迭代次数（默认：500）
  api_max_retries: 3           # 回退启动前每个 provider 的重试次数（默认：3）
```

当迭代预算完全耗尽时，CLI 向用户显示通知：`⚠ Iteration budget reached (500/500) — response may be incomplete`。

`agent.api_max_retries` 控制 Hermes 在回退 provider 切换启动**之前**对瞬时错误（速率限制、连接断开、5xx）重试 provider API 调用的次数。默认为 `3` —— 总共四次尝试。如果您配置了[回退 providers](./features/fallback-providers.md) 并希望更快地故障转移，请将其降至 `0`，这样主 provider 上的第一个瞬时错误会立即切换到回退，而不是对不稳定的端点进行重试。

### API 超时

Hermes 对流式传输有单独的超时层，以及用于非流式调用的陈旧检测器。陈旧检测器仅在您将其保留为隐式默认值时才会自动调整本地 provider。

| 超时 | 默认值 | 本地 providers | 配置/环境变量 |
|---------|---------|----------------|--------------|
| Socket 读取超时 | 120s | 自动提升至 1800s | `HERMES_STREAM_READ_TIMEOUT` |
| 陈旧流检测 | 180s | 自动禁用 | `HERMES_STREAM_STALE_TIMEOUT` |
| 陈旧非流检测 | 300s | 保持隐式时自动禁用 | `providers.<id>.stale_timeout_seconds` 或 `HERMES_API_CALL_STALE_TIMEOUT` |
| API 调用（非流式） | 1800s | 不变 | `providers.<id>.request_timeout_seconds` / `timeout_seconds` 或 `HERMES_API_TIMEOUT` |

**Socket 读取超时**控制 httpx 等待 provider 下一个数据块的时间。本地 LLM 在大上下文上预填充可能需要几分钟才能产生第一个 token，因此当 Hermes 检测到本地端点时，会将此值提升至 30 分钟。如果您显式设置 `HERMES_STREAM_READ_TIMEOUT`，无论端点检测如何，始终使用该值。

**陈旧流检测**终止接收 SSE 保活 ping 但没有实际内容的连接。对于本地 providers，这完全禁用，因为它们在预填充期间不发送保活 ping。

**陈旧非流检测**终止长时间没有响应的非流式调用。默认情况下，Hermes 在本地端点上禁用此功能，以避免长时间预填充期间的误报。如果您显式设置 `providers.<id>.stale_timeout_seconds`、`providers.<id>.models.<model>.stale_timeout_seconds` 或 `HERMES_API_CALL_STALE_TIMEOUT`，即使在本地端点上也会遵守该显式值。

周期性的提供商等待提示仅在至少 **60 秒没有响应活动**后显示。Codex Responses 的**等待状态**描述的是没有流事件的时间，而不是生成的总时长：持续收到流事件（包括推理内容）时不会显示等待警告。如果事件停止，提示会显示多久没有收到流事件，而不会声称尚未收到任何响应；事件恢复后提示会清除。当重连进入新的首事件看门狗阶段时，等待状态也会跟随该阶段。这只影响状态显示，不会延长独立的调用总时长限制，也不会更改看门狗超时。Chat Completions 流同样会在数据块恢复时及时清除无输出警告，但不会覆盖本地模型加载状态。

### 禁用 API 流式传输 {#disabling-api-streaming}

`model.streaming: false` 会为整个会话强制使用非流式请求 —— 父 agent 与 subagent 都一样。它是给自托管 OpenAI 兼容服务器的逃生通道：这些服务器的*流式*工具调用路径是坏的（例如 vLLM 配合 `--tool-call-parser qwen3_xml` 和推理解析器时，工具调用标记可能泄漏进纯文本并返回零个 `tool_calls`，导致委派任务静默空转）。默认 `true`；除非你遇到这类 bug，否则请保持默认，因为非流式调用会失去上文描述的活性属性。这与 `display.streaming` 无关，后者只控制终端中的 token 渲染。

当流式无法推进时，Hermes 也会自行把会话切到非流式：provider 报告不支持流式，或某个 OpenAI 兼容网关用无内容的 SSE 帧（只有 `data:` / `event: ping` 保活、没有载荷，常见于降级的中继）回应流式请求。该轮次会在无流式的情况下重试，显示警告，并且该会话剩余时间保持流式关闭。

```yaml
model:
  streaming: false
```

## 墙钟运行预算 {#wall-clock-run-budget}

独立于迭代预算，你可以为每次会话运行设置一个可选的**墙钟**预算。它面向在外部硬上限下运行的一次性调用与 eval-harness 调用（例如每个任务 900 秒的限制）：没有它，一次运行可能在活儿基本干完时超时——差一次生成就能输出最终答案，或者卡在一个挂起的 provider 调用上。

```yaml
agent:
  run_budget_seconds: null     # 可选；未设置/null = 该功能完全关闭（默认）
```

或通过 CLI 按次调用：

```bash
hermes chat --run-budget 850 -q "..."
```

设置预算后，会发生两件事：

1. **80% 时的收尾提示。** 当预算用掉 80% 时，Hermes 会注入一条**一次性**提示（以缓存安全的方式投递，附加到最新的工具结果上，类似 `/steer` 消息），告知模型停止新的发现/验证工作，并基于已有状态产出最终交付物。每次运行最多触发一次，机制与现有的迭代预算收尾一致——不会有重复的压力警告。
2. **按截止时间缩放的陈旧超时。** 隐式的非流式陈旧超时（90 秒默认值以及推理模型的保底值，例如 DeepSeek 推理模型的 600 秒）会被限制在 `max(60, 剩余预算 × 0.5)`，因此一次静默挂起的 provider 调用永远无法吃掉运行的剩余时间。该上限只会*收紧*超时——绝不上调——而显式配置的 `stale_timeout_seconds`（provider/模型配置或 `HERMES_API_CALL_STALE_TIMEOUT`）始终原样优先。

该预算按 `run_conversation` 轮次计算（每条用户消息都会重置），未设置时该功能完全休眠——不读时钟、不注入、不改超时。

## 停止前验证（编码验证） {#verify-on-stop-coding-verification}

启用后，如果 agent 在工作区编辑了代码却没有产生新的验证证据（一次通过的测试运行、构建、lint 等），Hermes 会拒绝在该轮次接受最终答案——它会注入一条合成追问，要求 agent 进行验证或解释为什么不能。纯文档/markdown/技能编辑不会触发它，且该循环有界，绝不会困住 agent。

```yaml
agent:
  verify_on_stop: false        # true | false | "auto"（感知表面：CLI/TUI/桌面端开启，消息平台关闭）
  verify_guidance: true        # 在缺少证据的提醒中附加创意 UI / 干净 diff 的指导
  max_verify_nudges: 3         # 每轮连续 continue 提醒的上限（内置 + pre_verify 钩子）
  coding_instructions: ""      # 追加到编码简报的项目级长期编码规则
```

`verify_on_stop` 接受 `true`（处处开启）、`false`（关闭——默认）或 `"auto"`（旧版感知表面行为：交互式编码表面——CLI、TUI、桌面端——以及程序化调用者开启；Telegram/Discord 这类消息表面关闭，因为验证叙述在聊天里读起来像噪音）。默认处处关闭：全新安装为 `false`，而配置迁移在既有安装上把它关掉了，所以启用它是显式选择加入。`HERMES_VERIFY_ON_STOP` 环境变量在设置后覆盖配置值。

为该守卫提供依据的证据（跑了哪些 test/lint/build 命令、自那以后编辑了哪些文件）存放在 `~/.hermes/verification_evidence.db`。该账本只在守卫启用期间写入或创建；`verify_on_stop: false` 时不会记录任何内容，已有文件可以随意删除。

如果需要在同一点上加一个用户/插件策略门——用你自己的检查让 agent 继续——请参阅 `pre_verify` 钩子（见 [hooks](./features/hooks.md)）。

## 常驻目标（`/goal`） {#standing-goals-goal}

当常驻目标处于活动状态时，Hermes 会判定每条助手响应是否满足该目标。若不满足，它会把一条继续提示送回同一会话，并持续工作直到目标完成、轮次预算耗尽，或用户暂停/清除它。轮次预算是真正的兜底——判定失败时**放开**（继续），因此一个不稳定的判定器永远不会卡住进展。

```yaml
goals:
  max_turns: 20   # Hermes 自动暂停目标前允许的最大继续轮次（默认：20）
```

`max_turns` 限制一个目标在 Hermes 自动暂停并要求用户 `/goal resume` 之前能驱动多少继续轮次。它可防止判定器的假阴性（目标其实已完成但判定器说继续）以及在模糊或无法达成的目标上无界的模型开销。完整功能见 [目标](./features/goals.md)。

## 上下文压力警告

与迭代预算压力分开，上下文压力跟踪对话距**压缩阈值**有多近 —— 即上下文压缩触发以摘要旧消息的点。这有助于您和 agent 了解对话何时变长。

| 进度 | 级别 | 发生的事情 |
|----------|-------|-------------|
| **≥ 60%** 到阈值 | 信息 | CLI 显示青色进度条；gateway 发送信息通知 |
| **≥ 85%** 到阈值 | 警告 | CLI 显示粗体黄色进度条；gateway 警告压缩即将发生 |

在 CLI 中，上下文压力在工具输出流中显示为进度条：

```
  ◐ context ████████████░░░░░░░░ 62% to compaction  48k threshold (50%) · approaching compaction
```

在消息平台上，发送纯文本通知：

```
◐ Context: ████████████░░░░░░░░ 62% to compaction (threshold: 50% of window).
```

如果自动压缩被禁用，警告会告诉您上下文可能被截断。

上下文压力是自动的 —— 无需配置。它纯粹作为面向用户的通知触发，不修改消息流或向模型上下文注入任何内容。

## 凭据池策略

当您为同一 provider 拥有多个 API 密钥或 OAuth token 时，配置轮换策略：

```yaml
credential_pool_strategies:
  openrouter: round_robin    # 均匀循环使用密钥
  anthropic: least_used      # 始终选择使用最少的密钥
```

选项：`fill_first`（默认）、`round_robin`、`least_used`、`random`。完整文档请参阅[凭据池](./features/credential-pools.md)。

## Prompt 缓存 {#prompt-caching}

当活跃 provider 支持时，Hermes 自动开启跨会话 prompt 缓存 —— 无需用户配置。

对于**原生 Anthropic**、**OpenRouter** 和 **Nous Portal** 上的 Claude，Hermes 在系统提示词和技能块上附加带有 1 小时 TTL（`ttl: "1h"`）的 `cache_control` 断点。在新鲜的一小时内首次发送时按完整输入费率计费；同一小时内任何会话的后续发送以折扣缓存读取费率从缓存中提取。这意味着系统提示词、加载的技能内容以及任何长上下文包含的早期部分在第一个小时内跨 `hermes` 会话和分叉子 agent 被重用。

Qwen Cloud（阿里巴巴 DashScope）上游将缓存 TTL 限制为 5 分钟，因此 Hermes 在那里使用 5 分钟断点 TTL。其他通过第三方的 Claude 路径（AWS Bedrock、Azure Foundry）回退到 provider 自己的缓存默认值。xAI Grok 使用单独的会话固定对话 ID 机制 —— 参阅 [xAI prompt 缓存](../integrations/providers.md#xai-grok--responses-api--prompt-caching)。

不存在禁用此功能的旋钮 —— 缓存始终开启，即使在单轮对话中也能节省费用，因为仅系统提示词就占输入 token 数的相当大比例。

## 辅助模型

Hermes 使用"辅助"模型处理图像分析、浏览器截图分析、会话标题生成和上下文压缩等附带任务。默认情况下（`auxiliary.*.provider: "auto"`），Hermes 将每个辅助任务路由到您的**主聊天模型** —— 与您在 `hermes model` 中选择的相同 provider/模型。您无需配置任何内容即可开始，但请注意，在昂贵的推理模型（Opus、MiniMax M2.7 等）上，辅助任务会增加显著成本。如果您希望无论主模型如何都使用便宜且快速的附带任务，请显式设置 `auxiliary.<task>.provider` 和 `auxiliary.<task>.model`（例如，在 OpenRouter 上使用 Gemini Flash 进行视觉分析）。（网页提取不是辅助任务：`web_extract` 和浏览器快照会确定性截断长内容，并将完整文本存储供 `read_file` 分页读取 —— 不涉及 LLM。）

:::note 为什么 "auto" 使用您的主模型
早期版本将聚合器用户（OpenRouter、Nous Portal）分流到便宜的 provider 端默认值。这令人惊讶 —— 付费购买聚合器订阅的用户会看到不同的模型处理其辅助流量。`auto` 现在对所有人使用主模型，`config.yaml` 中的每任务覆盖仍然优先（见下方[完整辅助配置参考](#full-auxiliary-config-reference)）。
:::

### 交互式配置辅助模型

无需手动编辑 YAML，运行 `hermes model` 并从菜单中选择**"配置辅助模型"**。您将获得交互式的每任务选择器：

```
$ hermes model
→ Configure auxiliary models

[ ] vision               currently: auto / main model
[ ] title_generation     currently: openrouter / google/gemini-3-flash-preview
[ ] compression          currently: auto / main model
[ ] approval             currently: auto / main model
[ ] triage_specifier     currently: auto / main model
[ ] kanban_decomposer    currently: auto / main model
[ ] profile_describer    currently: auto / main model
```

选择任务，选择 provider（OAuth 流程打开浏览器；API 密钥 provider 提示输入），选择模型。更改持久化到 `config.yaml` 中的 `auxiliary.<task>.*`。与主模型选择器相同的机制 —— 无需学习额外语法。

### 仅流式端点 {#stream-only-endpoints}

一些 OpenAI 兼容端点直接拒绝非流式 chat 请求（例如腾讯 Copilot 返回 HTTP 400 `"Non-stream chat request is currently not supported"`）。交互式聊天本来就流式，但辅助任务（标题生成、压缩、视觉）使用非流式调用，会在每次尝试时失败。Hermes 始终把 `copilot.tencent.com` 视为仅流式；对于其他这类端点，请在 `auxiliary.stream_only_base_urls` 下列出 URL 子串：

```yaml
auxiliary:
  stream_only_base_urls:
    - "my-stream-only-proxy.example.com"
```

匹配的辅助调用会以 `stream=True` 发送，数据块（包括工具调用增量）在客户端聚合 —— 其他端点行为不变。

### 视频教程

<div style={{position: 'relative', width: '100%', aspectRatio: '16 / 9', marginBottom: '1.5rem'}}>
  <iframe
    src="https://www.youtube.com/embed/NoF-YajElIM"
    title="Hermes Agent — Auxiliary Models Tutorial"
    style={{position: 'absolute', top: 0, left: 0, width: '100%', height: '100%', border: 0}}
    allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share"
    allowFullScreen
  />
</div>

### 通用配置模式

Hermes 中的每个模型槽位 —— 辅助任务、压缩、回退 —— 使用相同的三个旋钮：

| 键 | 作用 | 默认值 |
|-----|-------------|---------|
| `provider` | 用于认证和路由的 provider | `"auto"` |
| `model` | 请求的模型 | provider 的默认值 |
| `base_url` | 自定义 OpenAI 兼容端点（覆盖 provider） | 未设置 |

当设置 `base_url` 时，Hermes 忽略 provider 并直接调用该端点（使用 `api_key` 或 `OPENAI_API_KEY` 进行认证）。当仅设置 `provider` 时，Hermes 使用该 provider 的内置认证和基础 URL。

辅助任务的可用 providers：`auto`、`main`，以及[provider 注册表](../reference/environment-variables.md)中的任何 provider —— `openrouter`、`nous`、`openai-codex`、`anthropic`、`gemini`、`qwen-oauth`、`zai`、`kimi-coding`、`kimi-coding-cn`、`minimax`、`minimax-cn`、`minimax-oauth`、`deepseek`、`nvidia`、`xai`、`xai-oauth`、`ollama-cloud`、`alibaba`、`bedrock`、`huggingface`、`arcee`、`xiaomi`、`kilocode`、`opencode-zen`、`opencode-go`、`ai-gateway`、`azure-foundry` —— 或您 `custom_providers` 列表中任何命名的自定义 provider（例如 `provider: "beans"`）。

:::tip MiniMax OAuth
`minimax-oauth` 通过浏览器 OAuth 登录（无需 API 密钥）。运行 `hermes model` 并选择 **MiniMax (OAuth)** 进行认证。辅助任务自动使用 `MiniMax-M2.7-highspeed`。参阅 [MiniMax OAuth 指南](../guides/minimax-oauth.md)。
:::

:::tip xAI Grok OAuth
`xai-oauth` 通过浏览器 OAuth 为 SuperGrok 和 X Premium+ 订阅者登录（无需 API 密钥）。运行 `hermes model` 并选择 **xAI Grok OAuth (SuperGrok / Premium+)** 进行认证。相同的 OAuth token 可重用于每个直接到 xAI 的接口（聊天、辅助任务、TTS、图像生成、视频生成、转录）。参阅 [xAI Grok OAuth 指南](../guides/xai-grok-oauth.md)，如果 Hermes 在远程主机上，请参阅 [SSH/远程主机上的 OAuth](../guides/oauth-over-ssh.md)。
:::

:::warning `"main"` 仅用于辅助任务
`"main"` provider 选项表示"使用我的主 agent 使用的任何 provider" —— 它仅在 `auxiliary:`、`compression:` 和 `fallback_model:` 配置中有效。它**不是**顶级 `model.provider` 设置的有效值。如果您使用自定义 OpenAI 兼容端点，请在 `model:` 部分设置 `provider: custom`。所有主模型 provider 选项请参阅 [AI Providers](../integrations/providers.md)。
:::

如果端点直接拒绝推理字段（例如 OpenAI 兼容中继后面的纯聊天模型返回 `400 Unrecognized request argument supplied: reasoning_effort`），辅助调用会去掉所有推理字段重试一次，因此该任务（例如会话标题）仍会以端点的默认行为完成。

**后台审查有所不同：** 与主会话使用同一模型的审查分支始终继承主会话的推理强度；`auxiliary.background_review.reasoning_effort` 在这条路径上不会生效，即使显式指定了主会话的 provider/model 也一样。推理设置、系统 prompt、完整会话快照和工具定义保持逐字节一致，以复用 prompt 缓存前缀。没有用于同模型审查的独立推理强度开关。详见[同模型审查的推理强度](./features/memory.md#same-model-review-reasoning)。路由到其他模型时的独立问题见 [#94825](https://github.com/NousResearch/hermes-agent/issues/94825)。

### 完整辅助配置参考 {#full-auxiliary-config-reference}

```yaml
auxiliary:
  # 图像分析（vision_analyze 工具 + 浏览器截图）
  vision:
    provider: "auto"           # "auto"、"openrouter"、"nous"、"codex"、"main" 等
    model: ""                  # 例如 "openai/gpt-4o"、"google/gemini-2.5-flash"
    base_url: ""               # 自定义 OpenAI 兼容端点（覆盖 provider）
    api_key: ""                # base_url 的 API 密钥（回退到 OPENAI_API_KEY）
    timeout: 120               # 秒 —— LLM API 调用超时；视觉负载需要宽裕的超时
    download_timeout: 30       # 秒 —— 图像 HTTP 下载；慢速连接请增加

  # 危险命令审批分类器
  approval:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30                # 秒

  # 上下文压缩超时（与 compression.* 配置分开）
  compression:
    timeout: 120               # 秒 —— 压缩摘要长对话，需要更多时间
    # fallback_chain:           # 可选 —— 发生速率限制/连接故障时尝试的 provider
    #   - provider: nous
    #     model: deepseek/deepseek-chat
    #   - provider: openrouter
    #     model: google/gemini-2.5-flash
    #     base_url: ""
    #     api_key: ""

  # 技能中心 —— 技能匹配和搜索
  skills_hub:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30

  # MCP 工具调度
  mcp:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 30

  # Kanban 分类规格说明器 —— `hermes kanban specify <id>`（或
  # 仪表板上 Triage 列卡片的 ✨ Specify 按钮）使用此
  # 槽位将单行描述扩展为具体规格并将
  # 任务提升到 `todo`。便宜快速的模型在这里效果很好；规格扩展
  # 很短，不需要推理深度。
  triage_specifier:
    provider: "auto"
    model: ""
    base_url: ""
    api_key: ""
    timeout: 120
```

:::tip
每个辅助任务都有可配置的 `timeout`（秒）。默认值：vision 120s、approval 30s、compression 120s。如果您为辅助任务使用慢速本地模型，请增加这些值。Vision 还有单独的 `download_timeout`（默认 30s）用于 HTTP 图像下载 —— 对于慢速连接或自托管图像服务器，请增加此值。
:::

:::info
上下文压缩有自己的 `compression:` 块用于阈值，以及 `auxiliary.compression:` 块用于模型/provider 设置 —— 参阅上方的[上下文压缩](#context-compression)。主备用链使用顶层的 `fallback_providers:` 列表 —— 参阅[备用提供商](../integrations/providers.md#fallback-providers)。三者都遵循相同的 provider/model/base_url 模式。
:::

### 辅助任务的每任务回退链

每个辅助任务都可以选择性地定义一个 `fallback_chain` —— 一个 provider/model 条目列表，当主要辅助 provider 因速率限制、网络连接问题或付费限制而失败时，Hermes 会尝试使用该列表：

```yaml
auxiliary:
  compression:
    provider: openrouter
    model: openai/gpt-4o-mini
    fallback_chain:
      - provider: nous
        model: deepseek/deepseek-chat
      - provider: openrouter
        model: google/gemini-2.5-flash
```

当主要辅助 provider（`openrouter` / `openai/gpt-4o-mini`）返回速率限制、连接超时或需要付费错误时，Hermes 将依次遍历 `fallback_chain`。它会跳过 provider 与已失败 provider 相同的条目，并尝试每个剩余条目，直到有一个成功或该链耗尽。如果所有回退都失败，Hermes 会回退到主 agent 模型作为最终的安全网。

每个条目支持与任何辅助任务配置相同的三个旋钮：

| 键 | 描述 |
|-----|-------------|
| `provider` | Provider 名称（`nous`、`openrouter`、`anthropic`、`gemini`、`main` 等） |
| `model` | 该 provider 的模型名称 |
| `base_url` | （可选）自定义 OpenAI 兼容端点 |

`fallback_chain` 适用于任何辅助任务 —— `compression`、`vision`、`approval`、`skills_hub`、`mcp` 等。

### 限制辅助并发 {#limiting-auxiliary-concurrency}

`max_concurrency` 限制整个进程中 `compression`、`title_generation` 这类辅助任务的在飞 LLM 调用数。`auxiliary.vision.max_concurrency` 除外：它本来就只控制视觉的 CPU 密集型图像编码/缩放 worker，而不是 LLM 请求。这在以下情况最有用：

- 许多会话可能同时派生后台工作（Discord/Telegram 频道、多个终端）
- 你的 provider 受限速或正处于故障，重试会放大突发流量

默认不限。典型的安全上限是 `2`：

```yaml
auxiliary:
  title_generation:
    max_concurrency: 2
  compression:
    max_concurrency: 2
```

信号量包裹整个调用（包括重试和回退），因此一次缓慢的调用只占用一个名额。

### OpenRouter 路由和辅助任务的 Pareto Code

当辅助任务解析到 OpenRouter（显式或通过 `provider: "main"` 而您的主 agent 在 OpenRouter 上）时，主 agent 的 `provider_routing` 和 `openrouter.min_coding_score` 设置**不会传播** —— 按设计，每个辅助任务是独立的。要为特定辅助任务设置 OpenRouter provider 偏好或使用 [Pareto Code 路由器](../integrations/providers.md#openrouter-pareto-code-router)，请通过 `extra_body` 按任务设置：

```yaml
auxiliary:
  compression:
    provider: openrouter
    model: openrouter/pareto-code         # 为此任务使用 Pareto Code 路由器
    extra_body:
      provider:                            # OpenRouter provider 路由偏好
        order: [anthropic, google]         # 按此顺序尝试这些 providers
        sort: throughput                   # 或 "price" | "latency"
        # only: [anthropic]                # 限制到特定 provider
        # ignore: [deepinfra]              # 排除特定 providers
      plugins:                             # OpenRouter Pareto Code 路由器旋钮
        - id: pareto-router
          min_coding_score: 0.5            # 0.0–1.0；越高 = 更强的编码能力
```

形状与 OpenRouter 在聊天补全请求体中接受的内容一致。Hermes 原样转发整个 `extra_body`，因此 [openrouter.ai/docs](https://openrouter.ai/docs) 中记录的任何其他 OpenRouter 请求体字段都以相同方式工作。

### 更改视觉模型

使用 GPT-4o 而非 Gemini Flash 进行图像分析：

```yaml
auxiliary:
  vision:
    model: "openai/gpt-4o"
```

或通过环境变量（在 `~/.hermes/.env` 中）：

```bash
AUXILIARY_VISION_MODEL=openai/gpt-4o
```

### Provider 选项

这些选项适用于**辅助任务配置**（`auxiliary:`、`compression:`、`fallback_model:`），而非您的主 `model.provider` 设置。

| Provider | 描述 | 要求 |
|----------|-------------|-------------|
| `"auto"` | 最佳可用（默认）。Vision 尝试 OpenRouter → Nous → Codex。 | — |
| `"openrouter"` | 强制 OpenRouter —— 路由到任何模型（Gemini、GPT-4o、Claude 等） | `OPENROUTER_API_KEY` |
| `"nous"` | 强制 Nous Portal | `hermes auth` |
| `"codex"` | 强制 Codex OAuth（ChatGPT 账户）。请显式设置 `model`（例如 `gpt-5.4`）。 | `hermes model` → ChatGPT or Codex Subscription |
| `"minimax-oauth"` | 强制 MiniMax OAuth（浏览器登录，无需 API 密钥）。辅助任务使用 MiniMax-M2.7-highspeed。 | `hermes model` → MiniMax (OAuth) |
| `"xai-oauth"` | 强制 xAI Grok OAuth（SuperGrok 或 X Premium+ 订阅者的浏览器登录，无需 API 密钥）。相同的 OAuth token 涵盖聊天、TTS、图像、视频和转录。 | `hermes model` → xAI Grok OAuth (SuperGrok / Premium+) |
| `"main"` | 使用您的活跃自定义/主端点。可以来自 `OPENAI_BASE_URL` + `OPENAI_API_KEY` 或通过 `hermes model` / `config.yaml` 保存的自定义端点。适用于 OpenAI、本地模型或任何 OpenAI 兼容 API。**仅限辅助任务 —— 对 `model.provider` 无效。** | 自定义端点凭据 + 基础 URL |

当您希望附带任务绕过默认路由器时，主 provider 目录中的直接 API 密钥 providers 也在这里工作。配置 `GMI_API_KEY` 后，`gmi` 有效：

```yaml
auxiliary:
  compression:
    provider: "gmi"
    model: "anthropic/claude-opus-4.6"
```

对于 GMI 辅助路由，使用 GMI 的 `/v1/models` 端点返回的确切模型 ID。

### 常见设置

**使用直接自定义端点**（比 `provider: "main"` 对本地/自托管 API 更清晰）：
```yaml
auxiliary:
  vision:
    base_url: "http://localhost:1234/v1"
    api_key: "local-key"
    model: "qwen2.5-vl"
```

`base_url` 优先于 `provider`，因此这是将辅助任务路由到特定端点的最明确方式。对于直接端点覆盖，Hermes 使用配置的 `api_key` 或回退到 `OPENAI_API_KEY`；它不会为该自定义端点重用 `OPENROUTER_API_KEY`。两者都未设置时，只有当 `base_url` 与主端点的源（scheme、主机和端口）完全相同时，才会重用主模型的密钥。主端点是会话当前运行的端点（`/model` 切换后即为切换后的端点），并且只与它自己的密钥配对，绝不会使用另一个端点的密钥。

**使用 OpenAI API 密钥进行视觉：**
```yaml
# 在 ~/.hermes/.env 中：
# OPENAI_BASE_URL=https://api.openai.com/v1
# OPENAI_API_KEY=sk-...

auxiliary:
  vision:
    provider: "main"
    model: "gpt-4o"       # 或 "gpt-4o-mini" 更便宜
```

**使用 OpenRouter 进行视觉**（路由到任何模型）：
```yaml
auxiliary:
  vision:
    provider: "openrouter"
    model: "openai/gpt-4o"      # 或 "google/gemini-2.5-flash" 等
```

**使用 Codex OAuth**（ChatGPT Pro/Plus 账户 —— 无需 API 密钥）：
```yaml
auxiliary:
  vision:
    provider: "codex"     # 使用您的 ChatGPT OAuth token
    model: "gpt-5.4"      # Codex 路径没有隐式默认模型
```

**使用 MiniMax OAuth**（浏览器登录，无需 API 密钥）：
```yaml
model:
  default: MiniMax-M2.7
  provider: minimax-oauth
  base_url: https://api.minimax.io/anthropic
```
运行 `hermes model` 并选择 **MiniMax (OAuth)** 自动登录并设置此项。对于中国区域，基础 URL 将是 `https://api.minimaxi.com/anthropic`。完整演练请参阅 [MiniMax OAuth 指南](../guides/minimax-oauth.md)。

**使用本地/自托管模型：**
```yaml
auxiliary:
  vision:
    provider: "main"      # 使用您的活跃自定义端点
    model: "my-local-model"
```

`provider: "main"` 使用 Hermes 用于普通聊天的任何 provider —— 无论是命名的自定义 provider（例如 `beans`）、内置 provider（如 `openrouter`）还是旧版 `OPENAI_BASE_URL` 端点。

:::tip
如果您使用 Codex OAuth 作为主模型 provider，视觉会自动工作 —— 无需额外配置。Codex 包含在视觉的自动检测链中。
:::

:::warning
**视觉需要多模态模型。** 如果您设置 `provider: "main"`，请确保您的端点支持多模态/视觉 —— 否则图像分析将失败。
:::

### 环境变量（旧版）

辅助模型也可以通过环境变量配置。但是，`config.yaml` 是首选方法 —— 它更易于管理，并支持所有选项，包括 `base_url` 和 `api_key`。

| 设置 | 环境变量 |
|---------|---------------------|
| Vision provider | `AUXILIARY_VISION_PROVIDER` |
| Vision 模型 | `AUXILIARY_VISION_MODEL` |
| Vision 端点 | `AUXILIARY_VISION_BASE_URL` |
| Vision API 密钥 | `AUXILIARY_VISION_API_KEY` |

压缩和回退模型设置仅限 config.yaml。（`AUXILIARY_WEB_EXTRACT_*` 变量已废弃 —— 网页提取不再使用辅助 LLM。）

:::tip
运行 `hermes config` 查看您当前的辅助模型设置。覆盖仅在与默认值不同时显示。
:::

### 原生视觉嵌入预算（顶层 `vision:`）

与 `auxiliary.vision`（选择描述器模型）不同：当*主*模型支持视觉时，`vision_analyze` 和浏览器截图会把真实像素嵌入工具结果，并在之后的每一轮请求中重复发送。`vision.embed_target_bytes`（默认 `262144`，限制在 64 KiB..4 MiB 之间）控制单次嵌入的大小；`vision.max_calls_per_image` 限制同一张图片在一个会话中可被嵌入的次数（未设置 = 委派子代理内为 3，主代理不限；`0` = 不限）。参见 [视觉 → 原生嵌入伴随整个会话](./features/vision.md#原生嵌入伴随整个会话visionembed_target_bytes-与-visionmax_calls_per_image)。

## 推理努力程度

控制模型在响应前进行多少"思考"：

```yaml
agent:
  reasoning_effort: ""   # 空 = 中等。选项：none、minimal、low、medium、high、xhigh、max、ultra
```

未设置时（默认），推理努力程度默认为"medium" —— 适合大多数任务的平衡级别。设置值会覆盖它 —— 更高的推理努力程度在复杂任务上提供更好的结果，但代价是更多 token 和延迟。

### 回答长度（`text_verbosity`）

Responses-API 模型（OpenAI GPT-5 系列及更新、OpenAI 直连、ChatGPT Codex 与 Azure 路由）还接受一个独立的旋钮，控制最终自然语言回答的长度，与推理深度无关：

```yaml
agent:
  text_verbosity: ""   # 空 = 不发送（provider 默认值）。选项：low、medium、high
```

Hermes 仅在 Responses 家族路由上把它作为顶层 `text: {verbosity: ...}` 字段发送；它绝不会出现在 `chat_completions`、Anthropic 或 xAI 请求上，空值或未知值则什么都不发。通过 `request_overrides` 设置的结构化输出（`text.format`）会原样传递。

:::note 经 OpenRouter 的自适应思考模型（Claude 4.6+、Fable/Mythos 级）
这些模型使用*自适应*思考，不接受常见的 `reasoning.effort` 字段 —— OpenRouter 对它们忽略该字段。Hermes 会透明地把你的 `reasoning_effort` 路由到 OpenRouter 的 `verbosity` 参数（映射到 Anthropic 的 `output_config.effort`），因此同一个努力程度旋钮在所选模型支持的级别上继续工作。`none`（或未设置）让模型保持自己的自适应默认值。原生 Anthropic provider 直接控制努力程度，不受影响。
:::

:::note OpenRouter 模型与支持的努力级别
对于经 OpenRouter 路由的其他模型，Hermes 读取实时模型目录的推理元数据（`supported_parameters` 加每个模型的 `reasoning.supported_efforts`）来决定是否发送推理控制，并把请求的努力程度钳制到该路由实际支持的最接近级别（始终向下 —— 例如在止步于 `high` 的路由上，`ultra` 会变成 `high`，绝不会静默升级）。新的支持推理的厂商无需等待 Hermes 更新即可工作；当目录不可达或模型未列出时，Hermes 回退到内置的模型家族列表，并原样传递你的努力程度。
:::

:::note `ultra` 会被钳制到路由接受的最强级别
`ultra` 是 Hermes 内部的阶梯步：没有任何 provider 线路接受它，因此每条路由都会把它钳制到自己的最强级别（GPT-5.6 Codex 和 OpenAI 兼容路由为 `max`，更老的 Codex 模型为 `xhigh`）。努力程度选择器和 `/reasoning` 状态会显示 `ultra (sends max on this route)`，因此你看到的级别就是实际发送的级别。
:::

您也可以在运行时使用 `/reasoning` 命令更改推理努力程度：

```
/reasoning                # 显示当前努力程度和显示状态
/reasoning high           # 将推理努力程度设为 high（仅本次会话）
/reasoning high --global  # 设置并持久化到 config.yaml
/reasoning none           # 禁用推理（仅本次会话）
/reasoning show           # 在每次响应上方显示模型思考
/reasoning hide           # 隐藏模型思考
```

努力程度更改默认作用域为当前会话；加上 `--global` 可把新级别保存为 `agent.reasoning_effort` 默认值。

#### 按模型覆盖推理设置

可以为不同模型设置不同的推理努力程度。当你希望对复杂模型用高推理、对更快的模型用中等推理时很有用：

```yaml
agent:
  reasoning_effort: "medium"       # 全局默认
  reasoning_overrides:
    "openrouter/anthropic/claude-opus-4.5": "xhigh"
    "openai/gpt-5": "low"
    "claude-sonnet-4.6": "high"    # 裸模型名同样有效
```

键的匹配是**容忍拼写差异的** —— 任何合理的拼写都能匹配：
- `claude-opus-4.5`、`claude-opus-4-5`、`claude-opus.4.5`（点和横线可互换）
- `anthropic/claude-opus-4.5`、`openrouter/anthropic/claude-opus-4.5`（provider 前缀可选）
- 以具名自定义 provider 为前缀的键（`ollama-local/qwen3.6:27b-q4_k_m`）在请求只携带裸模型 id（`qwen3.6:27b-q4_k_m`，即回退条目和 `providers:` 路由所发送的形式）时同样适用
- 精确匹配优先于各种变体

#### 自定义推理档位名称

一些 OpenAI 兼容端点提供标准阶梯之外的思考档位（例如提供 `fast`/`thinking` 而非 `low`…`max` 的中继）。阶梯之外的裸字符串会被拒绝并提示 `Unknown reasoning_effort '<value>', using default (medium)`，因此笔误永远到不了线路。要请求 provider 自己的档位名，请使用显式的字典形式 —— `effort` 值会逐字作为顶层 `reasoning_effort` 字段发送：

```yaml
agent:
  reasoning_effort:
    enabled: true
    effort: thinking            # 原样发送
  reasoning_overrides:
    "my-relay/lumo-max":        # 字典形式也可以按模型设置
      enabled: true
      effort: fast
```

字典形式中的 `enabled: false` 会关闭思考，等同于 `reasoning_effort: none`。

字典形式只能通过直接编辑 `config.yaml` 设置：`/reasoning` 菜单、`hermes model` 和仪表盘的辅助模型选择器只提供标准阶梯（配置后，TUI 状态和设置向导仍会显示自定义档位名）。

:::note
模型 id 含点（`claude-opus-4.5`、`qwen3.6:27b`），`hermes config set` 会把点当作嵌套分隔符。用反斜杠转义即可写入字面键 —— `hermes config set 'agent.reasoning_overrides.ollama-local/qwen3\.6:27b-q4_k_m' low` —— 或直接编辑 YAML。参见[键名中的点](../reference/cli-commands.md)。
:::

:::note OpenAI Responses（`openai-api`、`openai-codex`）
在接受该字段的模型上（GPT-5.x），`reasoning_effort: none` 会被显式发送为 `reasoning.effort: "none"`：省略该字段会让模型保持默认努力程度（GPT-5.6 默认为 `medium`）。只有未设置努力程度才是省略该字段的唯一状态。`api.openai.com` 上的 Chat 时代模型（`gpt-4o`、`gpt-4.1` 及其 `-mini` 变体和微调版本）拒绝任何 `reasoning` 参数，因此 Hermes 对它们一律不发送，无论配置的努力程度如何，而不是以 `400 Unsupported parameter: 'reasoning.effort'` 失败。如果模型拒绝 `none`，Hermes 会给出警告，为该会话放弃禁用，并以模型默认值重试。
:::

:::note 本地 OpenAI 兼容端点
自定义 `base_url`（`http://localhost:11434/v1`、vLLM、SGLang 或 router 端点）会以标准顶层 `reasoning_effort` 请求字段收到解析后的努力程度 —— `agent.reasoning_effort` 或匹配的按模型覆盖 —— 并钳制到 OpenAI 兼容线路接受的值（`none`、`minimal`、`low`、`medium`、`high`、`xhigh`、`max`）。此处未设置的努力程度也会以 `medium` 发送，与 Nous Portal 和 OpenRouter 路由应用的默认值相同 —— 省略该字段等于把选择权交给端点，而托管推理模型自己的默认值可能就是它的上限（kimi-k3 默认为 `max`：推理 token 和延迟约为 `medium` 的 3 倍）。对于目录或 `model_overrides` 标记为 `supports_reasoning: false` 的模型、未带 `thinking` 能力拉取的本地 Ollama 模型，以及端点对该字段返回 `400` 之后的会话剩余时间，该字段保持关闭。嵌套的 `reasoning` 对象只保留给已知接受它的端点（Nous Portal、支持推理的 OpenRouter 模型、GitHub Models），因为任意服务器会用 HTTP 400 拒绝未知字段。如果你的服务器从不同字段读取思考预算（Ollama 的 `think`、vLLM 的 `chat_template_kwargs`、某个 router 专用键），请把它设置在自定义 provider 的 [`extra_body`](../integrations/providers.md) 下，该字段会合并进路由到那里的每个请求。
:::

**解析优先级：**

1. 会话级的 `/reasoning --session` 覆盖（仅 gateway）
2. `agent.reasoning_overrides` 中的按模型覆盖（容忍拼写差异）
3. 全局 `agent.reasoning_effort`
4. provider 默认值

该覆盖会自动在所有场景生效：CLI 启动、`hermes -p` 一次性运行、消息 gateway、Desktop/TUI、ACP 会话、cron 任务、会话中途的 `/model` 切换（包括在第一条消息之前发出的切换）、会话恢复（`--resume`、`/resume`）、`/new`，以及回退模型激活。

## Fast 模式 {#fast-mode}

Fast 模式以更高的价格向 provider 请求更快的输出：OpenAI 的 [Priority Processing](https://openai.com/api-priority-processing/)（`service_tier: priority`）以及在受支持的 OpenAI 模型上的 Ultrafast（`service_tier: ultrafast`），Grok 4.6 上的 xAI Priority Processing，以及 Anthropic 的 [Fast Mode](https://platform.claude.com/docs/en/build-with-claude/fast-mode)（`speed: fast`，仅限 Opus 4.8 / Opus 5 / Opus 5.5）。`openai` 与 `openai-api` 两个 provider 走 OpenAI 第一方端点。它**默认关闭**。

```yaml
agent:
  service_tier: ""          # "" / normal | fast | priority | ultrafast | auto | cold
  fast_auto_seconds: 60     # auto / cold 的窗口
```

| 模式 | 何时发送 fast 参数 | 适用场景 |
|------|---------------------------|------------|
| `normal`（默认，`""`） | 从不 | 最便宜；标准延迟 |
| `fast` | 每个请求 | 你总是想要速度的长交互会话 |
| `auto` | **每一**轮开头 `fast_auto_seconds` 内的请求 | 首条回复更利落；长时间工具循环回落到标准计费 |
| `cold` | 同一窗口，但仅限会话的**第一轮**（无历史） | 快速的入驻回复，之后按标准计费 |

`/fast normal|fast|ultrafast|auto|cold` 为当前会话切换模式（`/fast ultra` 是 `ultrafast` 的简写）。在 ChatGPT/Codex 订阅下，只有当你账号的 Codex 模型目录为当前模型列出了 Ultrafast 时才会提供该模式；否则后端仍会接受请求并按标准速度服务。加上 `--global` 可持久化到 `config.yaml`。单独使用 `/fast` 会显示当前模式。

**成本提示：** 两个 provider 都对 fast 请求按标准费率的倍数计费（Anthropic：Opus 5.5 为 $8 / $40 每百万 token 输入/输出，Opus 5 和 Opus 4.8 为 $10 / $50），并与 prompt 缓存价格叠加。Hermes 依据 API 在 `usage.speed` 中报告的速度来为每条 Anthropic 响应计价。`auto`/`cold` 只把这份溢价限制在窗口内。fast 参数只发给支持它们的第一方端点（`api.openai.com` / Codex 订阅、`api.anthropic.com`、`api.x.ai`）；OpenRouter、Nous Portal、Azure、Bedrock 和自定义 `base_url` 路由在任何模式下都不会收到它们。

**Prompt 缓存：** 请求之间只有这个按请求的参数在变；system prompt、工具和消息保持逐字节一致。Anthropic 为每种速度维护独立的 prompt 缓存，因此在 Anthropic 上，每个 `auto`/`cold` 窗口边界都会以新速度重写对话前缀。对较长的 Anthropic 会话，`fast` 或 `normal` 能保持一份热的缓存。

Fast 模式的加速体现在每秒输出 token 数上，因此长回答收益最大。

当某个 Anthropic 组织对某个模型没有 fast 模式容量（API 对 fast 请求返回 fast-mode 上限 0）时，Hermes 会在该会话剩余时间内把该模型切到标准速度并重试该请求。

### 网关或代理后的 Fast 档位 {#fast-tiers-behind-a-gateway-or-proxy}

仅限第一方这条规则是有意为之：fast 档位参数是一条计费指令，Hermes 只会把它发给价格表已知的端点。如果你运行的是暴露自家优先档位的 OpenAI 兼容网关、路由器或代理（自家的 `service_tier` 值，或字段名不同），请通过该 provider 的 `extra_body` 请求它，而不是 `agent.service_tier`。[具名自定义 provider](../integrations/providers.md) 上的 `extra_body` 会合并进路由到该端点的**每一个** chat-completions 请求，能跨 gateway 轮次和 `/fast` 切换持续存在，并在你通过 `/model` 离开该 provider 时再次被丢弃：

```yaml
providers:
  my-gateway:
    api: https://gateway.example.com/v1
    key_env: MY_GATEWAY_KEY
    default_model: fast-lane-model
    extra_body:
      service_tier: priority     # 你的网关文档中的档位值
```

与 `agent.service_tier` 的差别：该档位对该 provider 始终开启（没有 `auto`/`cold` 窗口），`/fast` 不会切换它，Hermes 也不校验该值——接受什么、怎么计费由网关决定。

## 工具使用强制

某些模型偶尔会将预期操作描述为文本而不是进行工具调用（"我会运行测试..."而不是实际调用终端）。工具使用强制会注入系统提示词指导，引导模型实际调用工具。

```yaml
agent:
  tool_use_enforcement: "auto"   # "auto" | true | false | ["model-substring", ...]
```

| 值 | 行为 |
|-------|----------|
| `"auto"`（默认） | 对匹配以下模型启用：`gpt`、`codex`、`gemini`、`gemma`、`grok`。对所有其他模型禁用（Claude、DeepSeek、Qwen 等）。 |
| `true` | 始终启用，无论模型如何。如果您注意到当前模型描述操作而不是执行操作，请使用此选项。 |
| `false` | 始终禁用，无论模型如何。 |
| `["gpt", "codex", "qwen", "llama"]` | 仅当模型名称包含列出的子字符串之一时启用（不区分大小写）。 |

### 注入的内容

启用后，系统提示词中可能会添加三层指导：

1. **通用工具使用强制**（所有匹配模型）—— 指示模型立即进行工具调用而不是描述意图，持续工作直到任务完成，永远不要以未来操作的承诺结束轮次。

2. **OpenAI 执行纪律**（仅限 GPT 和 Codex 模型）—— 针对 GPT 特定失败模式的额外指导：在部分结果上放弃工作、跳过先决条件查找、幻觉而不是使用工具、在未验证的情况下宣布"完成"。

3. **Google 操作指导**（仅限 Gemini 和 Gemma 模型）—— 简洁性、绝对路径、并行工具调用和编辑前验证模式。

这些对用户透明，仅影响系统提示词。已经可靠使用工具的模型（如 Claude）不需要此指导，这就是为什么 `"auto"` 排除它们。

### 何时开启

如果您使用的模型不在默认自动列表中，并注意到它经常描述它*会*做什么而不是实际去做，请设置 `tool_use_enforcement: true` 或将模型子字符串添加到列表中：

```yaml
agent:
  tool_use_enforcement: ["gpt", "codex", "gemini", "grok", "my-custom-model"]
```

## 执行纪律指导 {#execution-discipline-guidance}

与工具使用强制分开，Hermes 会为一些模型族注入一段**执行纪律**（execution-discipline）内容：这些模型族在 eval 轨迹中表现出共同的 agentic 失败模式——用散文而非代码做算术、在外部写入后跳过读回验证、"修复"格式错误的标识符、在计数不匹配的情况下声称完整、以及没有验证每一条验收标准就宣称"完成"。

```yaml
agent:
  execution_guidance: "auto"   # "auto" | true | false | ["model-substring", ...]
```

| 值 | 行为 |
|-------|----------|
| `"auto"`（默认） | 对匹配以下内容的模型启用：`gpt`、`codex`、`grok`、`deepseek`、`kimi`、`qwen`、`glm`、`minimax`、`mimo`、`mistral`、`muse`。 |
| `true` | 无论模型如何，始终启用。 |
| `false` | 无论模型如何，始终禁用。 |
| `["deepseek", "my-custom-model"]` | 仅当模型名包含列表中的某个子串时启用（不区分大小写）。 |

注入的内容涵盖：

- **工具持久性** ——持续调用工具直到任务完成*并*验证；对空、部分或可疑地狭窄的检索结果，用更宽泛或不同的查询重试后再下结论。
- **强制使用工具** ——算术、哈希、日期、系统状态和文件事实一律来自工具，绝不来自心算。
- **外部写入读回** ——对任何外部系统做状态变更写入后，在声称成功之前读回确切的目标（工具已确认的内部文件编辑不重复验证）。
- **计数核对** ——声明的总数（`total`、`reply_count`、`has_more`）是硬断言；不匹配时重新获取或用程序解析。
- **字面保留** ——绝不规范化或"修复"不符合既定格式的标识符；一次成功的查找并不能证明格式错误的源 token 有效。
- **以验证为门槛的完成** ——"完成"意味着每一条具名验收标准都已验证，绝不是看起来合理的子集。

该门槛独立于 `tool_use_enforcement`——两者可以单独开启。指导内容在会话启动时依据模型名选定一次，因此 system prompt 在整个对话期间保持逐字节稳定（对 prompt 缓存友好）。Gemini/Gemma 被排除在 auto 列表之外，因为它们会收到更具体的 Google 运营指导；Claude 被排除是因为它不表现出这些失败模式——任何模型都可以通过 `true` 或子串列表选择加入。

## 工具循环护栏 {#tool-loop-guardrails}

Hermes 能检测 agent 是否卡在无产出的工具调用循环中——同一次工具调用反复失败、同一个工具一次次失败，或者一个幂等调用返回相同结果而毫无进展。默认情况下，它会在工具结果中注入一条**警告**，让模型自我纠正。交互式 CLI、TUI、Desktop 和 ACP 会话保持仅警告，因为有人可以介入；无人值守的 gateway 和 cron 会话默认启用硬停止。

这个感知平台的默认值可以在无人值守部署上关闭，也可以在每个平台上显式启用硬停止：

```yaml
tool_loop_guardrails:
  warnings_enabled: true       # 向工具结果注入警告（默认：true）
  hard_stop_enabled: false     # 超过硬停止阈值时再 BLOCK 该调用（默认：false）
  non_interactive_hard_stop_enabled: true  # gateway/cron 的默认硬停止
  warn_after:
    exact_failure: 2           # 完全相同的失败调用重复 N 次
    same_tool_failure: 3       # 同一个工具失败 N 次（参数不同）
    idempotent_no_progress: 2  # 结果相同、无进展，N 次
  hard_stop_after:
    exact_failure: 5
    same_tool_failure: 8
    idempotent_no_progress: 5
  loop_caps:
    max_web_searches: 50       # 每轮最多 web_search 调用次数（0 = 不限）
    max_subagents: 50          # 每轮最多派生的 subagent 数（0 = 不限）
```

`hard_stop_enabled` 显式在每个平台上启用硬停止。当它保持 `false` 时，`non_interactive_hard_stop_enabled` 仍会为无人值守的 gateway/cron 类平台启用硬停止，同时对 CLI、TUI、Desktop、ACP、subagent 和 `api_server` 运行（有活跃父级或客户端的受监督任务循环）保持仅警告行为。将 `non_interactive_hard_stop_enabled: false` 可让无人值守部署退出该行为。另请参阅 [Docker / 无人值守部署](docker.md)。

硬停止旨在抓住**重放**——完全相同的调用、参数不变、期间什么都没有发生——而不是合理的迭代：

- **编辑 → 重跑永远不算循环。** 任何成功的变更调用（`write_file`、`patch`、返回绿的 `terminal`/`execute_code`、浏览器操作、任务/消息/cron 变更）会为所有仍在计数的失败调用标记进展。下一次相同的重试（修好后再跑一次红测试、点击后重新截图）会开始新的连续计数，而不是继续累积到封锁。
- **不同的红色命令是诊断，不是循环。** 对非零退出属于普通输出的工具（`terminal`、`execute_code`、进程轮询器、`browser_exec`、`web_extract`），`same_tool_failure` 阈值只会警告，永不中止。只有期间没有任何变更的完全相同的参数重放，或结果完全一致的连续重复，才能阻止它们。
- **中止只结束该轮次，不结束会话。** agent 会回复是哪个护栏触发以及原因；回复 "continue" 会以新的按轮计数器继续。

### 每轮失控循环上限 {#per-turn-runaway-loop-caps}

与上面的基于失败的阈值分开，`loop_caps` 为单个 agent 循环（轮次）可以发起的 `web_search` 调用数和 subagent 派生数设置硬上限。计数器在每轮开始时重置，因此合理的多轮会话永远不会被饿死——但单个螺旋成无界搜索或委派循环的轮次会被叫停。这些始终开启，且不论 `hard_stop_enabled` 如何都会触发。单个轮次发出几十次 web 搜索或派生几十个 subagent 本身就是病态的，因此默认值很低。达到上限时，触发的工具调用会被阻止并给出说明消息，轮次干净地停止，而不是烧掉剩余预算。将任一值设为 `0` 可完全禁用该上限。

单次 `delegate_task` 批量中的每个任务都会计入 `max_subagents`（一批 3 个消耗 3），因此该上限跟踪的是真正派生的 subagent 数，而不是 `delegate_task` 的调用次数。

这与 Claude Code 的每会话 WebSearch 与 subagent 上限（v2.1.212）相似，后者默认也是 200 并在 `/clear` 时重置。

### 运行时防卡死守卫 {#runtime-anti-stall-guards}

作为上述基于失败护栏的补充，`agent.stall_guards`（默认 `true`）启用两个保守的运行时守卫，防止浪费轮次。第一，**相同调用循环断路器**：当同一个工具被连续调用 3 次以上且参数完全相同*并且*返回完全相同的结果时，会在该工具结果后附加一行简短提示，告诉模型不要重复该调用——在仅警告会话中它永不阻止调用，且可合理重复的轮询器（`process_manage`、`*_get_result`、`*_poll`）豁免。当硬停止生效时（显式 `hard_stop_enabled`，或无人值守的 gateway/cron 平台），同样的连续计数一旦达到 `hard_stop_after.idempotent_no_progress` 次连续相同调用，也会成为硬停止——对**任何**工具都如此，而不只是 `idempotent_no_progress` 护栏跟踪的只读工具——因此重放同一次成功 `terminal` 或 `skill_view` 调用的模型会被中止，而不是耗尽迭代预算（`identical_call_streak_halt`）。第二，**继续意图恢复**：当模型结束轮次时没有工具调用，但其简短回复以宣告某个动作的话语收尾（"Let me now update the file…"），Hermes 会通过与意图确认恢复相同的受限继续机制重新提示它行动（每轮最多 2 次重提示）。两者都是缓存安全的（提示在构造结果时添加，绝不追溯添加），并且可以一起禁用：

```yaml
agent:
  stall_guards: false
```

同一个开关还会启用**结果引用存根**：当重新发出的相同工具调用返回逐字节相同的新结果时，重复载荷会以指向早先结果的简短引用存根进入上下文（工具名、`tool_call_id`、参数摘要，以及——如果第一个结果被持久化到磁盘——其溢出路径），而不是重复完整输出。工具仍然每次都执行，因此轮询语义得以保留：变化的结果总是完整通过。小于 512 字符的结果、错误结果和多模态结果永不做存根，而轮询器*会*做存根（未变化的轮询正是重复载荷不携带任何信息的情形）。

### 轮次存活看门狗 {#turn-liveness-watchdog}

`agent.turn_liveness` 限制一个对话轮次可以**毫无可观察进展**地持续多久，之后 Hermes 会强制恢复它。该看门狗依据活动时钟（与标记 API 等待、流 token 和工具心跳相同的信号——租约续期从不计入），因此一个在半空中静默卡住的轮次（issue #95548 中的观测：没有工具执行、没有 API 调用、没有错误，但会话无限期保持"忙碌"）会被醒目地暴露、被中断以便以可重试的中断轮次回卷，并且——当中断无法解开卡死时——其持久轮次租约停止续期，使陈旧轮次清理可以回收会话，而不是让它挂到进程被杀。

```yaml
agent:
  turn_liveness:
    timeout_s: 600.0   # 空闲上限；<= 0 禁用看门狗
    poll_s: 15.0        # 采样间隔（秒）
```

合理的慢速工作不会被惩罚：流式响应、工具心跳（工具运行期间每 30s 一次）和审批等待都在不断触碰时钟，因此只有在整个时限内**零**进展的轮次才会触发看门狗。无效值（拼写错误、`NaN`、`Inf`、非正的 `poll_s`）会记录警告并回落到默认值——它们绝不会让启动崩溃，也绝不会静默禁用看门狗。触发的中止会在开始恢复时报告卡死，并且只有当中断真正提交后才发布最终的 aborted/lease-stopped 结果。

## TTS 配置

```yaml
tts:
  provider: "edge"              # "edge" | "elevenlabs" | "openai" | "minimax" | "mistral" | "gemini" | "xai" | "neutts"
  speed: 1.0                    # 全局速度倍数（所有 provider 的回退）
  edge:
    voice: "en-US-AriaNeural"   # 322 种声音，74 种语言
    speed: 1.0                  # 速度倍数（转换为速率百分比，例如 1.5 → +50%）
  elevenlabs:
    voice_id: "pNInz6obpgDQGcFmaJgB"
    model_id: "eleven_multilingual_v2"
  openai:
    model: "gpt-4o-mini-tts"
    voice: "alloy"              # alloy、echo、fable、onyx、nova、shimmer
    speed: 1.0                  # 速度倍数（API 限制为 0.25–4.0）
    base_url: "https://api.openai.com/v1"  # 覆盖 OpenAI 兼容 TTS 端点
  minimax:
    speed: 1.0                  # 语音速度倍数
    # base_url: ""              # 可选：覆盖 OpenAI 兼容 TTS 端点
  mistral:
    model: "voxtral-mini-tts-2603"
    voice_id: "c69964a6-ab8b-4f8a-9465-ec0925096ec8"  # Paul - Neutral（默认）
  gemini:
    model: "gemini-2.5-flash-preview-tts"   # 或 gemini-2.5-pro-preview-tts
    voice: "Kore"               # 30 种预置声音：Zephyr、Puck、Kore、Enceladus 等
  xai:
    voice_id: "eve"             # xAI TTS 声音
    language: "en"              # ISO 639-1
    sample_rate: 24000
    bit_rate: 128000            # MP3 比特率
    # base_url: "https://api.x.ai/v1"
  neutts:
    ref_audio: ''
    ref_text: ''
    model: neuphonic/neutts-air-q4-gguf
    device: cpu
```

这控制 `text_to_speech` 工具和语音模式中的口语回复（CLI 中的 `/voice tts` 或消息 gateway）。

**速度回退层次：** provider 特定速度（例如 `tts.edge.speed`）→ 全局 `tts.speed` → `1.0` 默认值。设置全局 `tts.speed` 以在所有 provider 上应用统一速度，或按 provider 覆盖以进行精细控制。

## 显示设置

```yaml
display:
  tool_progress: all      # off | new | all | verbose
  tool_progress_command: false  # 在消息 gateway 中启用 /verbose 斜杠命令
  platforms: {}           # 每平台显示覆盖（见下文）
  interim_assistant_messages: true  # Gateway：将自然的轮次中 assistant 更新作为单独消息发送
  skin: default           # 内置或自定义 CLI 皮肤（参阅 user-guide/features/skins）
  personality: "kawaii"  # 旧版外观字段，仍在某些摘要中显示
  compact: false          # 紧凑输出模式（减少空白）
  resume_display: full    # full（恢复时显示之前的消息）| minimal（仅单行）
  bell_on_complete: false # 当 agent 完成时播放终端铃声（适合长任务）
  show_reasoning: false   # 在每次响应上方显示模型推理/思考（用 /reasoning show|hide 切换）
  streaming: false        # 将 token 实时流式传输到终端
  show_cost: false        # 在 CLI 状态栏中显示估计 $ 成本
  timestamps: false       # 为 true 时，在 CLI/TUI 记录中为用户和 assistant 标签添加 [HH:MM] 时间戳前缀
  tool_preview_length: 0  # 工具调用预览的最大字符数（0 = 无限制，显示完整路径/命令）
  runtime_footer:         # Gateway：在最终回复中附加运行时上下文页脚
    enabled: false
    fields: ["model", "context_pct", "cwd"]
  file_mutation_verifier: true    # 当本轮 write_file/patch 调用失败时附加建议性页脚
  language: en            # 静态消息的 UI 语言（审批提示、部分 gateway 回复）。en | zh | zh-hant | ja | de | es | fr | tr | uk | af | ko | it | ga | pt | ru | hu
```

### 每轮摘要与 spinner token 流 {#per-turn-summary-and-spinner-token-flow}

`display.turn_summary`（默认 `true`）在**交互式 CLI** 每轮结束后打印一行暗色统计，概述该轮实际做了什么：

```
⋯ 12.4s · edited 2 files +18 -3 · read 4 files · ran 3 commands
```

统计信息从 CLI 本来就接收的工具进度流中观察得到，因此不产生额外开销。细节：

- 墙钟时间是该轮的真实时长（超过一分钟显示为 `2m05s`）。
- 工具调用按动词分组（`edited`、`read`、`ran`、`searched` 等）并正确使用单复数；没有归入动词的插件/MCP 工具会合并为 `called N tools`。
- `+X -Y` 行数增减只在工具结果本身报告了 diff 时出现（目前是 `patch`）。Hermes 从不调用 git 来计算它们，因此 `write_file` 的编辑只计数、不带增减。
- **失败的工具调用不计入** —— 被拒绝的写入永远不会显示为成功的编辑（相应的警告见[文件变更验证器](#file-mutation-verifier)）。
- 长轮次最多四个动词段加一个 `+N more` 尾巴，因此该行永不换行。
- 没有工具调用的快速轮次完全不打印任何内容。

`display.spinner_token_flow`（默认 `true`）把当前轮累计的输出 token 追加到 CLI spinner 的实时计时器上：

```
  ⚡ Reading cli.py  (  2.3s · ↓ 1.2k tok)
```

计数按轮统计（会话总量在轮次开始时作为基线），并随该轮中每次 API 调用报告用量而更新。在首次用量报告到达之前不渲染任何内容，因此你永远不会看到误导性的 `↓ 0 tok`。

两个键都仅用于显示且仅限 CLI：在安静模式、`display.tool_progress` 为 `off` 时、在单查询/`-Q` 批量运行中，以及在 gateway/消息界面上（后者改用 `display.runtime_footer`）都会被抑制。把任一键设为 `false` 即可关闭。

### 文件变更验证器 {#file-mutation-verifier}

当 `display.file_mutation_verifier` 为 `true`（默认）时，每当本轮中 `write_file` 或 `patch` 调用失败且从未被对同一路径的成功写入取代时，Hermes 会在 assistant 的最终响应中附加一行建议。这捕获了"批量并行补丁，一半静默失败，模型总结成功"这类过度声明，而无需您在每次编辑后手动运行 `git status`。

示例页脚：

```
⚠️ File-mutation verifier: 3 file edit(s) FAILED this turn despite any wording above that may suggest otherwise. Run `git status` or `read_file` to confirm what actually landed.
  • concepts/automatic-organization.md — [patch] Could not find match for old_string
  • concepts/lora.md — [patch] Could not find match for old_string
  • concepts/rag-pipeline.md — [patch] Could not find match for old_string
```

设置 `file_mutation_verifier: false`（或 `HERMES_FILE_MUTATION_VERIFIER=0`）以禁止页脚。验证器仅在轮次结束时有真实失败未解决时触发 —— 在同一轮次内重试失败补丁并成功的模型不会为该文件触发它。

### 静态消息的 UI 语言

`display.language` 设置翻译一小组静态面向用户的消息 —— CLI 审批提示、少数 gateway 斜杠命令回复（例如重启排空通知、"审批已过期"、"目标已清除"）。它**不**翻译 agent 响应、日志行、工具输出、错误回溯或斜杠命令描述 —— 这些保持英文。如果您希望 agent 本身用另一种语言回复，只需在您的提示词或系统消息中告诉它。

支持的值：`en`（默认）、`zh`（简体中文）、`zh-hant`（繁体中文）、`ja`（日语）、`de`（德语）、`es`（西班牙语）、`fr`（法语）、`tr`（土耳其语）、`uk`（乌克兰语）、`af`（南非荷兰语）、`ko`（韩语）、`it`（意大利语）、`ga`（爱尔兰语）、`pt`（葡萄牙语）、`ru`（俄语）、`hu`（匈牙利语）。未知值回退到英文。

您也可以使用 `HERMES_LANGUAGE` 环境变量按会话设置，它会覆盖配置值。

```yaml
display:
  language: zh   # CLI 审批提示以中文显示
```

| 模式 | 您看到的内容 |
|------|-------------|
| `off` | 静默 —— 仅最终响应 |
| `new` | 仅在工具更改时显示工具指示器 |
| `all` | 每次工具调用附带简短预览（默认） |
| `verbose` | 完整参数、结果和调试日志 |

在 CLI 中，使用 `/verbose` 循环切换这些模式。要在消息平台（Telegram、Discord、Slack 等）中使用 `/verbose`，请在上方的 `display` 部分设置 `tool_progress_command: true`。该命令将循环切换模式并保存到配置。

### 聚焦视图（`/focus`，CLI + TUI） {#focus-view}

`display.focus_view: true` 启用**聚焦视图** —— 一个精简输出的显示模式，适用于你只想要答案、不想要逐步过程的时候。它是同一套 `tool_progress` 机制上的薄层，而不是第二条抑制路径：

- 打开它会把 `tool_progress` 钉在 `off`，并把之前的模式暂存到 `display.focus_saved_tool_progress`；
- `/focus off` 精确恢复该模式，因此 `/verbose verbose` 的配置能完整往返；
- 每个完成的轮次以一行暗色恢复提示结束 —— `⋯ 7 tool lines hidden · /focus off to show` —— 按*聚焦前*的模式计数，因此它绝不会声称隐藏了你本来就已关闭的行；
- 状态栏中常驻一个 `◉ focus` 徽章（prompt_toolkit CLI 与 Ink TUI 都是），因此精简模式永远不会不可见；
- 聚焦开启时循环 `/verbose` 会把模式交还 `/verbose` 并清除该徽章。

聚焦视图**仅用于显示**。它绝不修改对话历史、系统提示、工具 schema 或任何请求载荷 —— 被隐藏的细节只在屏幕上被抑制，绝不丢弃，prompt 缓存完全不受影响。

### 状态栏字段选择（CLI/TUI） {#status-bar-field-selection}

CLI/TUI 底部的交互式状态栏显示模型、上下文用量、压缩次数、后台活动计数器、计时器和模式徽章。`display.status_bar.fields` 选择其中哪些可见 —— 适用于极简状态栏（只有模型 + 时长），或显示需要主动开启的会话 token 总量：

```yaml
display:
  status_bar:
    fields: ["model", "duration", "total_tokens"]   # 只控制可见性；内置顺序保持不变
```

支持的字段：`model`、`context_detail`（已用/总 token）、`context_pct`（百分比 + 进度条）、`cache_hit`（prompt 缓存命中率 —— 切换模型和压缩时重置）、`latency`（滚动平均 API 延迟，最近 10 次调用）、`tps`（滚动输出 token/秒，最近 10 次调用）、`compressions`、`bg_tasks`、`bg_processes`、`bg_subagents`、`goal`、`git_branch`（⎇ 工作目录的当前 git 分支 —— 仅主动开启，默认永不显示；分离 HEAD 显示缩写提交）、`duration`、`prompt_elapsed`、`idle_since`、`focus`、`yolo`、`stash`、`battery`、`title`（右对齐的会话徽章）以及 `total_tokens`（会话 Σ —— 仅主动开启，默认永不显示）。

注意：

- 空列表（默认）保持标准集合 —— 除 `total_tokens` 和 `git_branch` 之外的一切。
- 配置控制的是**可见性，不是顺序**；字段按其内置位置渲染。
- 窄终端仍会丢弃仅宽屏字段（`context_detail`、`cache_hit`、`latency`、`tps`、`prompt_elapsed`、`idle_since`），无论配置如何（`cache_hit` 在中档 ≥52 列也会显示）。
- `latency`/`tps` 在记录到 API 调用之前保持隐藏（例如 Codex app-server 后端不报告延迟）。
- 这里的 `battery` 与 `title` 可见性会与它们各自的开关（`/battery`、`/title`）叠加 —— 两者都开启时该段才显示。
- 同一个键也会过滤 **Ink TUI** 的状态行（`hermes tui`），其中 `cache_hit`、`latency` 和 `tps` 在 ≥96/104/110 列的终端上分别渲染为宽度受控的尾段（◎ / ◷ / ↑）。
- 仅用于显示：不影响 prompt 缓存或请求载荷。更改在下一会话启动时生效。

### 运行时元数据页脚（仅限 gateway）

当 `display.runtime_footer.enabled: true` 时，Hermes 在每个 gateway 轮次的**最终**消息中附加一个小型运行时上下文页脚。目前页脚可显示模型、上下文窗口百分比和当前工作目录。默认关闭；如果您的团队希望每个回复都包含这些来源信息，请按 gateway 选择加入。

```yaml
display:
  runtime_footer:
    enabled: true
    fields: ["model", "context_pct", "cwd"]   # 支持字段：model、context_pct、cwd
```

`/footer` 斜杠命令在任何会话中运行时切换此功能。

附加到 Telegram/Discord/Slack 回复的示例页脚：

```
— claude-opus-4.7 · 12 tool calls · 2m 14s · $0.042
```

只有轮次的**最终**消息获得页脚；中间更新保持干净。

### 每平台进度覆盖

不同平台有不同的详细程度需求。例如，Signal 无法编辑消息，因此每次进度更新都会成为单独的消息 —— 很嘈杂。使用 `display.platforms` 设置每平台模式：

```yaml
display:
  tool_progress: all          # 全局默认
  platforms:
    signal:
      tool_progress: 'off'    # 在 Signal 上静默进度
    telegram:
      tool_progress: verbose  # 在 Telegram 上详细进度
    slack:
      tool_progress: 'off'    # 在共享 Slack 工作区中保持安静
```

没有覆盖的平台回退到全局 `tool_progress` 值。有效平台键：`telegram`、`discord`、`slack`、`signal`、`whatsapp`、`matrix`、`mattermost`、`email`、`sms`、`dingtalk`、`feishu`、`wecom`、`weixin`、`bluebubbles`、`qqbot`，以及任意插件平台的名称（例如 Home Assistant 插件的 `homeassistant`）。旧版 `display.tool_progress_overrides` 键仍可加载以向后兼容，但已弃用，并在首次加载时迁移到 `display.platforms`。

`interim_assistant_messages` 仅限 gateway。启用后，Hermes 将已完成的轮次中 assistant 更新作为单独的聊天消息发送。这与 `tool_progress` 无关，不需要 gateway 流式传输。

## 隐私

```yaml
privacy:
  redact_pii: false  # 从 LLM 上下文中删除 PII（仅限 gateway）
```

当 `redact_pii` 为 `true` 时，gateway 在将系统提示词发送到受支持平台上的 LLM 之前，会从中删除个人身份信息：

| 字段 | 处理方式 |
|-------|-----------|
| 电话号码（WhatsApp/Signal 上的用户 ID） | 哈希为 `user_<12-char-sha256>` |
| 用户 ID | 哈希为 `user_<12-char-sha256>` |
| 聊天 ID | 数字部分哈希，保留平台前缀（`telegram:<hash>`） |
| 主频道 ID | 数字部分哈希 |
| 用户名/昵称 | **不受影响**（用户选择的，公开可见） |

**平台支持：** 删除适用于 WhatsApp、Signal 和 Telegram。Discord 和 Slack 被排除，因为它们的提及系统（`<@user_id>`）需要 LLM 上下文中的真实 ID。

哈希是确定性的 —— 同一用户始终映射到同一哈希，因此模型仍然可以在群聊中区分用户。路由和传递在内部使用原始值。

### OpenAI Codex 请求身份 {#openai-codex-request-identity}

OpenAI 要求第三方 Codex harness 表明自己的身份。经 ChatGPT 认证、发往官方 Codex 端点的请求会自动发送 `originator: hermes-agent` 和 `User-Agent: HermesAgent/<version>`。现有的 ChatGPT 账户头会保留。不会发送额外的提示内容或遥测请求。直接的 OpenAI API 请求和自定义代理端点保持不变。

## 语音转文字（STT）

```yaml
stt:
  provider: "local"            # "local" | "groq" | "openai" | "mistral"
  local:
    model: "base"              # tiny、base、small、medium、large-v3
  openai:
    model: "whisper-1"         # whisper-1 | gpt-4o-mini-transcribe | gpt-4o-transcribe | gpt-transcribe
  # model: "whisper-1"         # 旧版回退键仍受支持
```

Provider 行为：

- `local` 使用在您机器上运行的 `faster-whisper`。使用 `python -c "import pm; pm.sync_venv(['stt-whisper'], explicit=True)"` 单独安装。静音幻觉防护默认开启:Silero VAD 过滤器让静音/噪声不会进入 Whisper,跨窗口条件预测被禁用,并且模型自己标记为"很可能不是语音"且低置信度的片段会被丢弃。设置 `stt.local.vad: false` 可用原始行为转录非语音音频(音乐、环境声)。
- `groq` 使用 Groq 的 Whisper 兼容端点，读取 `GROQ_API_KEY`。
- `openai` 使用 OpenAI 语音 API，读取 `VOICE_TOOLS_OPENAI_KEY`。

如果请求的 provider 不可用，Hermes 按此顺序自动回退：`local` → `groq` → `openai`。

Groq 和 OpenAI 模型覆盖由环境变量驱动：

```bash
STT_GROQ_MODEL=whisper-large-v3-turbo
STT_OPENAI_MODEL=whisper-1
GROQ_BASE_URL=https://api.groq.com/openai/v1
STT_OPENAI_BASE_URL=https://api.openai.com/v1
```

### 转写提示（词汇提示） {#transcription-prompt-vocabulary-hints}

`stt.prompt` 是传递给支持提示的 STT 后端的可选静态提示。用于专有名词、产品名，以及 Whisper 家族模型本来会听错的术语：

```yaml
stt:
  provider: "local"
  prompt: "Hermes, Teknium, Nous Research, kanban, Ollama"
```

**组合方式。** 配置值是基底。注册了 `pre_transcription` 钩子的插件在其上修改，逐字段后写者胜。多个插件的提示以确定性的方式组合：插件发现按插件 id 排序加载插件，每个插件的回调按自己的注册顺序运行，因此同一组插件总是产生相同的最终提示。钩子对 `prompt` 返回空字符串会为该请求清除配置中的提示。钩子也可以覆盖 `language` 和 `model`；`file_path` 只读，任何试图修改它的尝试都会被记录并丢弃。未注册钩子且未设置 `stt.prompt` 时，发出的请求与之前版本完全一致。

**Provider 支持。**

| Provider | 提示参数 | 行为 |
|----------|-----------------|----------|
| `local`（faster-whisper） | `initial_prompt` | 原样转发给本地模型 |
| `openai` | `prompt` | 原样转发到转写请求中 |
| `groq` | `prompt` | 原样转发到转写请求中 |
| `mistral` | `prompt` | 原样转发到转写请求中 |
| `deepinfra` | `prompt` | OpenAI 兼容路径，原样转发 |
| `xai` | 不支持 | 以 DEBUG 级别记录，请求不带提示继续 |
| `elevenlabs` | 不支持 | 以 DEBUG 级别记录，请求不带提示继续 |
| `local_command` | 不支持 | 以 DEBUG 级别记录，请求不带提示继续 |
| `stt.providers.<name>` 且 `type: command` | 不支持 | 以 DEBUG 级别记录，请求不带提示继续 |
| 插件注册的 provider | `transcribe(**extra)` kwargs 中的 `prompt` | 仅在设置了提示时发送，因此早于该键的 provider 看到的是不变的调用 |

**长度。** Whisper 家族模型只对最后的约 224 个提示 token 条件化。对于 whisper 家族后端（`local`、`openai`、`groq`、`deepinfra`），Hermes 在客户端强制执行该上限：过长的最终提示会被截断到尾部并记录警告 —— 请求绝不会因提示长度而报错。其他后端（`mistral`、插件 provider）收到未改动的提示，自行负责校验。无论哪种情况，都请保持提示简短具体。

:::warning 提示会随你的音频一起上传
最终提示会与音频文件一起发送给配置的 STT provider。请把机密和会话派生的上下文排除在 `stt.prompt` 以及 `pre_transcription` 钩子返回的任何内容之外，尤其是当 provider 是托管 API 而非本地 `faster-whisper` 时。
:::

## 语音模式（CLI）

```yaml
voice:
  record_key: "ctrl+b"         # CLI 内的按键通话键
  max_recording_seconds: 120    # 长录音的硬停止
  auto_tts: false               # /voice on 时自动启用口语回复
  beep_enabled: true            # 在 CLI 语音模式中播放录音开始/停止提示音
  silence_threshold: 200        # 语音检测的 RMS 阈值
  silence_duration: 3.0         # 自动停止前的静默秒数
```

在 CLI 中使用 `/voice on` 启用麦克风模式，使用 `record_key` 开始/停止录音，使用 `/voice tts` 切换口语回复。端到端设置和平台特定行为请参阅[语音模式](./features/voice-mode.md)。

## 流式传输

将 token 实时流式传输到终端或消息平台，而不是等待完整响应。

### CLI 流式传输

```yaml
display:
  streaming: true         # 实时将 token 流式传输到终端
  show_reasoning: true    # 同时流式传输推理/思考 token（可选）
```

启用后，响应在流式传输框内逐 token 出现。工具调用仍然静默捕获。如果 provider 不支持流式传输，它会自动回退到正常显示。

### Gateway 流式传输（Telegram、Discord、Slack）

```yaml
streaming:
  enabled: true           # 启用渐进式消息编辑
  transport: edit         # "edit"（渐进式消息编辑）或 "off"
  edit_interval: 0.3      # 消息编辑之间的秒数
  buffer_threshold: 40    # 强制编辑刷新前的字符数
  cursor: " ▉"            # 流式传输期间显示的光标
  fresh_final_after_seconds: 0    # 预览超过此时间时选择发送新的最终消息（Telegram）
```

启用后，bot 在第一个 token 时发送消息，然后随着更多 token 到来渐进式编辑它。不支持消息编辑的平台（Signal、Email、Home Assistant）在第一次尝试时自动检测 —— 该会话的流式传输被优雅地禁用，不会产生大量消息。

对于不带渐进式 token 编辑的独立自然轮次中 assistant 更新，请设置 `display.interim_assistant_messages: true`。

**溢出处理：** 如果流式传输的文本超过平台的消息长度限制（约 4096 字符），当前消息被最终化，新消息自动开始。

**新的最终消息（Telegram）：** Telegram 的 `editMessageText` 保留原始消息时间戳，因此长时间运行的流式回复即使在完成后也会保留第一个 token 的时间戳。设置 `fresh_final_after_seconds > 0` 可选择将旧预览作为全新的最终消息传递，并尽力删除旧预览。默认值为 `0`，始终就地最终化流式回复，避免某些客户端短暂显示重复消息再删除其中一条。

:::note
主开关 `streaming.enabled` 默认为 `false`——在你启用之前不会有任何流式传输。启用后，是否流式传输按**平台**决定：Telegram 默认带有 `display.platforms.telegram.streaming: true`（流式传输），Discord 为 `display.platforms.discord.streaming: false`（不流式传输）。因此启用流式传输后，Telegram 开箱即用地流式传输，Discord 在你修改其开关之前仍使用整条消息回复。你可以在仪表盘的 **Channels** 开关中或直接在 `~/.hermes/config.yaml` 中调整这些按平台的开关。
:::

## 群聊会话隔离

控制共享聊天是每个房间保持一个对话还是每个参与者一个对话：

```yaml
group_sessions_per_user: true  # true = 群组/频道中每用户隔离，false = 每个聊天一个共享会话
```

- `true` 是默认和推荐设置。在 Discord 频道、Telegram 群组、Slack 频道和类似共享上下文中，当平台提供用户 ID 时，每个发送者获得自己的会话。
- `false` 恢复到旧的共享房间行为。如果您明确希望 Hermes 将频道视为一个协作对话，这可能有用，但这也意味着用户共享上下文、token 成本和中断状态。
- 私信不受影响。Hermes 仍然像往常一样通过聊天/DM ID 键入 DM。
- 线程与其父频道保持隔离；使用 `true` 时，每个参与者在线程内也获得自己的会话。

有关行为详情和示例，请参阅[会话](./sessions.md)和 [Discord 指南](./messaging/discord.md)。

## 原生托管房间讨论上限 {#native-hosted-room-discussion-limits}

原生服务器托管房间在每个讨论中让所有成员共享这些讨论上限。它们适用于该服务拥有的所有原生托管房间，不适用于由 Desktop 渲染器驱动或镜像的房间。

```yaml
hosted_rooms:
  discussion:
    max_rounds: 3      # 每个讨论的默认最大排程轮数
    max_messages: 10   # 所有成员合计的默认可见 bot 回复总数
```

两个键都是可选的；省略时保留向后兼容的默认值 3 轮和 10 条回复。值必须是正整数：不接受 `0`、负值、`null`、布尔值、字符串（包括带引号的数字或 `"unlimited"`）以及浮点数。无效值会被忽略，并在 gateway 日志中给出警告，然后应用默认值，因此一个笔误永远不会让群聊离线。例如，把 `max_rounds: 6` 和 `max_messages: 30` 设成更大但仍受控的讨论预算（这样 5 人房间就能越过 10 条回复的停止点）。

请在持有房间存储的 Hermes home（紧邻 `shared-state.db` 的根 home）的 `config.yaml` 中设置这些键，而不是在单个房间成员的 profile 配置里。它们对 gateway 和 Desktop/TUI 后端所服务的房间同样生效。每个服务进程在启动时读取一次：修改后请重启共享该房间存储的每个进程。

这些是上限，不是目标，也不保证任务完成。讨论可能提前结束，Stop 与审批门仍然有效。

## 未授权 DM 行为

控制当未知用户发送私信时 Hermes 的行为：

```yaml
unauthorized_dm_behavior: pair

whatsapp:
  unauthorized_dm_behavior: ignore
```

- `pair` 是默认值。Hermes 拒绝访问，但在 DM 中回复一次性配对码。
- `ignore` 静默丢弃未授权的 DM。
- 平台部分覆盖全局默认值，因此您可以在广泛范围内保持配对启用，同时使一个平台更安静。

## 快速命令

定义自定义命令，这些命令要么在不调用 LLM 的情况下运行 shell 命令，要么将一个斜杠命令别名为另一个。Exec 快速命令是零 token 的，对于从消息平台（Telegram、Discord 等）进行快速服务器检查或实用脚本很有用。

```yaml
quick_commands:
  status:
    type: exec
    command: systemctl status hermes-agent
  disk:
    type: exec
    command: df -h /
  update:
    type: exec
    command: hermes update
  gpu:
    type: exec
    command: nvidia-smi --query-gpu=name,utilization.gpu,memory.used,memory.total --format=csv,noheader
  restart:
    type: alias
    target: /gateway restart
```

用法：在 CLI 或任何消息平台中输入 `/status`、`/disk`、`/update`、`/gpu` 或 `/restart`。`exec` 命令在宿主本地运行并直接返回输出 —— 无 LLM 调用，不消耗 token。`alias` 命令重写为配置的斜杠命令目标。

- **30 秒超时** —— 长时间运行的命令被终止并显示错误消息
- **优先级** —— 快速命令在技能命令之前检查，因此您可以覆盖技能名称
- **自动补全** —— 快速命令在调度时解析，不显示在内置斜杠命令自动补全表中
- **类型** —— 支持的类型为 `exec` 和 `alias`；其他类型显示错误
- **到处可用** —— CLI、Telegram、Discord、Slack、WhatsApp、Signal、Email、Home Assistant

仅字符串的 prompt 快捷方式不是有效的快速命令。对于可重用的 prompt 工作流，请创建技能或别名到现有斜杠命令。

## 人类延迟

在消息平台中模拟类人响应节奏：

```yaml
human_delay:
  mode: "off"                  # off | natural | custom
  min_ms: 800                  # 最小延迟（自定义模式）
  max_ms: 2500                 # 最大延迟（自定义模式）
```

## 代码执行

配置 `execute_code` 工具：

```yaml
code_execution:
  mode: project                # project（默认）| strict
  timeout: 300                 # 最大执行时间（秒）
  max_tool_calls: 50           # 代码执行中的最大工具调用次数
```

**`mode`** 控制脚本的工作目录和 Python 解释器：

- **`project`**（默认）—— 脚本在会话的工作目录中以活跃 virtualenv/conda 环境的 python 运行。项目依赖（`pandas`、`torch`、项目包）和相对路径（`.env`、`./data.csv`）自然解析，与 `terminal()` 看到的一致。
- **`strict`** —— 脚本在临时暂存目录中以 `sys.executable`（Hermes 自己的 python）运行。最大可重现性，但项目依赖和相对路径不会解析。

环境清理（删除 `*_API_KEY`、`*_TOKEN`、`*_SECRET`、`*_PASSWORD`、`*_CREDENTIAL`、`*_PASSWD`、`*_AUTH`）和工具白名单在两种模式下完全相同 —— 切换模式不会改变安全态势。

## Web 搜索后端

`web_search` 和 `web_extract` 工具支持五种后端 provider。在 `config.yaml` 中或通过 `hermes tools` 配置后端：

```yaml
web:
  backend: firecrawl    # firecrawl | searxng | parallel | tavily | exa

  # 或使用每功能键混合 provider（例如免费搜索 + 付费提取）：
  search_backend: "searxng"
  extract_backend: "firecrawl"
```

| 后端 | 环境变量 | 搜索 | 提取 |
|---------|---------|--------|---------|
| **Firecrawl**（默认） | `FIRECRAWL_API_KEY` | ✔ | ✔ |
| **SearXNG** | `SEARXNG_URL` | ✔ | — |
| **Parallel** | `PARALLEL_API_KEY` | ✔ | ✔ |
| **Tavily** | `TAVILY_API_KEY` | ✔ | ✔ |
| **Exa** | `EXA_API_KEY` | ✔ | ✔ |

**后端选择：** 如果未设置 `web.backend`，后端从可用的 API 密钥自动检测。如果仅设置了 `SEARXNG_URL`，使用 SearXNG。如果仅设置了 `EXA_API_KEY`，使用 Exa。如果仅设置了 `TAVILY_API_KEY`，使用 Tavily。如果仅设置了 `PARALLEL_API_KEY`，使用 Parallel。否则 Firecrawl 是默认值。

**SearXNG** 是一个免费、自托管、尊重隐私的元搜索引擎，查询 70+ 个搜索引擎。无需 API 密钥 —— 只需将 `SEARXNG_URL` 设置为您的实例（例如 `http://localhost:8080`）。SearXNG 仅限搜索；`web_extract` 需要单独的提取 provider（设置 `web.extract_backend`）。Docker 设置说明请参阅 [Web 搜索设置指南](./features/web-search.md)。

**自托管 Firecrawl：** 设置 `FIRECRAWL_API_URL` 指向您自己的实例。设置自定义 URL 后，API 密钥变为可选（在服务器上设置 `USE_DB_AUTHENTICATION=***` 以禁用认证）。

**Parallel 搜索模式：** 设置 `PARALLEL_SEARCH_MODE` 控制搜索行为 —— `fast`、`one-shot` 或 `agentic`（默认：`agentic`）。

**Exa：** 在 `~/.hermes/.env` 中设置 `EXA_API_KEY`。支持 `category` 过滤（`company`、`research paper`、`news`、`people`、`personal site`、`pdf`）和域名/日期过滤器。

## 浏览器

配置浏览器自动化行为。所有浏览器键都位于 `browser:` 下；各浏览器来源（本地 Chromium、云端提供商、Lightpanda、`/browser connect`、真实 profile）如何配合，请参阅[浏览器功能页面](./features/browser.md)。

```yaml
browser:
  inactivity_timeout: 120        # 关闭空闲浏览器前的秒数
  command_timeout: 30            # Chromium 启动超时下限（秒）
  headed: false                  # true = 显示 Chromium 窗口（本地）；轮次之间保持浏览器打开
  engine: auto                   # auto（Chromium）| lightpanda（"chrome" 视为 auto）
  # Hermes 启动的本地 Chromium
  executable_path: ""            # 覆盖 Chromium 程序；"" = 打包的 Chromium
  user_data_dir: ""              # 所有会话共享的持久 profile；"" = 每个会话使用全新 profile
  chromium_args: []              # 额外的 Chromium 命令行参数，例如 ["--lang=de-DE"]
  # 云端浏览器选择（由 `hermes tools` → Browser Automation 写入）
  cloud_provider: local          # local | browserbase | browser-use | firecrawl | nous | <提供商插件>；未设置 = 根据凭据自动检测
  auto_local_for_private_urls: true  # 使用云端提供商时，在本地 Chromium 中打开 LAN/localhost URL
  allow_private_urls: false      # 允许私有/内部地址（localhost、192.168.x.x 等）
  # 可选 CDP 覆盖 —— 设置后，浏览器工具附加到您自己的
  # Chromium 系浏览器（与 /browser connect 相同），而不是自行启动。
  cdp_url: ""
  use_real_profile: false        # 在您真实浏览器 profile 的托管副本上浏览（需主动开启）
  # 对话框监督器 —— 如何处理原生 JS 对话框（alert / confirm / prompt / beforeunload）
  dialog_policy: must_respond    # must_respond | auto_dismiss | auto_accept
  dialog_timeout_s: 300          # must_respond 下的安全自动关闭（秒）
```

**可执行文件解析顺序。** Hermes 启动它找到的第一个 Chromium：`browser.executable_path` → Hermes 包管理器安装的 Chromium（`hermes pm install chromium`）→ `/etc/hermes/chromium-executable-path`（官方 Docker 镜像）→ `PATH` 上的系统 `chromium` / `chromium-browser`（Termux 以及没有打包构建的平台）。都找不到时，浏览器工具会给出提示，而不是静默失败。

**关闭浏览功能。** 浏览器没有配置开关：通过 `hermes tools`（或 `hermes tools disable browser [--platform NAME]`）禁用 `browser` 工具集。

**对话框策略：**

- `must_respond`（默认）—— 捕获对话框，等待 agent 调用 `browser_dialog(action=...)`。在 `dialog_timeout_s` 秒内无响应后，对话框被自动关闭以防止页面的 JS 线程永久停滞。
- `auto_dismiss` —— 捕获，立即关闭。
- `auto_accept` —— 捕获，立即接受。适用于有激进 `beforeunload` 提示的页面。

完整对话框工作流请参阅[浏览器功能页面](./features/browser.md#browser_dialog)。

**已移除的键（自动迁移）。** `browser.backend` 已移除——`browser_exec` 是唯一的浏览器路径。旧的 `backend: "off"` 会在更新时被删除，并一次性提示改为禁用 `browser` 工具集；其他值会被静默删除。`snapshot_threshold`、`record_sessions`、`allow_unsafe_evaluate` 和 `restrict_evaluate` 属于已移除的内置浏览器工具，不再生效。`browser.camofox.*` 键由 Camofox 插件（`hermes plugins install camofox`）读取，而非核心。旧的 `AGENT_BROWSER_*` 环境变量不再被读取；更新时，若对应键未设置，会把 profile 的 `.env` 中的值复制过去（`AGENT_BROWSER_EXECUTABLE_PATH` → `executable_path`、`AGENT_BROWSER_PROFILE` → `user_data_dir`、`AGENT_BROWSER_ARGS` → `chromium_args`、`AGENT_BROWSER_HEADED` → `headed`、`AGENT_BROWSER_ENGINE` → `engine`）。

## 时区

使用 IANA 时区字符串覆盖服务器本地时区。影响日志中的时间戳、cron 调度和系统提示词时间注入。

```yaml
timezone: "America/New_York"   # IANA 时区（默认："" = 服务器本地时间）
```

支持的值：任何 IANA 时区标识符（例如 `America/New_York`、`Europe/London`、`Asia/Kolkata`、`UTC`）。留空或省略以使用服务器本地时间。

代理时钟、cron 调度和时间相关工具在所有操作系统上都遵循此时区。通过 `execute_code` 运行的代码在 Linux 和 macOS 上也会以 `TZ` 的形式继承它；在 Windows 上这些子进程保留操作系统配置的时区（Windows C 运行时只解析 POSIX 形式的 `TZ` 字符串，IANA 名称会产生错误的 UTC 偏移量），因此若子脚本必须以此时区显示本地时间，请直接设置 Windows 系统时区。

## Discord

为消息 gateway 配置 Discord 特定行为：

```yaml
discord:
  require_mention: true          # 在服务器频道中需要 @提及才能响应
  free_response_channels: ""     # 逗号分隔的频道 ID，bot 在这些频道无需 @提及即可响应
  auto_thread: true              # 在频道中 @提及时自动创建线程
```

- `require_mention` —— 为 `true`（默认）时，bot 仅在服务器频道中被 `@BotName` 提及时响应。DM 始终无需提及即可工作。
- `free_response_channels` —— 逗号分隔的频道 ID 列表，bot 在这些频道对每条消息响应，无需提及。
- `auto_thread` —— 为 `true`（默认）时，频道中的提及会自动为对话创建线程，保持频道整洁（类似 Slack 线程）。

## 安全

机密脱敏和网站黑名单：

```yaml
security:
  redact_secrets: false          # 在工具输出和日志中脱敏 API 密钥模式（默认关闭）
  website_blocklist:             # 参见下方网站黑名单部分
    enabled: false
    domains: []
    shared_files: []
```

- `redact_secrets` —— 为 `true` 时，自动检测并脱敏工具输出中看起来像 API 密钥、token 和密码的模式，然后再进入对话上下文和日志。**默认关闭** —— 如果您经常在工具输出中处理真实凭据并希望有安全网，请启用。显式设置为 `true` 以开启。

使用 tirith 扫描命令现在是可选插件，参见[内容级命令检查](security.md#内容级命令检查)。

## 网站黑名单

阻止 agent 的 web 和浏览器工具访问特定域名：

```yaml
security:
  website_blocklist:
    enabled: false               # 启用 URL 阻止（默认：false）
    domains:                     # 被阻止的域名模式列表
      - "*.internal.company.com"
      - "admin.example.com"
      - "*.local"
    shared_files:                # 从外部文件加载额外规则
      - "/etc/hermes/blocked-sites.txt"
```

启用后，任何匹配被阻止域名模式的 URL 在 web 或浏览器工具执行之前都会被拒绝。这适用于 `web_search`、`web_extract`、浏览器导航以及任何访问 URL 的工具。

域名规则支持：
- 精确域名：`admin.example.com`
- 通配符子域名：`*.internal.company.com`（阻止所有子域名）
- TLD 通配符：`*.local`

共享文件每行包含一条域名规则（空行和 `#` 注释被忽略）。缺失或不可读的文件记录警告，但不禁用其他 web 工具。

策略缓存 30 秒，因此配置更改无需重启即可快速生效。

## 智能审批

控制 Hermes 如何处理潜在危险命令：

```yaml
approvals:
  mode: smart   # smart | manual | off
```

| 模式 | 行为 |
|------|----------|
| `smart`（默认） | 使用辅助 LLM 评估被标记的命令是否真正危险。低风险命令仅对当前命令自动批准，真正危险的命令自动拒绝，不确定的情况升级给用户。 |
| `manual` | 在执行任何被标记的命令之前提示用户。在 CLI 中显示交互式审批对话框。在消息中排队待处理的审批请求。 |
| `off` | 跳过所有审批检查。等同于 `HERMES_YOLO_MODE=true`。**谨慎使用。** |

智能模式对于减少审批疲劳特别有用 —— 它让 agent 在安全操作上更自主地工作，同时仍然捕获真正破坏性的命令。

:::warning
设置 `approvals.mode: off` 会禁用终端命令的所有安全检查。仅在受信任的沙箱环境中使用。
:::

### 拒绝熔断器 {#denial-circuit-breaker}

`approvals.denial_breaker_threshold`（默认 `3`）防止 agent 反复尝试智能审批审查器一直拒绝的命令变体 —— 每次重试都要再烧一次 guardian LLM 调用。在一个会话中连续被拒这么多次之后，拒绝消息会升级为硬停止指令，告诉 agent 停下、报告被阻止的操作，并请你手动运行它或使用 `/approve`。任何批准都会重置计数；设为 `0` 可禁用：

```yaml
approvals:
  denial_breaker_threshold: 3   # 0 禁用熔断器
```

### 拒绝规则 {#deny-rules}

`approvals.deny` 是一组 glob 模式列表，会无条件阻止匹配的终端命令 —— 即使在 `--yolo`、`/yolo` 或 `mode: off` 下也是如此。它是内置硬线黑名单的用户可编辑对应物：

```yaml
approvals:
  deny:
    - "git push --force*"
    - "*curl*|*sh*"
```

模式是不区分大小写的 fnmatch glob，在 YAML 中必须加引号（裸的前导 `*` 会导致解析错误）。详情见[安全 —— 用户自定义拒绝规则](./security.md)。

### 自定义智能审批策略 {#custom-smart-approval-policy}

`approvals.smart_policy` 让你把自己的规则追加到智能审批审查器的指令中。设置后，该文本会加入 guardian LLM 的系统提示（受信通道 —— 绝不与不受信任的命令文本并列），因此你可以针对自己的环境收紧或放宽它的判断，而无需改代码：

```yaml
approvals:
  smart_policy: |
    Always ESCALATE commands that modify anything under /etc.
    APPROVE docker compose restarts in ~/deploys — they are routine here.
```

## 检查点

破坏性文件操作之前的自动文件系统快照。详情请参阅[检查点与回滚](./checkpoints-and-rollback.md)。

```yaml
checkpoints:
  enabled: false                 # 启用自动检查点（也可：hermes chat --checkpoints）。默认：false（选择加入）。
  max_snapshots: 20              # 每个目录保留的最大检查点数（默认：20）
```


## 委托

为委托工具配置子 agent 行为：

```yaml
delegation:
  # model: "google/gemini-3-flash-preview"  # 覆盖模型（空 = 继承父级）
  # provider: "openrouter"                  # 覆盖 provider（空 = 继承父级）
  # base_url: "http://localhost:1234/v1"    # 直接 OpenAI 兼容端点（优先于 provider）
  # api_key: "local-key"                    # base_url 的 API 密钥（回退到 OPENAI_API_KEY）
  # api_mode: ""                            # base_url 的线路协议："chat_completions"、"codex_responses" 或 "anthropic_messages"。空 = 从 URL 自动检测（例如 /anthropic 后缀 → anthropic_messages）。对启发式无法检测的非标准端点显式设置。
  max_concurrent_children: 3                # 每批并行子 agent 数（下限 1，无上限）。也可通过 DELEGATION_MAX_CONCURRENT_CHILDREN 环境变量设置。
  max_spawn_depth: 1                        # 委托树深度上限（1-3，截断）。1 = 扁平（默认）：父级生成无法委托的叶子。2 = 编排器子级可以生成叶子孙级。3 = 三级。
  orchestrator_enabled: true                # 全局终止开关。为 false 时，role="orchestrator" 被忽略，每个子级无论 max_spawn_depth 如何都被强制为叶子。
```

**子 agent provider:model 覆盖：** 默认情况下，子 agent 继承父 agent 的 provider 和模型。设置 `delegation.provider` 和 `delegation.model` 将子 agent 路由到不同的 provider:model 对 —— 例如，在您的主 agent 运行昂贵推理模型时，为范围较窄的子任务使用便宜/快速的模型。

**直接端点覆盖：** 如果您想要明显的自定义端点路径，请设置 `delegation.base_url`、`delegation.api_key` 和 `delegation.model`。这将子 agent 直接发送到该 OpenAI 兼容端点，并优先于 `delegation.provider`。如果省略 `delegation.api_key`，Hermes 仅回退到 `OPENAI_API_KEY`。

**线路协议（`api_mode`）：** Hermes 从 `delegation.base_url` 自动检测线路协议（例如以 `/anthropic` 结尾的路径 → `anthropic_messages`；Codex/原生 Anthropic/Kimi-coding 主机名保留其现有检测）。对于启发式无法分类的端点 —— 例如 Azure AI Foundry、MiniMax、Zhipu GLM 或前置 Anthropic 形状后端的 LiteLLM 代理 —— 请将 `delegation.api_mode` 显式设置为 `chat_completions`、`codex_responses` 或 `anthropic_messages` 之一。留空（默认）以保持自动检测。

委托 provider 使用与 CLI/gateway 启动相同的凭据解析。所有配置的 provider 均受支持：`openrouter`、`nous`、`zai`、`kimi-coding`、`minimax`、`minimax-cn`。设置 provider 时，系统自动解析正确的基础 URL、API 密钥和 API 模式 —— 无需手动凭据连接。

**优先级：** 配置中的 `delegation.base_url` → 配置中的 `delegation.provider` → 父 provider（继承）。配置中的 `delegation.model` → 父模型（继承）。仅设置 `model` 而不设置 `provider` 仅更改模型名称，同时保留父级凭据（适用于在同一 provider（如 OpenRouter）内切换模型）。

**宽度和深度：** `max_concurrent_children` 限制每批并行运行的子 agent 数量（默认 `3`，下限 1，无上限）。也可通过 `DELEGATION_MAX_CONCURRENT_CHILDREN` 环境变量设置。当模型提交的 `tasks` 数组超过上限时，`delegate_task` 返回工具错误解释限制，而不是静默截断。`max_spawn_depth` 控制委托树深度（截断到 1-3）。在默认 `1` 时，委托是扁平的：子级无法生成孙级，传递 `role="orchestrator"` 静默降级为 `leaf`。提升到 `2` 使编排器子级可以生成叶子孙级；`3` 用于三级树。Agent 通过 `role="orchestrator"` 按调用选择编排；`orchestrator_enabled: false` 强制每个子级回到叶子，无论如何。成本呈乘法增长 —— 在 `max_spawn_depth: 3` 和 `max_concurrent_children: 3` 时，树可以达到 3×3×3 = 27 个并发叶子 agent。使用模式请参阅[子 Agent 委托 → 深度限制和嵌套编排](features/delegation.md#depth-limit-and-nested-orchestration)。

## 澄清

配置澄清提示行为：

```yaml
clarify:
  timeout: 120                 # 等待用户澄清响应的秒数
```

## 上下文文件（SOUL.md、AGENTS.md）

Hermes 使用两种不同的上下文范围：

| 文件 | 用途 | 范围 |
|------|---------|-------|
| `SOUL.md` | **主要 agent 身份** —— 定义 agent 是谁（系统提示词第 #1 槽位） | `~/.hermes/SOUL.md` 或 `$HERMES_HOME/SOUL.md` |
| `.hermes.md` / `HERMES.md` | 项目特定指令（最高优先级） | 向上走到 git 根目录 |
| `AGENTS.md` | 项目特定指令、编码规范 | 递归目录遍历 |
| `CLAUDE.md` | Claude Code 上下文文件（也会检测） | 仅工作目录 |
| `.cursorrules` | Cursor IDE 规则（也会检测） | 仅工作目录 |
| `.cursor/rules/*.mdc` | Cursor 规则文件（也会检测） | 仅工作目录 |

- **SOUL.md** 是 agent 的主要身份。它占据系统提示词的第 #1 槽位，完全替换内置的默认身份。编辑它以完全自定义 agent 是谁。
- 如果 SOUL.md 缺失、为空或无法加载，Hermes 回退到内置默认身份。
- **项目上下文文件使用优先级系统** —— 仅加载一种类型（第一个匹配优先）：`.hermes.md` → `AGENTS.md` → `CLAUDE.md` → `.cursorrules`。SOUL.md 始终独立加载。
- **AGENTS.md** 是分层的：如果子目录也有 AGENTS.md，所有都会合并。
- 如果 `SOUL.md` 不存在，Hermes 会自动生成默认的 `SOUL.md`。
- 所有加载的上下文文件上限为 `context_file_max_chars` 个字符（默认 20,000），并进行智能截断。

另请参阅：
- [个性与 SOUL.md](./features/personality.md)
- [上下文文件](./features/context-files.md)

## 工作目录

| 上下文 | 默认值 |
|---------|---------|
| **CLI（`hermes`）** | 运行命令的当前目录 |
| **消息 gateway** | 主目录 `~`（用 `MESSAGING_CWD` 覆盖） |
| **Docker / Singularity / Modal / SSH** | 容器或远程机器内用户的主目录 |

覆盖工作目录：
```bash
# 在 ~/.hermes/.env 或 ~/.hermes/config.yaml 中：
MESSAGING_CWD=/home/myuser/projects    # Gateway 会话
TERMINAL_CWD=/workspace                # 所有终端会话
```

## 网络 {#network}

出站 HTTP 的连通性变通方案：

```yaml
network:
  force_ipv4: false   # 强制出站连接使用 IPv4（默认：false）
```

`force_ipv4` —— 在 IPv6 损坏或不可达的服务器上，Python 会先解析 AAAA 记录，可能在回落到 IPv4 之前挂满整个 TCP 超时。Hermes 已经为它发出的每个出站连接同时竞速 IPv6 与 IPv4（Happy Eyeballs，RFC 8305：IPv4 尝试在 IPv6 之后 250 ms 启动，谁先连上算谁赢），因此一条被通告但实际黑洞的 IPv6 路由每次连接只花约四分之一秒，而不是整个超时。这覆盖 gateway 的 WebSocket 拨号（relay 连接器、平台适配器）以及 HTTP。只有当你希望完全跳过 IPv6 直接走 IPv4 时才把它设为 `true`。`hermes doctor` 会执行 `IPv6 route` 检查，检测死亡的 IPv6 路径并指向该设置。

## 引导 {#onboarding}

首次接触的引导提示与首条消息的邀约：

```yaml
onboarding:
  profile_build: "ask"   # "ask"（默认）| "off"
  seen: {}               # 内部锁存 —— 保持为空
  run: null              # 桌面端首次运行引导：null（全新安装）| true | false
```

- `profile_build` —— 控制有史以来第一条消息上的邀约，在 TUI、桌面应用和消息平台私信中生效（群聊中永不出现）。`"ask"`（默认）会主动提出构建用户画像；该邀约是**选择加入且需征得同意的** —— agent 在任何查找之前先询问，绝不静默读取已连接的账户，并把确认的事实保存到用户记忆。`"off"` 只停止该邀约；agent 仍会给出简单的介绍。该邀约每个 profile 最多出现一次。
- `seen` —— 内部状态。Hermes 在此锁存每个已展示的提示，使其不再触发；首条消息的邀约在展示后也记录在此。不要手工编辑它 —— 如果想重新看到所有提示，请清空整个 `onboarding` 段。
- `run` —— 桌面应用下次启动时是否打开首次运行引导聊天。未设置表示仅在全新安装时打开（尚无会话和 profile）。引导开始时 Hermes 写入 `true`，完成或跳过时写入 `false`。它始终位于默认 profile 的 `config.yaml` 中，无论应用运行在哪个 profile。设为 `false` 可永不显示；托管安装可以在托管配置中钉住该值。

## Webapp

gateway 可以在自己的监听器上提供[浏览器中的 Desktop 工作区](./desktop.md#use-the-desktop-workspace-from-a-browser)。在你启用它之前，它是关闭的。

```yaml
webapp:
  enabled: false     # true：gateway 提供 Webapp
  host: 127.0.0.1    # 127.0.0.1、::1 或 localhost；或 0.0.0.0 / :: 供其他机器访问（需要仪表盘登录）
  port: 9120         # 固定端口，这样 gateway 重启后浏览器的地址保持不变
  public_url: ""     # 反向代理或隧道提供 Webapp 所用的 https:// 地址（需要登录）
```

- `enabled` — gateway 在启动时读取它；修改后需重启 gateway（`hermes gateway restart`）。
- `host` — `127.0.0.1`、`::1` 或 `localhost` 让 Webapp 只留在本机。`0.0.0.0` 或 `::` 使其可从其他机器访问，并要求配置一个[仪表盘登录提供商](./features/web-dashboard.md#authentication-gated-mode)；没有提供商时，gateway 只提供原生客户端服务，`hermes webapp --status` 会说明原因。添加提供商后请重启 gateway。其他任何值（包括 `127.0.0.2` 这类另一个回环地址）都会让 Webapp 保持关闭。
- `port` — 独立 profile 的 gateway 需要使用与默认 gateway 不同的端口。**Settings → Web Access → Start** 会为未设置端口的独立 profile 选择一个端口（9120 以上、其他 profile 未占用的第一个空闲端口）并保存到此处；它绝不会替换你设置的端口。端口被占用时，gateway 同样只提供原生客户端服务，`hermes webapp --status` 会报告原因。这也会替换 Webapp 的浏览器登录，因为该端口可能已被其他程序收到，所以端口释放后已打开的标签页需要重新执行 `hermes webapp`。
- `public_url` — 当你运行的反向代理或隧道以自身地址（转发到 `host:port`）提供 Webapp 时设置它。它即使在回环 `host` 上也会开启仪表盘登录，是 Webapp 接受请求的唯一外部主机名，登录回调也基于它构建。没有登录提供商时，与 `0.0.0.0` 的情况一样，gateway 只提供原生客户端服务。它必须是 `http://` 或 `https://` 地址，且不带凭据、查询串或片段；其他任何值都会让 Webapp 保持关闭。`dashboard.public_url` 用于 `hermes dashboard`，永远不会作用于 Webapp。

无论 `host` 是什么，只要 `host` 为 `0.0.0.0` 或 `::`，或设置了 `public_url`，就会启用登录。此时 `hermes webapp` 会打开 Webapp 的地址（设置了 `public_url` 时就是登录返回的地址）供你用仪表盘账户登录：它的一次性链接只会打开无需登录的 Webapp。原生客户端（CLI、TUI 和 Desktop 应用）从不登录：gateway 为它们提供的监听器永远不会要求登录，无论配置如何。

Webapp 设置从为该 profile 提供服务的 gateway 的配置中读取：对于它服务的具名 profile，读取默认 profile 的配置；否则读取独立 profile 自己的配置。

## 仪表盘 {#dashboard}

[Web 仪表盘](./features/web-dashboard.md)的配置 —— 视觉主题、公共 URL 与认证提供商。认证提供商（OAuth、基础密码、drain）在 web-dashboard 页面有详细说明；这里是 `config.yaml` 中的形状。

```yaml
dashboard:
  theme: "default"            # "default" | "midnight" | "ember" | "mono" | "cyberpunk" | "rose"
  show_token_analytics: false # 重新启用（仅本地估算的）token/成本分析界面
  public_url: ""              # OAuth redirect_uri 的完整公共权威（环境变量：HERMES_DASHBOARD_PUBLIC_URL）
  trusted_proxies: []         # 允许提供 X-Forwarded-* 头的代理 IP/CIDR
  oauth:                      # Portal OAuth 门禁（配合 --host 且非 --insecure 时启用）
    client_id: ""             # agent:{instance_id} —— 由 Portal 提供
    portal_url: ""            # 留空 → 插件默认值（生产 Portal）
  basic_auth:                 # 自托管用户名/密码门禁（dashboard_auth/basic 插件）
    username: ""              # 留空 → 插件不生效
    password_hash: ""         # scrypt$...（首选 —— 静态存储无明文）
    password: ""              # 明文回退（加载时在内存中哈希）
    secret: ""                # token 签名密钥；留空 → 每进程随机
    session_ttl_seconds: 0    # 0 → 插件默认值（12 小时）
  drain_auth:                 # Drain 控制服务凭据门禁（dashboard_auth/drain 插件）
    scope: "drain"            # 已验证主体的能力标签
    min_secret_chars: 43      # 熵要求（url-safe-b64 字符；43 ≈ 256 位）
  ws_ping_interval: 20.0      # 非回环 WebSocket 保活 ping 间隔（秒）
  ws_ping_timeout: 20.0       # 非回环 WebSocket 保活 pong 超时（秒）
  ws_orphan_reap_grace_s: 20.0 # WS 分离会话被回收前的宽限期（秒）
  ssh_isolated_idle_grace_s: 900.0 # 经 SSH 的 Desktop 后端在无客户端且无运行中轮次这么久后退出
  ws_orphan_activity_stale_s: 600.0 # 分离的 RUNNING 轮次被中断前的活动空闲上限（秒）
  startup_orphan_sweep: true  # 启动时关闭因 gateway 进程死亡而遗留的会话行
```

- `theme` —— 仪表盘视觉主题。
- `show_token_analytics` —— 默认关闭。Analytics 页面与 token/成本数字是**本地下界估算**（不含辅助调用、重试、回退和缓存写入），因此可能远低于 provider 账单。只有在你明白它们不是账单时才设为 `true`。
- `public_url` —— 设置后，它就是构建 OAuth `redirect_uri` 所用的完整权威（scheme + host + 可选路径前缀）。对于部署在不稳定转发 `X-Forwarded-*` 头的反向代理后的环境，请设置它。留空则使用代理头重建。
- `trusted_proxies` —— 允许提供 `X-Forwarded-Proto` 和 `X-Forwarded-For` 的 IP 地址或有界 CIDR 网段。回环地址始终自动受信。当 TLS 反向代理从另一个容器或主机连接时配置它。优先使用代理的精确 IP；只有当地址是动态的时候才使用小型专用网段。通配符和 `/0` 网段会被拒绝。
- `oauth` / `basic_auth` / `drain_auth` —— 由随附的 dashboard-auth 插件读取的认证提供商配置。drain 密钥本身**不**在此设置；它通过 `HERMES_DASHBOARD_DRAIN_SECRET` 环境变量提供。完整认证设置见 [Web 仪表盘](./features/web-dashboard.md)。
- `ws_ping_interval` / `ws_ping_timeout` —— 非回环绑定的 WebSocket 保活调优（回环连接从不 ping）。在高延迟链路（Tailscale、远距离 SSH 隧道）上提高它们，因为 20 秒默认值可能制造虚假的 1006 断连。
- `ssh_isolated_idle_grace_s`（默认 `900`）—— 经 SSH 访问的、由 Desktop 拥有的 `hermes serve --isolated` 后端会被有意与 SSH 会话分离，因此睡眠中的笔记本不会把它拆掉；以前每次暗唤醒重连都会留下另一个持有 `state.db` 的后端。现在，一旦这段时间内没有客户端 WebSocket 连接且没有 agent 轮次在运行，后端就会自行退役（有轮次在跑时它保持存活；轮次状态不可读时也保持存活）。如果你依赖分离的后端在笔记本睡眠后完成长任务，请调高该值。这类后端还会发送慢速 WebSocket ping（60 秒，超时 10 分钟），以便发现半开隧道。独立于该宽限期，只要 Desktop 在远程主机上的所有权锁指向同一连接的更新派生，后端就会在轮次之间退役，因此重连风暴无法堆叠分离的后端；这覆盖 POSIX 和原生 Windows SSH 主机。
- `ws_orphan_reap_grace_s` —— WS 分离会话在孤儿回收器收集它之前等待多久。如果客户端重连缓慢，请与保活值一起调高。周期性的会话维护也会为已关闭的 socket 完成清理，并重新武装缺失的孤儿定时器，因此分离的聊天不能仅因为其初始清理或定时器丢失就一直占着所有权租约。重连会取消该定时器；活跃的委托工作和健康的运行中轮次仍受常规孤儿回收检查保护。（`HERMES_TUI_WS_ORPHAN_REAP_GRACE_S` 仍作为内部覆盖保留。）
- `ws_orphan_activity_stale_s`（默认 `600`）—— 分离的**运行中**轮次的活动时钟（与 `agent.turn_liveness` 看门狗采样的同一时钟：API 等待、流 token、工具心跳）必须空闲多久，孤儿回收器才会中断它。客户端缺席但仍在积极产出结果的轮次会继续分离运行直至完成 —— 合上笔记本、把移动应用切到后台，或桌面端更新都不再取消健康的长轮次；只有真正卡死的轮次会被中断。设为 `0` 可无视活动状态、一到宽限期就中断（旧行为）。
- `startup_orphan_sweep`（默认 `true`）—— 上面的 WS 孤儿回收定时器是进程内的，因此在它触发之前发生的 gateway 重启（更新、崩溃、systemd）会让会话行永远保持打开 —— 在 `/resume` 和仪表盘中表现为幽灵般的"活跃"工作。每次 gateway 启动时 —— 无论是 stdio TUI（`entry.main`）还是 desktop/dashboard WebSocket sidecar（`handle_ws`）—— 来源为 `tui` / `desktop` / `subagent` / `unknown`（token 记账守卫不得不自行实体化的行）且开始时间**和**最新消息都早于会话 TTL（`HERMES_TUI_SESSION_TTL_S`，默认 6 小时）的行，会以 `end_reason: startup_orphan_reap` 关闭。消息平台会话（Telegram、Discord 等）永不被触碰，活跃的内存中会话（已经恢复的客户端）被排除，被清理的会话仍可恢复。

