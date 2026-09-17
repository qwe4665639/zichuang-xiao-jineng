# 查询与取证手册

本文件里的代码片段都在 2026-09-17 的 DSH 环境（Windows / `pwsh`）实测通过，可直接粘贴。标了"未实测"的只作跨平台备用。

---

## 1. 计算时间窗口

```powershell
function Get-IsoWeek([datetime]$d) {
  $thu  = $d.Date.AddDays(3 - (([int]$d.DayOfWeek + 6) % 7))   # 本周周四
  $week = [int][math]::Floor(($thu.DayOfYear - 1) / 7) + 1
  "{0}-W{1:D2}" -f $thu.Year, $week
}

$now    = Get-Date
$monday = $now.Date.AddDays(-(([int]$now.DayOfWeek + 6) % 7))   # 本周一
$month1 = (Get-Date -Day 1).Date

"今天的快照时刻 T_snap : {0:yyyy-MM-dd HH:mm}" -f $now
"本周窗口起点         : {0:yyyy-MM-dd}" -f $monday
"本月窗口起点         : {0:yyyy-MM-dd}" -f $month1
"周榜文件周号         : {0}" -f (Get-IsoWeek $now)
"月榜文件月份         : {0:yyyy-MM}" -f $now
```

自检样例（实测输出）：`2026-09-17 → 2026-W38`、`2027-01-01 → 2026-W53`、`2025-12-29 → 2026-W01`。
**不要用 `Get-Date -Format ww`**：那是日历周，不是 ISO 周，跨年时会错一周。
Linux/macOS（未实测）：`date -d "last monday" +%F`、`date -d "$(date +%Y-%m-01)" +%F`、`date +%G-W%V`。

---

## 2. 抓 trending 的真实窗口涨星

`https://github.com/trending?since=weekly` 与 `?since=monthly` 的每个条目里带 "N stars this week/month" —— 这是**唯一免费拿到的真实窗口涨星**。页面约 700 KB，**用 shell 抓取并本地解析，结果只 print 几行**，绝不要 `web_fetch` 整页。

```powershell
$since = 'weekly'   # 或 'monthly'
$h = @{ 'User-Agent' = 'Mozilla/5.0' }
$t = (Invoke-WebRequest -UseBasicParsing -TimeoutSec 25 `
      "https://github.com/trending?since=$since" -Headers $h).Content

$t -split '<article class="Box-row">' | Select-Object -Skip 1 | ForEach-Object {
  $repo = [regex]::Matches($_, 'href="/([A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+)"') |
            ForEach-Object { $_.Groups[1].Value } |
            Where-Object { $_ -notmatch '^(sponsors|topics|login|collections|orgs)/' } |
            Select-Object -First 1
  $gain = ([regex]::Match($_, '([\d,]+) stars this (week|month)')).Value `
            -replace ' stars this (week|month)', ''
  if ($repo) { "{0}`t{1}" -f $repo, $gain }
}
```

实测输出（本周，22 条）：`ayghri/i-have-adhd → 13,737`、`bilawalsidhu/gods-eye-view → 14,777`、`openai/plugins → 771` …

要点：

- **必须过滤 `sponsors/<user>`**：有赞助按钮的仓库会在仓库链接前先出现一条 `href="/sponsors/xxx"`，不过滤就会把用户当成仓库。
- 加语言过滤：`https://github.com/trending/python?since=weekly`；**trending 不支持话题过滤**，话题维度只能靠第 3/4 节。
- 榜单里只有 ~22–25 条，且是 GitHub 自己的口径，必须用搜索另开两路候选，别把它当全集。

Linux/macOS（未实测）：

```bash
curl -sL -A "Mozilla/5.0" "https://github.com/trending?since=weekly" \
| tr '\n' ' ' | sed 's|<article class="Box-row">|\n|g' \
| while IFS= read -r row; do
    repo=$(printf '%s' "$row" | grep -oE 'href="/[A-Za-z0-9_.-]+/[A-Za-z0-9_.-]+"' | grep -v '"/sponsors/' | head -1 | sed 's|href="/||;s|"||')
    gain=$(printf '%s' "$row" | grep -oE '[0-9,]+ stars this (week|month)' | head -1)
    [ -n "$repo" ] && printf '%s\t%s\n' "$repo" "$gain"
  done
```

