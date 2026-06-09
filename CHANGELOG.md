# Changelog / 更新日志

All notable changes to this project will be documented in this file.

本项目的所有重要变更都会记录在此文件中。

---

## [Unreleased]

### 🐛 Fixes / 修复
- doctor 现在能检测 Reddit 登录状态 / doctor now detects Reddit auth status (#261)
- 修复 Windows 上 doctor 误判微信公众号渠道不可用 / Fixed WeChat 公众号 false-negative in doctor on Windows (#263)

### 📚 Docs & i18n / 文档与多语言
- English skill locale support (#223)
- Korean README translation (#251); Douyin MCP install/config doc fixes (#256, #257)
- Added SECURITY.md (#254)

> 以上为 v1.4.0 tag 之后、尚未打新版本号的 main 分支变更 / Changes on `main` after the v1.4.0 tag, not yet released under a new version.

---

## [1.4.0] - 2026-03-31

### 🔄 Upstream Migrations / 上游工具迁移

After several upstream tools broke or were deleted, most channels were re-pointed to actively-maintained upstreams. 在多个上游工具失效/删除后，把大部分渠道重新指向活跃维护的上游工具。

- **Twitter/X:** migrated from the deleted `bird` CLI to `twitter-cli`; later supports both and never uninstalls user-installed tools (#231)
- **Reddit + B站:** Reddit switched to `rdt-cli`, added `bili-cli` for Bilibili; Reddit no longer needs a proxy (#235)
- **小红书 / XiaoHongShu:** switched from Docker MCP to `xhs-cli` (pipx install) (#236)
- **微信公众号 / WeChat:** search switched from miku_ai to Exa, upgraded to tier 0 (zero-config) (#233)
- **雪球 / Xueqiu:** added `rookiepy` as primary cookie backend, browser_cookie3 as fallback (#232)
- **YouTube:** platform-aware yt-dlp config path + locale-safe reads (#225)

### 🆕 New / 新增

- **小宇宙播客 / Xiaoyuzhou:** podcast audio-to-text via Groq Whisper (#124)
- **Web `read()`** via Jina Reader, plus a skill-install fallback (#238)
- Channel count / 渠道数量: 15 → 16

### 📦 Install & Skill / 安装与 Skill

- `agent-reach install` split into **core + optional channels** via a `--channels` flag (#237)
- Skill system refactored into a **router + references** structure (#192)

### 🧹 Removed / 移除

- Removed the trial Discord and Toutiao channels after upstream issues (#234)

---

## [1.3.1] - 2026-03-27

### 🐛 Bug Fixes / 修复

#### 📈 Xueqiu (雪球) — 全面修复

- **修复 400 错误根本原因：** `_ensure_cookies()` 仅访问首页只能获取 `acw_tc`（防 DDoS token），`xq_a_token` 由雪球前端 JS 动态生成，无法通过纯 HTTP 请求获取。新增三级 cookie 加载策略：① 读取 config 文件（`--from-browser` 保存的）→ ② 自动从本地 Chrome 浏览器提取（需安装 browser-cookie3）→ ③ homepage fallback
- **修复 User-Agent：** `"agent-reach/1.0"` 被雪球反爬系统识别拒绝，改为真实 Chrome UA
- **修复缺失 `Referer` 头：** 所有 API 请求加上 `Referer: https://xueqiu.com/`
- **修复 `get_hot_posts()` 端点：** 原端点 `/statuses/hot/listV3.json` 已废弃（返回空 body），改为 `/v4/statuses/public_timeline_by_category.json`，正确解析 `item.data` JSON 字符串获取 author/likes/text
- **修复 `urllib.request.quote` → `urllib.parse.quote`：** 明确使用正确模块
- **修复 `configure --from-browser` 不提取雪球 Cookie：** `PLATFORM_SPECS` 加入 Xueqiu，检测 `xq_a_token` 存在才保存
- **修正文档误导：** README/SKILL.md 中"无需配置"/"public API, no login required" → 准确描述需要 browser cookie
- **改善错误信息：** `check()` 失败时提示 `configure --from-browser chrome` 而非"可能需要代理"

---

## [1.3.0] - 2026-03-12

### 🆕 New Channels / 新增渠道

#### 💻 V2EX
- Hot topics, node topics, topic detail + replies, user profile via public JSON API
- Zero config — no auth, no proxy, no API key required
- `get_hot_topics(limit)`, `get_node_topics(node_name, limit)`, `get_topic(id)`, `get_user(username)`
- 通过公开 JSON API 获取热门帖子、节点帖子、帖子详情+回复、用户信息
- 零配置，无需认证、无需代理、无需 API Key

### 📈 Improvements / 改进

- Channel count: 14 → 15
- 渠道数量：14 → 15

---

## [1.1.0] - 2025-02-25

### 🆕 New Channels / 新增渠道

#### ~~📷 Instagram~~ (removed — upstream blocked)
- ~~Read public posts and profiles via [instaloader](https://github.com/instaloader/instaloader)~~
- **Removed:** Instagram's aggressive anti-scraping measures broke all available open-source tools (instaloader, etc.). See [instaloader#2585](https://github.com/instaloader/instaloader/issues/2585). Will re-add when upstream recovers.
- **已移除：** Instagram 反爬封杀导致所有开源工具（instaloader 等）失效。上游恢复后会重新加回。

#### 💼 LinkedIn
- Read person profiles, company pages, and job details via [linkedin-scraper-mcp](https://github.com/stickerdaniel/linkedin-mcp-server)
- Search people and jobs via MCP, with Exa fallback
- Fallback to Jina Reader when MCP is not configured
- 通过 linkedin-scraper-mcp 读取个人 Profile、公司页面、职位详情
- 通过 MCP 搜索人才和职位，Exa 兜底
- 未配置 MCP 时自动 fallback 到 Jina Reader

#### 🏢 Boss直聘
- QR code login via [mcp-bosszp](https://github.com/mucsbr/mcp-bosszp)
- Job search and recruiter greeting via MCP
- Fallback to Jina Reader for reading job pages
- 通过 mcp-bosszp 扫码登录
- MCP 搜索职位、向 HR 打招呼
- Jina Reader 兜底读取职位页面

### 📈 Improvements / 改进

- Channel count: 9 → 12
- `agent-reach doctor` now detects all 12 channels
- CLI: added `search-linkedin`, `search-bosszhipin` subcommands
- Updated install guide with setup instructions for new channels
- 渠道数量：9 → 11
- `agent-reach doctor` 现在检测全部 11 个渠道
- CLI：新增 `search-linkedin`、`search-bosszhipin` 子命令
- 安装指南新增渠道配置说明

---

## [1.0.0] - 2025-02-24

### 🎉 Initial Release / 首次发布

- 9 channels: Web, Twitter/X, YouTube, Bilibili, GitHub, Reddit, XiaoHongShu, RSS, Exa Search
- CLI with `read`, `search`, `doctor`, `install` commands
- Unified channel interface — each platform is a single pluggable Python file
- Auto-detection of local vs server environments
- Built-in diagnostics via `agent-reach doctor`
- Skill registration for Claude Code / OpenClaw / Cursor
- 9 个渠道：网页、Twitter/X、YouTube、B站、GitHub、Reddit、小红书、RSS、Exa 搜索
- CLI 支持 `read`、`search`、`doctor`、`install` 命令
- 统一渠道接口 — 每个平台一个独立可插拔的 Python 文件
- 自动检测本地/服务器环境
- 内置诊断 `agent-reach doctor`
- Skill 注册支持 Claude Code / OpenClaw / Cursor
