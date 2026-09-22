# herdr 终端多路复用器使用指南（写给初学者）

> 目标：不追求全功能，读完就能用 herdr 同时带着几个 AI coding agent 干活，合上终端也不丢工作。
> 全文讲法：大白话 + 图 + 例子。所有命令都可以开个测试目录照着敲。
> 本机（Ubuntu 22.04）截至 2026-09-22 **尚未安装**，先看第 2 节；装好后记得回第 2 节补记版本。
> 姊妹篇：[`lazy-git-usage.md`](./lazy-git-usage.md)（agent 产出的 diff 用 lazygit 审最舒服）、[`yazi-usage.md`](./yazi-usage.md)。herdr 键位不是 vim 风格，是 tmux 式前缀键 + 鼠标，学习成本反而更低。

---

## 目录

1. [先想明白：跑一队 AI agent，普通终端为什么不够用](#1-先想明白跑一队-ai-agent普通终端为什么不够用)
2. [安装](#2-安装)
3. [心智模型：一张图记住五个概念](#3-心智模型一张图记住五个概念)
4. [第 1 天必会：启动、分屏、detach 退出](#4-第-1-天必会启动分屏-detach-退出)
5. [灵魂功能：agent 状态侧边栏](#5-灵魂功能agent-状态侧边栏)
6. [CLI 入门：让脚本和 agent 指挥 herdr](#6-cli-入门让脚本和-agent-指挥-herdr)
7. [持久化的三种含义（最容易踩的认知坑）](#7-持久化的三种含义最容易踩的认知坑)
8. [远程与多机](#8-远程与多机)
9. [配置入门](#9-配置入门)
10. [进阶一瞥：worktree、插件、--skill](#10-进阶一瞥worktree插件--skill)
11. [速查表](#11-速查表)
12. [常见问题](#12-常见问题)
13. [3 天练习路线](#13-3-天练习路线)

---

## 1. 先想明白：跑一队 AI agent，普通终端为什么不够用

先看一个每天都在发生的场景：

```
 你同时开着 4 个 AI agent：
 ├── agent A 在改网站
 ├── agent B 在审查 A 的改动
 ├── agent C 在另一个项目跑测试
 └── agent D 停在那等你批准一条命令 —— 而你已经忘了它是哪个终端
```

AI agent 时代的新瓶颈不是「agent 不够强」，而是**注意力管理**：哪个还在干活？哪个卡住等批复？哪个悄悄干完了？关掉笔记本 lid 怎么全没了？

herdr（[GitHub](https://github.com/herdrdev/herdr)，Apache-2.0，单个 Rust 二进制，无账号无 Electron 无遥测）就是为这个问题造的：**一个认识 agent 的终端多路复用器**。最短解释是「**tmux for coding agents**」——

- 像 tmux：真实终端、后台常驻、detach 后重连一切还在、SSH 直连可用
- 不像 tmux：它知道每个 pane 里跑的是 Claude Code 还是 Codex，是**干活中 / 等你批准 / 干完了 / 空闲**，全显示在侧边栏
- 比 tmux 更现代：鼠标原生（点击聚焦、右键分屏、拖边框调大小），还有一整套 CLI，脚本和 agent 也能指挥它

### 和现有方案对比

| 能力 | 多开终端窗口 | tmux / Zellij | herdr |
|---|---|---|---|
| 跑真实终端进程 | ✅ | ✅ | ✅ |
| 关掉窗口工作还活着（detach/重连） | ❌ | ✅ | ✅ |
| 分屏、标签页 | 看终端脸色 | ✅ | ✅ |
| 认识 pane 里的 agent，显示状态 | ❌ | ❌ | ✅ working/blocked/done 一眼看清 |
| 鼠标原生（点击/右键/拖动） | 部分 | 弱 | ✅ |
| CLI + socket API，agent 能指挥 agent | ❌ | 只有通用终端控制 | ✅ |
| SSH 场景直接可用 | ✅ | ✅ | ✅ |
| 一个窗口看多台机器 | ❌ | 一个客户端一个服务器 | ✅（0.9+） |

> **herdr 不做什么**（先打预防针）：它不隔离文件改动（同目录两个 agent 会打架，要靠 worktree/任务划分，见第 10 节）、不是任务管理平台、对 agent 的识别也不是 100% 准（见第 12 节）。

> tmux 已经够用、只跑一个 agent 的场景不必换。**同时跑 2 个以上 agent、或工作全在 SSH 服务器上**，herdr 的价值立刻显现。

---

## 2. 安装

### 2.1 安装命令

```bash
# 官方安装脚本（macOS / Linux）
curl -fsSL https://herdr.dev/install.sh | sh

# 或者用包管理器
brew install herdr          # Homebrew
mise use -g herdr           # mise
# Windows 处于 preview 渠道：powershell -ExecutionPolicy Bypass -c "irm https://herdr.dev/install.ps1 | iex"
```

验证与补全：

```bash
herdr --version             # 出版本号即装好（写作时最新 v0.9.1，2026-09-16 发布）
source <(herdr completion zsh)   # zsh 补全，建议写进 ~/.zshrc（还支持 bash/fish 等）
```

### 2.2 升级

```bash
herdr update                # 安装脚本装的用它升级；brew/mise 走各自的包管理器
herdr status                # 看客户端和 server 各自的版本
```

> **升级的坑**：herdr 是「客户端 + 后台 server」结构（见第 3.2 节）。二进制升级后，后台 server 可能还是旧版在跑。`herdr status` 会告诉你版本是否一致；需要重启 server 时 `herdr server stop && herdr`——**stop 会杀掉 pane 里所有进程**，先干完活！实验性的 `herdr update --handoff` 可以平滑交接。

### 2.3 本机注意事项

本机 Ubuntu 22.04，glibc 2.35。参考 yazi 的教训（见 [`yazi-usage.md`](./yazi-usage.md) 第 2 节）：新版预编译二进制可能要求更高的 glibc。**若 `herdr --version` 报 `GLIBC_x.xx not found`**，去 [releases 页](https://github.com/herdrdev/herdr/releases) 找静态/musl 构建或改用 mise 安装。

装好后在此补记：版本 ____，安装路径 ____，补全是否配置 ____。（2026-09-22 检查：未安装）

---

## 3. 心智模型：一张图记住五个概念

### 3.1 层级结构

herdr 的所有概念就五层，大白话对应关系：

```
 session（会话）＝ 一个后台大管家（默认就一个，够用）
 └── workspace（工作区）＝ 一个项目/一个仓库
     ├── tab（标签页）＝ 项目的某个视图，如 agents / dev / tests
     │   ├── pane（分屏格）＝ 一块真实的终端
     │   │   └── 跑着 claude  ← herdr 认出它是个 agent，开始跟踪状态
     │   └── pane（分屏格）＝ 一块真实的终端
     │       └── 跑着 npm run dev  ← 只是普通进程，herdr 不跟踪状态
     └── tab: tests
         └── pane ── 跑着 npm test
```

| 概念 | 一句话 | 类比 |
|---|---|---|
| **session** | 持久化的后台服务器命名空间 | 大管家（绝大多数人只需要默认的这一个） |
| **workspace** | 一个项目，拥有自己的 tabs/panes，并汇总里面 agent 的状态 | IDE 里的「项目」 |
| **tab** | 项目内的一种布局/视图 | 浏览器标签页 |
| **pane** | 一块**真实终端**（不是聊天窗口的转述！），跑什么完全由你 | tmux 的 pane |
| **agent** | herdr 在某个 pane 里**认出来**的编码 agent 进程 | —— |

> 关键区别：**pane 永远存在**（哪怕只跑个 shell），**agent 是 herdr 认出来之后才有的概念**。dev server、测试、`ssh` 都只是 pane；只有 Claude Code、Codex 这类 agent 才会进侧边栏的状态列表。

经验法则：**一个仓库一个 workspace**，切 workspace = 切换整个项目上下文；长期任务、交互过程全在真终端里，不在聊天记录里。

### 3.2 客户端 / 服务器架构

`herdr` 命令看起来是一个程序，其实内部有两个角色：

```
 终端1（客户端，只负责显示+收输入）──┐
 终端2（客户端）───────────────────┼──▶ herdr 后台 server（真正持有进程）
 SSH 上来的客户端 / 手机 ────────────┘         │
                                          ├─ pane PTY：claude（agent）
                                          ├─ pane PTY：npm run dev
                                          └─ pane PTY：npm test

 关掉客户端 = 只关掉「显示器」；server 和里面的进程照常活着
```

这就是「关掉终端工作不丢」的原理：**进程归 server 管，不归你的终端窗口管**。多个客户端可以同时连同一个 server。

---

## 4. 第 1 天必会：启动、分屏、detach 退出

### 4.1 启动

```bash
cd ~/myproject
herdr
```

首次运行：创建/连接默认后台 session；没有 workspace 就自动建一个。左边是侧边栏（上半 workspace 列表、下半 agent 状态），右边是一个正常的 shell——就这么简单，没有任何 socket 配置要记。

界面长这样（示意）：

```
 ┌────────────────┬─────────────────────────────────────────┐
 │  侧边栏          │  pane 区（真实终端们）                      │
 │                │ ┌───────────────────┬───────────────────┐│
 │ ▾ 工作区         │ │ ● builder          │ ○ reviewer        ││
 │   myproject     │ │  (claude·干活中)    │  (codex·空闲)      ││
 │                │ │ $ claude           │ $ codex           ││
 │ ▾ agents        │ │  > 正在修改 a.py…   │  等待任务…          ││
 │   ⏳ builder    │ └───────────────────┴───────────────────┘│
 │   ✔ reviewer   │                                          │
 └────────────────┴─────────────────────────────────────────┘
   左边看「谁需要我」，右边是真终端；点谁聚焦谁，右键能分屏
```

### 4.2 鼠标（第一天就能全靠它）

| 动作 | 效果 |
|---|---|
| 点击 pane | 聚焦它 |
| 右键 pane | 菜单：分屏、新建 tab 等 |
| 拖动 pane 边框 | 调整大小 |
| 点击侧边栏的 workspace/agent | 直接跳过去 |

### 4.3 键盘：前缀键（tmux 风格）

herdr 用**前缀键**防止抢占 shell/agent 的正常按键：先按 `Ctrl+b` **松开**，再按动作键（手感类似 vim 的 leader 键）。

| 动作 | 按键 |
|---|---|
| 向右分屏 | `Ctrl+b` 然后 `v` |
| 向下分屏 | `Ctrl+b` 然后 `-` |
| 新建标签页 | `Ctrl+b` 然后 `c` |
| 下一个 / 上一个标签页 | `Ctrl+b` 然后 `n` / `p` |
| 聚焦左侧 pane（其余方向同理） | `Ctrl+b` 然后 `h` |
| workspace 导航 | `Ctrl+b` 然后 `w` |
| 放大/还原当前 pane（zoom） | `Ctrl+b` 然后 `z` |
| **detach（脱离）** | `Ctrl+b` 然后 `q` |
| **显示当前所有快捷键** | `Ctrl+b` 然后 `?` |

> 键位忘了永远按 `Ctrl+b ?`，比翻文档快。想改键位见第 9 节。

### 4.4 最重要的一组操作：detach 和重连

```bash
# 在 herdr 里按 Ctrl+b q —— 客户端关闭，但 server 和所有进程继续跑
# （等价于：直接关掉终端窗口 / SSH 断线，效果相同）

herdr                       # 随时重连，回到一模一样的活现场
herdr status                # 不进界面，看看 server 还在不在、版本对不对
```

> **核心认知：关终端窗口 ≠ 退出 herdr**。把终端窗口想成显示器，herdr server 是主机——显示器关了主机照常干活。

> 进阶：需要把两摊工作彻底隔离时才用命名 session：`herdr session attach work`、`herdr session attach experiments`。新手先用 workspace 就好。

---

## 5. 灵魂功能：agent 状态侧边栏

持久 pane 只是及格线，herdr 真正的卖点是：**不用挨个翻终端，侧边栏直接告诉你该看谁**。

### 5.1 五种状态

| 状态 | 大白话 | 你该做什么 |
|---|---|---|
| `working` | 正在干活 | 别打扰 |
| `blocked` | 停下来等你：批准命令/回答问题/做决定 | **去看它** |
| `done` | 在后台干完了，你还没看过 | 去看结果 |
| `idle` | 就绪待命，且它的 tab 你已经看过 | 可以派新活 |
| `unknown` | 认出是个 agent 但判断不了状态 | 用 `herdr agent explain` 查原因 |

> `done` 不是 agent 自己上报的「完成」，而是一种**注意力状态**：agent 在它的 tab 处于后台时变回了就绪，等你来瞄一眼。你看过它，就变回 `idle`。

状态是**向上汇总**的：pane → tab → workspace。某个后台 workspace 里的 agent 卡住了，侧边栏的 workspace 行上就能看到，不用切过去才知道。

### 5.2 状态怎么流转

```
        ┌─────────┐   提交任务    ┌─────────┐  要批准/要回答  ┌─────────┐
        │  idle   │ ──────────▶ │ working │ ────────────▶ │ blocked │
        │ 就绪待命 │             │  干活中  │ ◀──────────── │ 等你发话 │
        └─────────┘             └────┬────┘   你给出了输入    └─────────┘
             ▲                       │
             │    你正看着它的 tab，       │ 后台跑完了（你没在看）
             │    干完直接回 idle        ▼
             │                   ┌─────────┐
             └─────────────────  │  done   │   看过它的 tab ──▶ 回 idle
                    查看即确认     │ 等你来看 │
                                  └─────────┘
```

### 5.3 herdr 怎么认出 agent 的？

两层机制，**不需要你配置任何东西**：

1. **进程识别 + 屏幕匹配（screen manifests）**：herdr 看 pane 前台进程是谁，再看终端屏幕底部的内容是否匹配已知 agent 的界面特征。开箱即认：Claude Code、Codex、Cursor Agent CLI、OpenCode、GitHub Copilot CLI、Devin、Kimi、Droid、grok 等。
2. **官方集成（integration）**：给用得最多的 agent 装上，能上报**精确状态**和**原生会话 ID**（后者用于 server 重启后续聊，见第 7 节）：

```bash
herdr integration install claude     # Claude Code 用户装这个
herdr integration install codex      # 用 Codex 的装这个
herdr integration status             # 看装了哪些
```

> 状态看起来不对劲时，问 herdr 自己：
>
> ```bash
> herdr agent explain builder             # 哪个检测源、哪条规则判的
> herdr agent explain builder --verbose   # 更详细：证据、回退原因
> ```
>
> `unknown` 不代表失败，`idle` 也不绝对代表干完了——**行动前先读一眼 pane**。

---

## 6. CLI 入门：让脚本和 agent 指挥 herdr

> 这一节可以先跳过，用熟界面再回来。但它是 herdr 和 tmux 拉开差距的地方：脚本不用「模拟敲键盘 + sleep 30 秒」，而是直接问「agent 干完了吗」。

### 6.1 命令套路和 ID

所有命令一个套路：`herdr <对象> <动作>`，对象有 `workspace` / `tab` / `pane` / `agent` / `worktree` / `machine` / `integration` / `server`。

每个对象都有稳定 ID：workspace `w1`、tab `w1:t1`、pane `w1:p2`。**脚本里永远从命令返回的 JSON 里取 ID**，不要瞎猜：

```bash
created=$(herdr tab create --workspace w1 --label tests --no-focus)
pane=$(printf '%s\n' "$created" | jq -r '.result.root_pane.pane_id')
```

agent 名字规则：小写字母开头，`[a-z0-9_-]`，最长 32 位（如 `builder`、`agent-20260922`）。

### 6.2 常用命令

**pane 命令（管普通进程）**：

| 目的 | 命令 |
|---|---|
| 列出 panes | `herdr pane list --workspace w1` |
| 分屏 | `herdr pane split --current --direction right` |
| 跑命令 | `herdr pane run w1:p3 "npm test"` |
| 等特定输出出现 | `herdr pane wait-output w1:p3 --regex "passed\|failed" --timeout 120000` |
| 读最近的屏幕输出 | `herdr pane read w1:p3 --source recent-unwrapped --lines 120` |

**agent 命令（管 agent）**：

| 目的 | 命令 |
|---|---|
| 列出 agents | `herdr agent list` |
| 在指定 pane 启动 agent | `herdr agent start reviewer --kind claude --pane w1:p2` |
| 派活并等它干完 | `herdr agent prompt reviewer "审查当前 diff" --wait` |
| 只等它卡住（来问问题） | `herdr agent wait reviewer --until blocked` |
| 读它的输出 | `herdr agent read reviewer --source recent-unwrapped --lines 120` |
| 解释状态判定 | `herdr agent explain reviewer --verbose` |

> **pane 还是 agent？** 普通进程（dev server、测试）用 pane 命令，靠匹配输出文本判断；agent 用 agent 命令，直接等**状态**（`blocked` 等的是「真的在等你」，不是屏幕上碰巧有 "blocked" 这个词）。
>
> `agent start` 的前提：那个 pane 的 shell 正停在提示符上，没有别的程序在前台。

### 6.3 完整实战：builder 干活，reviewer 审查

一个具体的会话：改动一个功能、留着 dev server、另一个 agent 负责审查。

```bash
cd ~/myproject && herdr

# 0. 给 workspace 起个名（先 list 看 ID，ID 以实际返回为准）
herdr workspace list
herdr workspace rename w1 myproject

# 1. 建一个 agents 标签（--no-focus 表示不抢当前焦点）
agents=$(herdr tab create --workspace w1 --cwd "$PWD" --label agents --no-focus)
builder_pane=$(printf '%s\n' "$agents" | jq -r '.result.root_pane.pane_id')

# 2. 向右劈一个 pane 给 reviewer
review=$(herdr pane split "$builder_pane" --direction right --cwd "$PWD" --no-focus)
review_pane=$(printf '%s\n' "$review" | jq -r '.result.pane.pane_id')

# 3. dev server 不是 agent，用 pane 命令（先建个 dev 标签）
dev=$(herdr tab create --workspace w1 --cwd "$PWD" --label dev --no-focus)
dev_pane=$(printf '%s\n' "$dev" | jq -r '.result.root_pane.pane_id')
herdr pane run "$dev_pane" "npm run dev"

# 4. 启动两个 agent
herdr agent start builder  --kind claude --pane "$builder_pane"
herdr agent start reviewer --kind claude --pane "$review_pane"

# 5. 给 builder 派活并等结果（--wait 直到 idle/done/blocked）
herdr agent prompt builder \
  "实现 XX 功能，跑相关测试，先不要提交" --wait --timeout 600000

# 6. 若返回 blocked：先读它的屏幕，看它要什么再答复
herdr agent read builder --source recent-unwrapped --lines 120

# 7. 让 reviewer 审查（审查方只读不写！）
herdr agent prompt reviewer \
  "审查当前 diff，只报告可执行的意见，带文件和行号" --wait --timeout 600000
```

最后的决定权在你：看 diff（配 lazygit，见姊妹篇）、看测试输出、看审查意见，然后决定是否提交。herdr 负责把终端和状态组织好，**不替你做判断**。

### 6.4 让 agent 自己指挥 herdr

```bash
herdr --skill
```

输出一份 herdr 使用说明（skill），喂给 agent 后它就会**自己开 pane、自己分屏、自己派活**。Shopify CEO Tobi Lütke 的名场面演示：开一个 workspace、启动一个 agent，让它读完 `herdr --skill` 然后自己分出十个 pane。实用性姑且不论，这说明 CLI 面是完整的——人和 agent 用同一套接口操作同一个系统。

---

## 7. 持久化的三种含义（最容易踩的认知坑）

「persistent」这个词在 herdr 里指三件不同的事，混起来必然踩坑：

```
 发生了什么事？
 ├── ① 只是关了终端窗口 / SSH 断了（detach）
 │      └─▶ server 和所有进程都活着；herdr 重连 = 回到原样 ✔ 最常见，随便关
 │
 ├── ② herdr server 停了（升级重启 / 机器重启）
 │      ├─▶ workspace、tab、pane 布局、目录、焦点结构：会恢复
 │      ├─▶ 普通进程（dev server、测试）：没了，要重新跑
 │      └─▶ 装了集成的 agent：对话可以续（如 claude resume），靠集成上报的会话 ID
 │
 └── ③ pane 屏幕历史（实验功能，默认关）
        └─▶ config 里 pane_history = true 才会把最近屏幕内容存盘
             注意：屏幕内容可能含密钥/代码，默认不存是故意的，想清楚再开
```

**一句话记忆：detach 保进程，重启保布局。**长期任务（构建、部署、agent 跑长活）放心 detach；但别指望 server 重启后 `npm run dev` 还活着。

---

## 8. 远程与多机

### 8.1 最简单的远程用法：就是 SSH

```bash
ssh you@server
herdr                    # 在服务器上跑，server/agent/进程全在服务器上
# 断线、合盖、换电脑…… 回来 ssh 上去再敲 herdr，现场原样回来
```

界面自适应窄终端，所以**手机 SSH 上来瞄一眼哪个 agent 卡住了**也是现实操作。

### 8.2 本机当遥控器：`--remote`

```bash
herdr --remote workbox                        # workbox 是 ~/.ssh/config 里的主机名
herdr --remote ssh://you@server:2222          # 或者完整地址
```

本地二进制变成远程 server 的瘦客户端：干活在服务器，但本地能力保留（比如把本地剪贴板里的图片粘进远程会话）。

### 8.3 多台机器一个窗口（0.9+）

```bash
herdr machine add workbox --label "构建机"    # 保存一台远程机器
herdr machine list                           # 管理：rename / disable / enable / remove
```

之后打开 herdr，侧边栏同时出现 **Local** 和 **构建机**：点谁操作谁（键盘鼠标和屏幕流跟着切），其他机器继续上报 agent 状态和通知——远程 agent 卡住了，你在本地 pane 里打字也能看见。

> 细节：
> - 添加时 herdr 会检查 SSH 连通性、远程的 herdr 版本、server 是否在跑；需要装/换东西会先问你（默认 No，因为换 server 会停进程）。
> - profile 不存密码/私钥，认证仍走 OpenSSH；带密码短语的 key 先 `ssh-add`。
> - 某台机器断线：它的状态还在侧边栏（变暗的缓存），不能输入，其他机器不受影响。
> - **CLI 一次只对一台 server 说话**：自动化脚本请到「拥有那台 agent 的机器上」执行。
> - 官方还在做 herdr Cloud（免配置 SSH 的中继，端到端加密），写作时是 waitlist。

---

## 9. 配置入门

不配置完全能用。想调时写 `~/.config/herdr/config.toml`，只写要改的项，改完不用重启：

```bash
herdr server reload-config
```

常用配置示例（按需摘取）：

```toml
# ~/.config/herdr/config.toml

[keys]
prefix = "ctrl+b"              # 前缀键
new_tab = "prefix+c"
next_tab = "prefix+n"
previous_tab = "prefix+p"
focus_pane_left = "prefix+h"
split_horizontal = "prefix+minus"

[ui.toast]                     # agent 需要/完成时的通知
delivery = "herdr"             # 还可选终端提醒、系统通知、"none" 关闭
delay_seconds = 1

[ui.toast.herdr]
position = "bottom-right"      # 正在看的 tab 不弹，只提醒你没盯着的

[ui.sidebar.agents]            # 侧边栏 agent 行显示什么
rows = [
  ["state_icon", "agent", "state_text"],
  ["workspace", "tab"],
]

[worktrees]                    # worktree 检出目录（见第 10 节）
directory = "~/.herdr/worktrees"

# [experimental]               # 慎重：屏幕历史可能含密钥
# pane_history = true
```

> 其他：首次运行的引导做完后可加 `onboarding = false` 关掉。日志在 `~/.config/herdr/`（herdr.log / herdr-client.log / herdr-server.log），诊断时 `HERDR_LOG=herdr=debug herdr` 开 debug。**日志可能含终端内容，别直接贴网上。**

---

## 10. 进阶一瞥：worktree、插件、--skill

### 10.1 worktree：并行 agent 不打架的正解

两个 agent 在同一目录改同一个文件必然冲突。herdr 把 Git worktree 包装成了一等公民：一条命令 = `git worktree add` + 新建一个 workspace，并在侧边栏归到仓库名下。

```bash
herdr worktree create --cwd ~/myproject --branch fix-header --no-focus
# 分支存在就检出，不存在就从 --base（默认 HEAD）新建
herdr worktree remove --workspace <workspace-id>   # 删检出，不删分支；有未提交改动要 --force
```

配合脚本，每个任务一行：

```bash
#!/bin/sh   # 存成 spawn，chmod +x
set -e
repo=${REPO:-$PWD}
stamp=$(date +%Y%m%d-%H%M%S)
created=$(herdr worktree create --cwd "$repo" --branch "agent/$stamp" --label "$stamp" --no-focus)
pane=$(printf '%s\n' "$created" | jq -r '.result.root_pane.pane_id')
herdr agent start "agent-$stamp" --kind claude --pane "$pane"
herdr agent prompt "agent-$stamp" "$1"
```

```bash
spawn "修复移动端 header 重叠"
spawn "加一条 sitemap 路由"
spawn "升级测试框架并修好挂掉的用例"
```

三条命令 = 三个独立检出 + 三个 agent，侧边栏看着，谁 blocked 谁 done 一目了然。

> worktree 解决「改同一个文件」，解决不了「改出互相不兼容的设计」——任务划分和最终 review 仍然是你的活。

### 10.2 插件

插件本质是带 `herdr-plugin.toml` 清单的可执行包（Bash/JS/Lua/Rust 都行），没有独立 SDK——**整个 herdr CLI 就是插件的 API**。典型用途：新 worktree 自动装依赖、agent blocked 时发通知、开部署面板弹窗等。

市场在 [herdr.dev/plugins](https://herdr.dev/plugins)，本质是 GitHub 上打了 `herdr-plugin` 标签的仓库的**自动索引**（每 30 分钟刷新一次；2026-09-22 检索时已有 1200+ 插件），不是人工审核的应用商店：

```bash
herdr plugin install ogulcancelik/herdr-plugin-examples/worktree-bootstrap
```

先会管理插件的四条命令，再谈装：

```bash
herdr plugin list                      # 已装了什么
herdr plugin action list --plugin ID   # 这个插件提供哪些动作
herdr plugin action invoke ID.ACTION   # 手动触发一个动作
herdr plugin log list --plugin ID      # 排查插件问题看日志
```

#### 初学者推荐（多篇第三方评测一致点名；star 数为 2026-08 时点数据）

| 插件 | 解决什么痛点 | 安装 |
|---|---|---|
| **herdr-plus**（253★，呼声最高） | 每开一个项目都要手工搭一遍布局。**Projects**：一个 TOML 文件定义整个 workspace（tabs、panes、工作目录、启动命令），一键拉起，还能直接开成 git worktree；**Quick Actions**：模糊搜索一键跑常用脚本 | `herdr plugin install cloudmanic/herdr-plus` |
| **herdr-spreader** | tmuxp 老用户的迁移路径：YAML 声明式布局文件，支持嵌套分屏、每 pane 工作目录/环境变量、`wait_for`（等某 pane 输出匹配后再开下一个） | `herdr plugin install yuk1ty/herdr-spreader` |
| **Herdr Board** | 终端里直接开看板（Kanban），卡片式管理手头任务 | 市场搜 "board" |
| **Agent Quota** | 一个视图看 Claude / Gemini / Codex 各烧了多少额度 | 市场搜 "quota" |
| **Herdr Resurrect** | 把配置好的 spaces/tabs 存成模板，跨机器迁移或分享给别人 | 市场搜 "resurrect" |

生态里其他按 star 数的热门（2026-08）：**herdr-reviewr**（496★，diff 侧栏代码审查，可评论、打回 agent 的工作）、**herdr-remote**（278★，菜单栏/手机/Telegram 遥控）、**herdr-sidebar**（167★，VS Code 风格文件树 + git 侧栏）。逛市场还可以看精选目录 [awesome-herdr](https://github.com/yigitkonur/awesome-herdr)。

> **插件用你的权限跑本地命令，装前先读源码**——当成对待任何来路不明的可执行文件。市场无人审核，装前四步：打开仓库 → 读 `herdr-plugin.toml` → 看它声明的命令跑的脚本源码 → 确认信任作者。
>
> 新手节奏：**先只装 herdr-plus**（治「每次开项目重复搭布局」这个最大痛点），用顺了再按需加审查/额度/遥控类。

---

## 11. 速查表

### 救命三键

```
Ctrl+b ?    忘了任何键位就按它（列出全部绑定）
Ctrl+b q    detach：关掉界面，工作继续跑
herdr       重连，回到现场
herdr status 在外看看 server 状态和版本
```

### 前缀键（先 Ctrl+b 松开，再按动作键）

```
v / -       右分屏 / 下分屏        c          新标签页
n / p       下一个/上一个标签       w          workspace 导航
z           放大当前 pane          h j k l    移动 pane 焦点
q           detach                ?          键位帮助
```

### 鼠标

```
点 pane      聚焦                  拖边框      调大小
右键         分屏/新 tab 菜单        点侧边栏    跳到 workspace/agent
```

### CLI 常用

```
herdr status                          客户端/server 版本与状态
herdr workspace list|create|rename    工作区管理
herdr tab create --workspace w1 ...   建标签
herdr pane split --current --direction right   分屏
herdr pane run <pane> "cmd"           在 pane 里跑命令
herdr pane read <pane> --source recent-unwrapped --lines 120   读输出
herdr agent list                      列出 agent
herdr agent start <名字> --kind claude --pane <pane>            启动 agent
herdr agent prompt <名字> "任务" --wait  派活并等结束
herdr agent wait <名字> --until blocked 只等它来提问
herdr agent explain <名字> --verbose   为什么是这个状态
herdr integration install claude       装集成（精确状态+重启续聊）
herdr worktree create --cwd ... --branch ...   worktree 即 workspace
herdr machine add <主机> --label ...   多机一窗（0.9+）
herdr plugin install <owner/repo>      装插件（市场 herdr.dev/plugins）
herdr plugin list                      已装插件
herdr server reload-config             配置热重载
herdr --skill                          输出喂给 agent 的使用说明
```

### agent 状态

```
working 干活中   blocked 等你批准/回答   done 后台干完没看过
idle 就绪(已看过) unknown 认不出(agent explain 查)
```

---

## 12. 常见问题

**Q1：herdr 没认出我的 agent？**
先确认前台进程确实是 agent（`herdr pane process-info --current`），再 `herdr agent explain <目标>` 看判定依据。**最常见的坑：在 pane 里又套了一层 tmux**——herdr 看到的前台是 tmux 而不是 agent。tmux 想用就套在 herdr 外面，别插在 herdr 和 agent 之间。检测规则可更新：`herdr server update-agent-manifests`。不在支持列表里的 agent 也能正常跑，只是没有状态跟踪。

**Q2：状态显示不对 / 一直是 unknown？**
`herdr agent explain <目标> --verbose` 看匹配了哪条规则、证据是什么。agent 界面改版会导致误判，`unknown` 就是老老实实告诉你「我不确定」——别拿它当「完成了」的信号，读一眼 pane 再行动。

**Q3：升级后界面/功能没变？**
`herdr status` 大概率显示 client 新、server 旧。`herdr server stop && herdr` 换新 server——**stop 会杀掉 pane 里的进程**，先收尾；或者试实验性的 `herdr update --handoff`。

**Q4：`Ctrl+b` 按了没反应？**
被外层终端或桌面快捷键吃掉了。`Ctrl+b ?` 确认绑定还在，然后排查终端/桌面的快捷键冲突。想换直接键，官方建议优先挑没用过的 `Ctrl+Alt` 组合。

**Q5：能和 tmux 共存吗？**
能。tmux 套在 herdr 外面（外层复用器）没问题；herdr 的 pane 里跑 tmux 也没问题，但那样 herdr 就认不出 agent 了（见 Q1）。

**Q6：两个 agent 改同一个文件打架？**
herdr 不管文件隔离，这是设计边界。解法：给任务划清范围（各改各的模块）、或用 worktree（第 10.1 节）、或按 builder/reviewer 分工（第 6.3 节）。Git 和 review 兜底。

**Q7：远程机器连不上 / 侧边栏显示 Attention？**
后台连接不会弹交互提示。先在普通终端里手动 `ssh workbox` 排查（key 密码短语要先 `ssh-add`），再 `herdr --remote workbox` 交互处理一次（装依赖、确认 host key），然后重启客户端让它重连。远程和本地 herdr 版本有差异不一定要动它，herdr 自己协商兼容。

**Q8：键位像 vim 吗？**
不像。herdr 是「前缀键 + 鼠标」流派，没有模式切换，上手反而比 vim 系工具（yazi/vim）更快。会 tmux 的话 prefix 手感直接迁移。

**Q9：和 yazi 那样有任务队列之类的东西吗？**
没有对应物。herdr 管的是「终端进程和 agent 状态」，不是文件操作。它是纯 Rust 单二进制，也没有内置浏览器/聊天面板——你要的「多 agent 总控台」就是侧边栏 + CLI。

---

## 13. 3 天练习路线

| 天 | 任务 | 达标标准 |
|---|---|---|
| 1 | 装好 herdr；`cd` 到一个测试仓库启动；纯鼠标分屏/建 tab；`Ctrl+b q` detach 后**直接关掉终端窗口**，重开 `herdr` 回来 | 能 detach/重连，并亲眼确认 detach 期间后台进程还活着（比如 pane 里跑个 `ping`，detach 后重连还在刷） |
| 2 | 在 pane 里启动 claude 干个小活，盯侧边栏看状态流转 working→blocked→done→idle；练 `Ctrl+b z` 放大、`w` 切 workspace；`herdr integration install claude` | 不翻终端，仅凭侧边栏说出每个 agent 的状态 |
| 3 | 走一遍第 6.3 节的 CLI 实战（builder+reviewer）；把 `spawn` 脚本跑起来开 3 个 worktree 并行任务；试试把 `herdr --skill` 的输出喂给 agent 让它自己排版 | 能用 CLI 完成「启动→派活→等结束→读结果」全流程 |

之后就是日常自然变强。忘了按键第一反应：**`Ctrl+b ?` 查键位，`herdr --help` 查命令**。

官方文档：[quick-start](https://herdr.dev/docs/quick-start/) · [concepts](https://herdr.dev/docs/concepts/) · [supported agents](https://herdr.dev/docs/agents/) · [configuration](https://herdr.dev/docs/configuration/) · [socket api](https://herdr.dev/docs/socket-api/)

---

*生成于 2026-09-22。基于 herdr v0.9.1（2026-09-16 发布；GitHub herdrdev/herdr，Apache-2.0，40k+ stars）的官方文档、README 与 Flavio Copes 深度评测（均经 Tavily 检索核对）。*
*10.2 节插件推荐另据 2026-09-22 的 Tavily 检索（Josh Finnie 博客、Developers Digest 生态分析、flaviocopes 插件指南、YouTube 教程）；插件数与 star 数均为检索时点数据。*
*本机（Ubuntu 22.04）截至 2026-09-22 尚未安装；安装后请在第 2.3 节补记版本、路径与补全配置。*
