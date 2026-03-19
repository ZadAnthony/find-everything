# find-everything

**[English](#english) | [中文](#中文)**

---

<a name="english"></a>

## English

> Cross-platform resource search orchestrator for AI Agents. Search 15+ sources in one query — find Skills, MCP Servers, Prompt templates, and open-source projects.

### Features

- **15 search sources** — Skills.sh, GitHub, Prompts.chat (MCP), ClawHub, SkillHub, MCPServers.org, and more
- **3-tier search strategy** — Tier 1 (CLI/MCP) → Tier 2 (WebSearch) → Tier 3 (broad search), converges fast
- **Pre-install security scanning** — deterministic code scanning + LLM context evaluation, quantified score 0-100
- **Natural language trigger** — just say "find me a xxx tool", no commands to memorize
- **Extensible registry** — discover and add new sources on the fly

### How It Works

```
User query
  ↓
Step 1:  Intent classification → skill / mcp / prompt / repo
Step 2:  Read registry.json, filter available sources
Step 3:  Tier 1 parallel search (CLI + MCP simultaneously)
Step 4:  Enough results? (≥3 high-relevance → skip to 6)
Step 5:  Tier 2/3 supplementary search
Step 6:  Relevance evaluation
Step 7:  Safety quick-check ([SAFE] / [CAUTION] / [RISK])
Step 8:  Dedup + sort (relevance > safety > popularity)
Step 9:  Display recommendations
Step 10: User action → "install" triggers deep security scan
Step 11: New site discovery → prompt to add to registry
```

### Search Sources (15 sources, 3 tiers)

#### Tier 1 — Direct API/CLI (fastest, most accurate)

| Source | Method | Categories |
|---|---|---|
| Skills.sh | `npx skills find` | Skill |
| GitHub | `gh search repos` | Repo / MCP / Skill |
| ClawHub | `clawhub search` | Skill |
| Prompts.chat | MCP `search_prompts` | Prompt |
| Prompts.chat | MCP `search_skills` | Skill |

#### Tier 2 — WebSearch with site: filter

SkillHub, MCPServers.org, SkillsMP, Prompts.chat, ClawHub, AI Short, NanoPrompts, AI Art Pics, LocalBanana

#### Tier 3 — Broad web search (fallback)

### Installation

```bash
# Install as a Claude Code skill
npx skills add ZadAnthony/find-everything --skill find-everything

# Or manually: symlink skill/ directory to your agent's skill path
ln -s /path/to/find-everything/skill ~/.claude/skills/find-everything
```

### Usage

```
/find-everything find a code review skill
/find-everything search aggregator MCP
/find-everything blockchain analyst prompt
```

Or just use natural language — "is there a tool for xxx", "find me a xxx".

### Optional: Install Prompts.chat MCP

Unlocks Tier 1 prompt search:

```bash
claude mcp add prompts-chat -- npx -y @fkadev/prompts.chat-mcp
```

### Security Scanning

Two layers of protection before installing any resource:

**Layer 1 — Quick check (metadata-based)**
- [SAFE]: from known platform + installs > 100
- [CAUTION]: low installs or unknown source
- [RISK]: suspected typosquat or red flags

**Layer 2 — Deep scan (triggered on install)**

`security_scan.py` detects:
- Prompt injection (15 patterns, including base64/URL-encoded/HTML-hidden)
- Dangerous commands (curl|sh, rm -rf, sudo, eval, etc.)
- Data exfiltration (curl POST, netcat, etc.)
- Credential access (.ssh, .aws, .env, etc.)
- Unicode homoglyph attacks
- Zero-width character injection
- Typosquat detection (edit distance + homoglyphs, against 200+ known skills)

Combined with `security-checklist.md` for LLM-assisted 5-dimension scoring (0-100):
source trust / code transparency / permission scope / network risk / community signals

### Project Structure

```
find-everything/
├── skill/                          # Skill package (what gets installed)
│   ├── SKILL.md                    # Core logic (11-step search orchestration)
│   ├── references/
│   │   ├── registry.json           # Search source registry
│   │   ├── security-checklist.md   # Security scoring rubric
│   │   └── known_skills.txt        # Known skill names (for typosquat detection)
│   └── scripts/
│       └── security_scan.py        # Deterministic security scanner
├── tests/                          # Tests for security_scan.py
├── docs/                           # Design documents
├── LICENSE
└── README.md
```

### Running Tests

```bash
python -m pytest tests/ -v
```

---

<a name="中文"></a>

## 中文

> 跨平台资源搜索编排器 for AI Agent。一句话搜遍 15 个源，找到你需要的 Skill、MCP Server、Prompt 模板或开源项目。

### 功能

- **15 个搜索源** — Skills.sh、GitHub、Prompts.chat (MCP)、ClawHub、SkillHub、MCPServers.org 等
- **3 层搜索策略** — Tier 1 (CLI/MCP) → Tier 2 (WebSearch) → Tier 3 (广域搜索)，快速收敛
- **安装前安全扫描** — 确定性代码扫描 + LLM 上下文评估，量化评分 0-100
- **自然语言触发** — 说"找个 xxx 工具"自动搜索，无需记命令
- **注册表可扩展** — 发现新站点一键加入

### 工作原理

```
用户查询
  ↓
Step 1:  意图分类 → skill / mcp / prompt / repo
Step 2:  读取 registry.json，筛选可用源
Step 3:  Tier 1 并行搜索（CLI + MCP 同时跑）
Step 4:  结果够了吗？（≥3 条高匹配 → 跳到 6）
Step 5:  Tier 2/3 补充搜索
Step 6:  相关性评估
Step 7:  安全快筛（[SAFE] / [CAUTION] / [RISK]）
Step 8:  去重 + 排序（相关度 > 安全 > 热度）
Step 9:  格式化展示推荐
Step 10: 用户操作 → "安装"触发深度安全扫描
Step 11: 新站点发现 → 提示加入注册表
```

### 搜索源（15 个，3 层）

#### Tier 1 — 直接 API/CLI（最快最准）

| 源 | 方法 | 覆盖类别 |
|---|---|---|
| Skills.sh | `npx skills find` | Skill |
| GitHub | `gh search repos` | Repo / MCP / Skill |
| ClawHub | `clawhub search` | Skill |
| Prompts.chat | MCP `search_prompts` | Prompt |
| Prompts.chat | MCP `search_skills` | Skill |

#### Tier 2 — WebSearch + site: 限定

SkillHub、MCPServers.org、SkillsMP、Prompts.chat、ClawHub、AI Short、NanoPrompts、AI Art Pics、LocalBanana

#### Tier 3 — 广域搜索（兜底）

### 安装

```bash
# 作为 Claude Code skill 安装
npx skills add ZadAnthony/find-everything --skill find-everything

# 或手动：将 skill/ 目录链接到你的 agent skill 路径
ln -s /path/to/find-everything/skill ~/.claude/skills/find-everything
```

### 使用

```
/find-everything 找一个代码审查的 skill
/find-everything search aggregator MCP
/find-everything 区块链分析师 prompt
```

也可以自然语言触发："有没有 xxx 工具"、"帮我找个 xxx"。

### 可选：安装 Prompts.chat MCP

解锁 Tier 1 级别的 prompt 搜索：

```bash
claude mcp add prompts-chat -- npx -y @fkadev/prompts.chat-mcp
```

### 安全扫描

安装任何资源前有两层防护：

**第一层 — 快筛（基于元数据）**
- [SAFE]：来自已知平台 + 安装量 > 100
- [CAUTION]：安装量低或来源不明
- [RISK]：疑似 typosquat 或其他红旗

**第二层 — 深度扫描（安装时触发）**

`security_scan.py` 检测：
- Prompt 注入（15 种模式，含 base64/URL 编码/HTML 注释隐藏）
- 危险命令（curl|sh、rm -rf、sudo、eval 等）
- 数据外传（curl POST、netcat 等）
- 凭证访问（.ssh、.aws、.env 等）
- Unicode 同形字攻击
- 零宽字符注入
- Typosquat 检测（编辑距离 + 同形字，对比 200+ 已知 skill）

配合 `security-checklist.md` 做 LLM 辅助 5 维度评分（0-100）：
来源可信度 / 代码透明度 / 权限范围 / 网络风险 / 社区信号

### 项目结构

```
find-everything/
├── skill/                          # Skill 本体（安装到 agent 的内容）
│   ├── SKILL.md                    # 核心逻辑（11 步搜索编排流程）
│   ├── references/
│   │   ├── registry.json           # 搜索源注册表
│   │   ├── security-checklist.md   # 安全评估评分体系
│   │   └── known_skills.txt        # 已知 skill 名单（typosquat 检测用）
│   └── scripts/
│       └── security_scan.py        # 确定性安全扫描脚本
├── tests/                          # 测试
├── docs/                           # 设计文档
├── LICENSE
└── README.md
```

### 运行测试

```bash
python -m pytest tests/ -v
```

---

## License

MIT