---

## 3. GitHub Search API（无 MCP 时的替代）

**限速实测**：未认证搜索 `X-RateLimit-Limit: 10`，即约 10 次/分钟。一次 `per_page=30` 拿够，别循环多次；整轮梳理的搜索调用控制在 ~6 次以内。

```powershell
$h = @{ 'User-Agent' = 'Mozilla/5.0' }
$q = 'mcp server created:>=2026-09-10'
$u = "https://api.github.com/search/repositories?q=" + [uri]::EscapeDataString($q) + "&sort=stars&order=desc&per_page=30"
$r = Invoke-WebRequest -UseBasicParsing -TimeoutSec 30 $u -Headers $h
($r.Content | ConvertFrom-Json).items | ForEach-Object {
  "{0}`t{1}`t{2}`t{3}" -f $_.full_name, $_.stargazers_count, $_.created_at.Substring(0,10), $_.pushed_at.Substring(0,10)
}
```

**三路查询模板**（把 `<W>` 换成窗口起点）：

| 目的 | `q` |
| --- | --- |
| 新星（新仓库爆红） | `<关键词> created:>=<W> stars:>=200` |
| 活跃老库（trending 漏掉的） | `<关键词> pushed:>=<W> stars:>=500` |
| 话题限定 | `topic:mcp-server created:>=<W>` / `topic:agent-skills pushed:>=<W>` |
| 补齐元数据 | `repo:<owner>/<name>` |

`sort=stars` 排的是**累计星标**，不是窗口涨星。窗口涨星只有 trending（第 2 节）能给；拿不到就明说"以快照星标为准"。

---

## 4. 关键词 / 话题矩阵

每轮至少各取一条，覆盖四类插件 + 一类基建：

| 类别 | 关键词 / 话题 |
| --- | --- |
| MCP 服务 | `topic:mcp-server`、`topic:mcp`、`topic:model-context-protocol`、`"mcp server"`、`mcp gateway`、`remote mcp` |
| Agent Skills | `topic:agent-skills`、`topic:claude-skills`、`topic:skills`、`"agent skills"`、`skills marketplace` |
| Claude Code 插件 | `topic:claude-code`、`topic:claude-code-plugin`、`topic:claude-code-marketplace`、`plugin marketplace` |
| 其他 harness 插件 | `dsh plugin`、`topic:deepseek-harness`、`openclaw`、`codex plugin`、`cursor rules plugin` |
| 生态基建 | `skill eval`、`plugin security scanner`、`mcp security`、`prompt injection scan`、`plugin registry` |

中文社区热度（微信/小红书/即刻）抓不到，报告里写进"未能覆盖"。

---

## 5. 逐条取证

### 5.1 工具可用性（2026-09-17 本环境实测，别信文档里的想当然）

| 想做的事 | 可用 | 不可用 |
| --- | --- | --- |
| 搜仓库 / 元数据 | `mcp__github__search_repositories`；`api.github.com/search/repositories`（10 次/分钟） | — |
| **读文件正文（README 等）** | **`api.github.com/repos/<o>/<r>/readme`**（稳定）、**`mcp__github__search_code`**（直接返回正文片段） | `mcp__github__get_file_contents`（只回 embedded resource，模型看不到正文）、`web_fetch`（域名被拦）、`raw.githubusercontent.com`（3/3 超时） |
| 目录 / `skills/` 清单 | `api.github.com/repos/<o>/<r>/contents/<path>` | `get_file_contents` 看目录（同上读不到） |
| 窗口活跃度 | `mcp__github__list_commits`（`since`）、`mcp__github__list_releases` | — |
| 链接校验 | shell 发 HEAD 请求（见第 6 节） | `web_fetch` |

`web_fetch` 对本环境的 GitHub 域一律失败，实测报错原文：
`Error: URL hostname "github.com" resolves to a non-public IP address`。

### 5.2 读 README：只取"能干什么、要什么条件、边界在哪"

```powershell
$h = @{ 'User-Agent' = 'Mozilla/5.0'; 'Accept' = 'application/vnd.github.raw' }
$raw = (Invoke-WebRequest -UseBasicParsing -TimeoutSec 30 `
        "https://api.github.com/repos/tt-a1i/archify/readme" -Headers $h).Content
$md = if ($raw -is [byte[]]) { [System.Text.Encoding]::UTF8.GetString($raw) } else { $raw }
"长度: $($md.Length)"
# 只看标题结构 + 能力/前置条件相关句
($md -split "`r?`n") | Where-Object { $_ -match '^#{1,4}\s' } | Select-Object -First 30
($md -split "`r?`n") | Where-Object { $_ -match '(?i)requires|needs|before you|prerequisite|sign in|login|api key|only works|limitation|not supported|does not|doesn.t' } | Select-Object -First 25
```

**注意 `Content` 是 `Byte[]` 不是字符串**（带 `Accept: application/vnd.github.raw` 时实测如此），必须先 `[System.Text.Encoding]::UTF8.GetString(...)` 再切行，否则 `-match` 永远匹配不到（实测踩过）。

**要提取的只有四样**：① 它到底能干什么；② 给谁用；③ **要用它得先准备什么**（登录？联网？先有素材？先打开某个软件？）；④ 能力边界。**不要提取命令和配置** —— 报告里不写这些。

### 5.3 用代码搜索确认能力与前置条件

命令不进报告，但**代码搜索仍然是确认"它到底靠什么干活、要不要额外东西"的最快手段**：

```jsonc
// mcp__github__search_code
{ "query": "repo:volcengine/OpenViking MCP",     "perPage": 4, "fields": ["path", "text_matches"] }
{ "query": "repo:ayghri/i-have-adhd answer",     "perPage": 3, "fields": ["path", "text_matches"] }
```

实测能直接拿到有用的事实，例如：OpenViking 的文档里有 `"mcpServers": { "ov-mcp-server": { "url": "{{OPENVIKING_BASE_URL}}/mcp" } }`（说明它要靠一个跑起来的服务），并且仓库自带 `examples/dsh-memory-plugin/`、`examples/agent-hook-plugin/` 两套现成接入示例；i-have-adhd 的 `skills/i-have-adhd/SKILL.md` 写明了五条设计事实、`evals/rubric.md` 给出了 35/25/20/10/10 的评分维度。

👉 这些是用来写"**怎么启用**"和"**它帮不上忙的地方**"的素材，**不是**抄进报告的命令。

### 5.4 看根目录判定形态（决定"怎么启用"怎么写）

```powershell
$h = @{ 'User-Agent' = 'Mozilla/5.0'; 'Accept' = 'application/vnd.github+json' }
(Invoke-WebRequest -UseBasicParsing -TimeoutSec 30 `
  "https://api.github.com/repos/addyosmani/agent-skills/contents/" -Headers $h).Content |
  ConvertFrom-Json | ForEach-Object { $_.type + ' ' + $_.name }
