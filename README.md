# multiply-agent-work

> **双 Agent 协同审计与闭环修订 Skill（Dual-Agent Audit & Remediation Loop）**  
> 让「主控 Agent（负责编排与改文件）」与「只读审查 Agent（默认 OpenAI Codex CLI，负责挑刺、定验收标准与复检）」按固定协议自动完成 **全量审计 → 按优先级修复 → 逐项复检 → 无法解决时升级人工拍板** 的闭环。

---

## 为什么需要它？

单个 AI 自己写方案、自己检查，很容易“盲区自洽”；而两个 AI 如果没有约束地互相检查，又容易出现：
- 挤牙膏式挑刺，无限来回死循环；
- 审查方越权乱改原文件；
- 把“本机没装数据库/未部署”当成设计缺陷反复判 `FAIL`；
- 遇到业务口径分歧时卡死或擅自替人拍板。

`multiply-agent-work` 用 **4 条硬约束 + 结构化 JSON 协议** 解决上述全部问题。

---

## 核心机制与防死循环红线

1. **最大轮次上限（`max_rounds = 3`）**：默认最多跑 3 轮；超上限仍未收敛时强制退出循环并交由人工决策。
2. **审计边界隔离**：启动时明确区分「纯设计/静态逻辑审计」与「工程实跑审计」，避免把缺少运行环境判成设计失败。
3. **只读权限隔离**：审查 Agent 固定运行在只读沙箱（如 Codex `-s read-only`），**绝不直接修改任何项目文件**；只有主控 Agent 按 `P0 → P1 → P2` 顺序修订。
4. **一次性全量 + 自我复查**：首轮要求通读全文、一次性挖出所有问题，并在输出前做 Self-Critique 剔除伪问题。
5. **无法解决时的“人工决策包”升级**：遇到缺业务拍板、缺外部原件/环境、或达到 3 轮仍有分歧时，立即暂停循环，向用户输出：
   - **1. 不同的可选方案（Option A / B / C）**
   - **2. 每个建议方案及对应风险**
   - **3. 需要用户方提供的支持**
   并通过交互提问卡（或列表）由用户拍板。

---

## 目录结构

```text
multiply-agent-work/
├── README.md                        # 使用说明（本文件）
├── SKILL.md                         # Skill 核心执行指令与闭环状态机
├── schemas/
│   ├── review_schema.json           # 首轮全量审计输出 Schema（含 P0/P1/P2 与人工阻塞标记）
│   └── verify_schema.json           # 复检 PASS/FAIL 逐项核销 Schema
└── references/
    └── reviewer_adapters.md         # 审查器 CLI 适配模板与跨工具迁移指南
```

---

## 快速安装与使用

### 1. 安装（推荐软链接“一源多链”）

先将本仓库克隆到你的统一技能仓库目录（例如 `~/.agents/skills/multiply-agent-work`），再软链接到你使用的 AI 工具技能目录：

```bash
# 1) 克隆到本地统一目录
mkdir -p ~/.agents/skills
git clone https://github.com/baibaigithub/multiply-agent-work.git ~/.agents/skills/multiply-agent-work

# 2) 软链接到 Antigravity 全局技能目录（其他工具同理）
mkdir -p ~/.gemini/config/skills
ln -s ~/.agents/skills/multiply-agent-work ~/.gemini/config/skills/multiply-agent-work
```

### 2. 前置条件

- 默认审查器为 **OpenAI Codex CLI**（需本地已安装 `codex` 且已完成登录）。
- 若要切换为 Claude Code 或其他支持非交互式命令行的工具，请参考 [`references/reviewer_adapters.md`](references/reviewer_adapters.md)。

### 3. 对话触发示例

在任意项目对话中直接这样下达指令即可：

> 请使用 `multiply-agent-work` 技能，对 `docs/方案设计.html` 进行**纯设计级**只读审计与闭环修订：  
> 1. 让 Codex 一次性查完所有设计/逻辑/数据口径问题并给出可验证的验收标准；  
> 2. 你按 `P0 → P1 → P2` 顺序修改原文件；  
> 3. 改完后交回 Codex 复检，最多 3 轮；  
> 4. 遇到无法解决或需要业务拍板的问题时，按方案选项、风险、需提供的支持向我汇报。

---

## 运行产物说明

每次执行会在目标项目根目录下生成 `.collab/`（建议加入 `.gitignore` 或按需归档）：

- `.collab/codex_prompt.txt`：发给审查器的首轮指令
- `.collab/codex_review.json`：首轮全量审计问题清单与验收标准
- `.collab/codex_verify_r1.json` ~ `r3.json`：各轮逐项 `PASS / FAIL` 复检结果

---

## 跨工具迁移

本 Skill 将 **协作协议（`.collab/` + JSON Schema）** 与 **审查器 CLI 命令** 解耦：
- 可直接迁移至 **Workbuddy、Codex、Claude Code** 等任意支持读取 `SKILL.md` 与执行终端命令的 Agent；
- 迁移时只需复制本目录，并按 [`references/reviewer_adapters.md`](references/reviewer_adapters.md) 替换审查器启动命令即可。
