# Agent Reach — 架构与用途速读（AI-readable notes）

> 本文件供 AI Agent / 开发者快速理解本项目的**用途、架构与用法**，所有主张均带 `file:line` 证据。
> 生成方式：对全仓库做并行深读 + 关键事实抽查验证。代码版本：`agent_reach/__init__.py` = `1.4.0`。
> 语言：简体（与 `README.md` / `docs/` 既有文档一致）。

---

## 1. 一句话定位

**Agent Reach 是一个 Python CLI + library，帮任何「能跑 shell 命令的 AI Agent」一键装上「读取 / 搜索 16 个互联网平台」的能力。**

它的本质是 **installer + doctor + config 工具**，**不是 wrapper，也不是 framework**：
- CLI 入口 `cli.py` 里**没有 `read` / `search` 子命令**，只有安装/诊断/配置类命令（`agent_reach/cli.py:2-10` docstring 明确限定范围）。
- MCP server 只暴露一个 `get_status`（即 doctor 报告），**不代理任何读取/搜索**（`agent_reach/integrations/mcp_server.py:6-8`）。
- 核心 `AgentReach` class 只有 `doctor()` / `doctor_report()`（`agent_reach/core.py:23-43`）。

> **关键认知**：装完之后，Agent **直接调用上游工具**（`yt-dlp` / `twitter-cli` / `rdt-cli` / `gh` / `mcporter` 等），不经过 Agent Reach 的包装层（README.md:157-163 自称「脚手架 scaffolding，不是框架」）。

---

## 2. 它解决什么问题（用途）

AI Agent 会写代码改文档，但上网找东西就抓瞎——每个平台都有门槛：Twitter API 付费、Reddit 服务器 IP 被 403、小红书必须登录、B站海外 IP 被屏蔽、YouTube 拿不到字幕、网页抓回一堆 HTML 标签、没有免费好用的搜索（README.md:19-35）。

Agent Reach 把「逐个踩坑、装工具、调配置、清洗数据」收敛成**一句话**，并自动处理工具选型、安装、认证、Cookie、诊断。

卖点（README.md:57-61 / 90-91 / 241 / 356-358）：
- 💰 **全免费** — 工具开源、API 免费；唯一可能花钱是服务器代理（~$1/月），本地不用。
- 🔒 **隐私** — Cookie 仅存本地 `~/.agent-reach/config.yaml`，权限 `600`（`agent_reach/config.py:48-66` 用 `os.open` + `0o600` 防 world-readable race），不上传。
- 🔄 **持续追更** — 上游工具由项目盯着更新。
- 🤖 **通吃所有 Agent** — Claude Code / OpenClaw / Cursor / Windsurf。
- 🩺 **自带诊断** — `agent-reach doctor`。

---

## 3. 四层架构

```
散发层   SKILL.md(意图路由) · MCP server(get_status) · guides/setup-*.md
         · config/mcporter.json(Exa + 小红书 MCP) · llms.txt(AI 发现入口)
入口层   cli.py(1743 行 argparse)：install/doctor/configure/setup/skill/...  ← 只做安装、诊断、配置
核心层   core.py(门面) · config.py(YAML+env) · doctor.py · channels/__init__.py(ALL_CHANNELS 注册表)
Channel  每平台一文件，继承 base.Channel：youtube/reddit/twitter/xiaohongshu/... ×16  ← 只负责「检测 + 描述上游工具」
```

目录映射（见 `CLAUDE.md` Structure 段）：
- `agent_reach/cli.py` — CLI 入口（argparse）
- `agent_reach/core.py` — 门面，仅 doctor
- `agent_reach/config.py` — 配置管理（YAML + env 回退）
- `agent_reach/doctor.py` — 诊断引擎
- `agent_reach/channels/` — 每平台一文件，继承 `base.py`
- `agent_reach/integrations/mcp_server.py` — MCP 集成
- `agent_reach/skill/` + `agent_reach/guides/` — Agent skill 与配置指南
- `config/mcporter.json` — MCP 工具配置

### Channel 契约的真相（重要：纠正 CLAUDE.md）