```

实测输出：`dir .agents, dir .claude-plugin, dir .claude, dir .codex-plugin, … dir skills`。
配合 `contents/skills` 可列出全部技能目录（该仓库实测 25 个，如 `skills/context-engineering`、`skills/security-and-hardening`）。

### 5.5 窗口活跃度

```jsonc
// mcp__github__list_commits
{ "owner": "addyosmani", "repo": "agent-skills", "since": "2026-09-14T00:00:00Z",
  "perPage": 3, "fields": ["sha", "commit"] }
```

实测：该仓库窗口内返回 `[]`（最近提交停在 09-12），而它本周仍在 trending 上拿 2,119 星 —— **"本周无提交但仍然爆红"是真实存在的组合**，报告里要如实写，不要硬凑"本期有发版"。

---

## 6. 链接可用性校验

`web_fetch` 在本环境对 GitHub 域不可用，用 shell 发 HEAD：

```powershell
$h = @{ 'User-Agent' = 'Mozilla/5.0' }
function Test-Link([string]$u) {
  try   { $r = Invoke-WebRequest -UseBasicParsing -Method Head -TimeoutSec 20 $u -Headers $h
          "{0}  {1}" -f $r.StatusCode, $u }
  catch { "{0}  {1}" -f $_.Exception.Response.StatusCode.value__, $u }
}
Test-Link "https://github.com/addyosmani/agent-skills"        # 实测 200
Test-Link "https://github.com/this-owner-does-not-exist-xyz/nope"  # 实测 404
```

对收录进榜单的每一条跑一次；非 200 的直接剔除或标"链接异常"。私有/需登录的入口要说明这一点，不能当成可直接安装。

---

## 7. 类型 → "怎么启用" 的对照

报告里**不写安装方式**，只给网址。读者唯一要做的动作是"把链接丢给 AI 说帮我装上"。真正要动脑的是**装好之后他怎么开口**，按类型套：

| 判定出的类型 | "怎么启用"写成 | 实测素材从哪来 |
| --- | --- | --- |
| 一句话触发的技能（画图、改文风、写汇报） | 触发词是什么（"话里带上'画图''画流程'"）；没反应时怎么点名 | README 的功能段 + 仓库里的 `SKILL.md` |
| 装完自动生效的（输出风格、记忆、上下文管理） | "以后不用管它，它自己作用在你每次提问上" | README 的 "How it works" 段 |
| 要打开某个软件才有的（IDE 插件、桌面应用） | 在哪个场景下会遇到它、你要打开什么 | 仓库根目录是否有 `.cursor-plugin/` 等 |
| 市场 / 货架 | "它本身不是工具，是货架 —— 你从里面挑" | 是否有 `.claude-plugin/marketplace.json` |
| 要先喂素材的（记忆、知识库） | 先要给什么、给完之后怎么用 | README 的 Quick start 里"起服务/初始化"那几步的**含义**（不是命令本身） |

**判定类型**用第 5.4 节的根目录清单：有 `skills/` → 技能；有 `.claude-plugin/`、`.cursor-plugin/` → 插件或货架；有 `.mcp.json` / 文档里出现 `mcpServers` → 服务（多半要先跑起来）。DSH 用户的安装入口是技能根 `C:\Users\Administrator\.dsh\skills\<name>\SKILL.md`（即 `$DSH_HOME/skills`）或项目级 `<仓库根>/.dsh/skills/`，但**这些路径也不要写进报告**。

### 写"怎么启用"的三条语言规则

1. **说触发，不说机制**："你只要说'画一张流程图'，它自己就接手" ✅ ／ "通过 skill 自动匹配意图后调用" ❌
2. **说准备，不说配置**："你得先让它把资料读一遍" ✅ ／ "先执行 init 初始化数据目录" ❌
3. **说效果，不说产物格式**："给你一张能直接发出去的图" ✅ ／ "输出自包含 HTML + SVG" ❌

### 报告里一律不出现的词

`npx`、`uvx`、`docker`、`/plugin`、`pip install`、`mcpServers`、MCP、hook、CLI、manifest、CI、token、沙箱、渲染、渐进式披露、`SKILL.md`。

---

## 8. 收尾自检

- [ ] 前 10 条每条都有：**入口网址**、快照星标 + `T_snap`、一句大白话"它是什么"、**≥2 个不同处境的使用场景**、**"怎么开口启用"**、**"它帮不上忙的地方"**。
- [ ] **报告里搜不到任何命令**（`npx` / `/plugin` / `docker` / `pip` / `uvx` 一个都没有），也没有"安装方式"这一栏。
- [ ] 搜不到术语：MCP、hook、CLI、manifest、CI、token、沙箱、渲染、渐进式披露、`SKILL.md`。
- [ ] 场景**不是从 README 抄的** —— 每个场景都能回答"谁、什么处境、想干什么、以前多麻烦"。
- [ ] 两个场景确实是**两种不同处境**，不是同一件事换个说法。
- [ ] 每条 URL 都用 shell HEAD 校验过（200），异常链接已剔除或标注。
- [ ] 星标/URL/场景 全部能追溯到本次工具返回；没有"大概""据说"。
- [ ] 周榜和月榜各自独立前 10，名次没有互相借用。
- [ ] 失败的源写进了"未能覆盖"；生态基建类若被纳入也带了标签。
