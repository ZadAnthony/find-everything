# find-everything

跨平台资源搜索编排器，为 Claude Code 打造。一句话搜遍 15 个源，找到你需要的 Skill、MCP Server、Prompt 模板或开源项目。

## 功能

- **15 个搜索源**：Skills.sh、GitHub、Prompts.chat (MCP)、ClawHub、SkillHub、MCPServers.org 等
- **3 层搜索策略**：Tier 1 (CLI/MCP) → Tier 2 (WebSearch) → Tier 3 (广域搜索)，快速收敛
- **安装前安全扫描**：确定性代码扫描 + LLM 上下文评估，量化评分 0-100
- **自动触发**：说"找个 xxx 工具"自动搜索，无需记命令
- **注册表可扩展**：发现新站点一键加入

## 安装

```bash
# 作为 Claude Code skill 安装
npx skills add ZadAnthony/find-everything --skill find-everything

# 或手动：将 skill/ 目录链接到 ~/.claude/skills/find-everything
```

## 使用

```
/find-everything 找一个代码审查的 skill
/find-everything search aggregator MCP
/find-everything 区块链分析师 prompt
```

也可以自然语言触发："有没有 xxx 工具"、"帮我找个 xxx"。

## 可选：安装 Prompts.chat MCP

安装后可解锁 Tier 1 级别的 prompt 搜索能力：

```bash
claude mcp add prompts-chat -- npx -y @fkadev/prompts.chat-mcp
```

## 项目结构

```
find-everything/
├── skill/                          # Skill 本体（安装到 ~/.claude/skills/ 的内容）
│   ├── SKILL.md                    # 核心逻辑（11 步搜索编排流程）
│   ├── references/
│   │   ├── registry.json           # 搜索源注册表
│   │   ├── security-checklist.md   # 安全评估评分体系
│   │   └── known_skills.txt        # 已知 skill 名单（typosquat 检测用）
│   └── scripts/
│       └── security_scan.py        # 确定性安全扫描脚本
├── tests/                          # 测试
│   ├── test_security_scan.py
│   └── test_data/
├── docs/                           # 设计文档
└── README.md
```

## 安全扫描能力

`security_scan.py` 检测以下威胁：

- Prompt 注入（15 种模式，含 base64/URL 编码/HTML 注释隐藏）
- 危险命令（curl|sh、rm -rf、sudo、eval 等）
- 数据外传（curl POST、netcat 等）
- 凭证访问（.ssh、.aws、.env 等）
- Unicode 同形字攻击
- 零宽字符注入
- Typosquat 检测（编辑距离 + 同形字）

## 运行测试

```bash
python -m pytest tests/ -v
```

## License

MIT