`CLAUDE.md` 写「channel 必须实现 `can_handle/read/search/check`」，**但实际 `agent_reach/channels/base.py:26-29` 只有 `can_handle` 是 `@abstractmethod`**；`check()` 有默认实现（`base.py:31-36`），`read`/`search` 是各 channel **零散自行实现**（多数 channel 两个都没有，例 `channels/web.py:22-32` 有 `read`），而且**没有中央 URL→channel 派发器**——`core.py` 的 `AgentReach` 只做 doctor。也就是说 channel 真正在做的事是 `check()`（平台可用性探测），呼应「装好、确认通，读取交给上游」的定位。

### doctor 引擎

`agent_reach/doctor.py:12-106`：遍历 `ALL_CHANNELS` 逐个跑 `check()`（例 YouTube 检查 `which yt-dlp` → JS runtime → yt-dlp config，见 `channels/youtube.py:23-46`），按 **Tier 0/1/2** 分组用 Rich 渲染，并检查 config 文件权限是否过松。

---

## 4. 支持的 16 个平台与上游工具

| 平台 | 上游工具 | 认证 | Tier | 证据 |
|---|---|---|---|---|
| 网页 | Jina Reader (`r.jina.ai`) | 免配置 | 0 | channels/web.py:22-32 |
| YouTube | `yt-dlp`（需 deno/node） | 免配置 | 0 | channels/youtube.py:15,24-46 |
| RSS | feedparser | 免配置 | 0 | channels/rss.py |
| 全网搜索 | Exa MCP（via mcporter，免 Key） | 免配置 | 0 | channels/exa_search.py:12,15-37 |
| GitHub | `gh` CLI | 公开仓库免配置 | 0 | channels/github.py:12,20-32 |
| 微信公众号 | Exa（via mcporter） | 免配置 | 0 | channels/wechat.py:30,38-63 |
| V2EX | V2EX 公开 JSON API | 免配置 | 0 | channels/v2ex.py:23,52-189 |
| 微博 | `mcp-server-weibo`（via mcporter） | 免配置 | 1 | channels/weibo.py:12,20-52 |
| Twitter/X | `twitter-cli` | 需 Cookie | 1 | channels/twitter.py:12,22-35 |
| B站 | `yt-dlp` + `bili-cli` | 海外需代理 | 1 | channels/bilibili.py:13,30,42-65 |
| Reddit | `rdt-cli` | 需登录 `rdt login` | 0/1 | channels/reddit.py:21,31-73 |
| 小红书 | `xhs-cli` | 需登录 Cookie | 1 | channels/xiaohongshu.py:121,131-156 |
| 雪球 | 雪球 JSON API | 需 `xq_a_token` Cookie | 1 | channels/xueqiu.py:24-135 |
| 小宇宙播客 | Groq Whisper + ffmpeg | 需免费 Groq Key | 1 | channels/xiaoyuzhou.py:13,21-54 |
| 抖音 | `douyin-mcp-server`（via mcporter） | 需配置 | 2 | channels/douyin.py:12,21-56 |
| LinkedIn | `linkedin-scraper-mcp` / Jina fallback | 公开页免配置 | 2 | channels/linkedin.py:12,19-41 |

> Tier 0 = 装好即用；Tier 1 = 需免费 Key 或登录；Tier 2 = 选配。`mcporter` 为 Exa/抖音/微博/LinkedIn/微信共用的 MCP 桥接。`ALL_CHANNELS` 注册 16 个（`channels/__init__.py:28-45`，`WebChannel` 置于最后作 catch-all）。

---

## 5. CLI 命令清单（来自 `cli.py` 完整 argparse）

| 命令 | 作用 | 证据 |
|---|---|---|
| `agent-reach install [--env local\|server\|auto] [--proxy] [--safe] [--dry-run] [--channels ...]` | 一键安装：系统依赖 + mcporter+Exa + 可选平台 + Cookie + doctor + skill | cli.py:62-74,155-321 |
| `agent-reach doctor` | 平台可用性诊断（顺带自动装 skill，`fixes #154`） | cli.py:89,1419-1431 |
| `agent-reach setup` | 交互向导：配 Exa/GitHub token/Reddit/Groq | cli.py:59,1434-1520 |
| `agent-reach configure <key> <value...>` | 设单一配置（keys: proxy, github-token, groq-key, twitter-cookies, youtube-cookies, xhs-cookies） | cli.py:77-86,1017-1122 |
| `agent-reach configure --from-browser chrome\|firefox\|edge\|brave\|opera` | 从浏览器自动抽取各平台 Cookie | cli.py:84-86 |
| `agent-reach skill --install \| --uninstall` | 安装/移除 SKILL.md 到 agent skill 目录 | cli.py:99-104,323-459 |
| `agent-reach format xhs` | 清洗 stdin 传入的小红书 API JSON | cli.py:107-108,462-481 |
| `agent-reach check-update` | 查 GitHub 是否有新版并给升级命令 | cli.py:111,1614-1675 |
| `agent-reach watch` | 排程用：仅在有问题时输出（健康+更新检查） | cli.py:114,1678-1739 |
| `agent-reach uninstall [--dry-run] [--keep-config]` | 卸载 config / skill / mcporter 条目 | cli.py:92-96,1321-1417 |
| `agent-reach version` / `--version` | 打印版本 | cli.py:117,128-130 |
| `agent-reach -v/--verbose` | 开 loguru INFO 日志 | cli.py:54-56 |

`--env=auto` 会自动判断 local（个人机）vs server（侦测 SSH/Docker/headless/云 VM 等 ≥2 个指标，`cli.py:975-1014`），server 才需代理。

---

## 6. 用法

### 方式 A — 给 AI Agent 用（主要设计）

把这句直接贴给 Agent，它会自己读文档、自己装完：
```
帮我安装 Agent Reach：https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/install.md
```
更新：
```
帮我更新 Agent Reach：https://raw.githubusercontent.com/Panniantong/agent-reach/main/docs/update.md
```
配某平台直接说「帮我配 Twitter / 帮我配小红书」，Agent 会引导（Cookie 类平台统一走浏览器 Cookie-Editor 导出）。
> OpenClaw 用户先开 exec 权限：`openclaw config set tools.profile "coding"`。

### 方式 B — 直接命令行
```bash
pipx install https://github.com/Panniantong/agent-reach/archive/main.zip   # 或 pip install -e .
agent-reach install --env=auto        # 一键
agent-reach install --env=auto --safe # 安全模式：只检查不自动装系统包
agent-reach doctor                    # 诊断
```
装完后 Agent 直接调用上游工具，例：全网搜索 `mcporter call exa.web_search_exa`、YouTube 字幕 `yt-dlp ...`。

### 作为 library
```python
from agent_reach.core import AgentReach
AgentReach().doctor_report()   # 仅暴露诊断；读取/搜索走上游工具
```

---

## 7. 文档一致性修复记录（2026-06-09）

原先发现 4 项不一致，已处理如下：

1. **版本号** — ✅ 已修复。`pyproject.toml:3` 与 `agent_reach/__init__.py:4` 本就是 `1.4.0`；本次把 `CLAUDE.md` 抬头 `Version: 1.3.0` → `1.4.0`，并在 `CHANGELOG.md` 补上 `## [1.4.0] - 2026-03-31`（上游工具迁移发布）与 `## [Unreleased]`（v1.4.0 tag 之后、尚未打新版号的 main 变更）。
2. **`CLAUDE.md` channel 契约描述** — ✅ 已修复。原称 channel 必须实现 `read/search`，实际 `base.py` 只有 `can_handle` 是 `@abstractmethod`、`check()` 有默认实现、无 `read/search`（见 §3）。已改为准确描述；同时修正「inherits from `BaseChannel`」（基类实际名为 `Channel`）与「`core.py` — Core read/search routing logic」（实际仅 doctor 门面）两处连带错误。
3. **平台数说法不一** — ✅ 已修复。统一为 **16**（`ALL_CHANNELS` 实有 16 个，README 表格亦 16 行）：`CLAUDE.md` `14+` → `16`、`pyproject.toml` 描述 `10+` → `16+`。验证时另发现第 5 处同类笔误：`skill/SKILL.md` 与 `skill/SKILL_en.md` 散文写「17 平台」，但两份文件自身只列举 16 个平台 → 一并改为 `16`。注：`~/.agents` / `~/.claude` 等**已安装**的 skill 副本需重跑 `agent-reach doctor` 或 `skill --install` 才会同步更新。
4. **`doctor` 自动装 skill** — ℹ️ 非缺陷，保持现状。这是 `fixes #154` 的刻意行为（`cli.py:1431`）：用户跑 `doctor` 时顺带补装缺失的 skill。属设计选择而非不一致，不作代码改动，仅在此记录。
