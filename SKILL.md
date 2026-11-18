---
name: multiply-agent-work
description: Orchestrate a structured, bounded dual-agent audit loop (default: Primary Agent edits + Codex CLI read-only reviews) using .collab/ JSON schemas, P0->P1->P2 remediation, max 3 rounds, and interactive human escalation with option/risk/support tables when blocked.
---

# 双 Agent 协同审计与闭环修订（Dual-Agent Audit）

本 Skill 用于让**主控 Agent（负责修改与编排）**与**外部审查 Agent（默认 OpenAI Codex CLI，只读审查）**按固定协议协同完成「全量审计 → 按优先级修复 → 逐项复检 → 无法解决时升级人工拍板」的闭环。

---

## 一、强制前提与防死循环红线

执行本 Skill 前，必须先明确并锁定以下 4 项前提：

1. **最大轮次上限**：默认 **`max_rounds = 3`**（用户未另行指定时严禁超过 3 轮）。
2. **审计边界隔离（极重要）**：在首轮提示词中明确本次属于：
   - **纯设计/静态审计**：只审文档、架构、DDL、代码静态逻辑，**禁止**将“本机无数据库实例/后端未开发/未来年份官方数据未发布”等工程实跑项判为 `FAIL`；
   - **工程实跑审计**：需要验证真实建库、接口测试与性能压测。
3. **只读权限隔离**：外部审查 Agent 必须运行在只读模式（如 Codex 的 `-s read-only`），**绝不允许直接修改用户文件**；所有文件修改均由主控 Agent 按 `P0 → P1 → P2` 顺序执行。
4. **通过门槛**：默认要求全部 `P0 + P1 + P2` 均 `PASS`（若用户明确放行 `P2`，则以 `P0 + P1` 全 `PASS` 为准）。

---

## 二、标准执行流程

### Step 0：初始化 `.collab/` 协议目录
在当前工作区根目录下创建 `.collab/`，将本 Skill 目录下的：
- `schemas/review_schema.json` → 复制到 `.collab/review_schema.json`
- `schemas/verify_schema.json` → 复制到 `.collab/verify_schema.json`

### Step 1：一次性全量审计（第 1 轮）
1. 在 `.collab/codex_prompt.txt` 写入审计指令，强制要求审查 Agent：
   - **一次性通读目标文件全文**，严禁敷衍或留到下一轮再补；
   - **第一性原理 + 零废话**：每条问题必须给出原文/代码证据、违背的底层原理、具体可落地的修复建议、以及可客观判定 `PASS/FAIL` 的 `acceptance_criteria`；
   - **强制自我复查（Self-Critique）**：输出前自行剔除吹毛求疵的伪问题和不可落地建议，将过程写入 `self_review_notes`；
   - **显式标记无法靠改文件解决的项**：若某问题需要业务方拍板、外部官方原件或本地运行环境，必须置 `blocked_needs_human = true` 并在 `human_support_needed` 写明所需支持。
2. 调用审查器 CLI（默认 Codex，其他工具见 [reviewer_adapters.md](references/reviewer_adapters.md)）：
   ```bash
   codex exec -s read-only --skip-git-repo-check -C <WORKDIR> \
     --output-schema .collab/review_schema.json \
     -o .collab/codex_review.json \
     "请严格读取并执行 .collab/codex_prompt.txt 中的全部审计要求，输出中文 JSON。"
   ```
   - 记下返回日志中的 `session id`（用于后续 `resume` 复检）。

### Step 2：按 `P0 → P1 → P2` 实施修复与本地自检
1. 读取 `.collab/codex_review.json`：
   - 若存在 `blocked_needs_human == true` 的 `P0` 阻塞项且不拍板就无法正确修改，直接跳至 **Step 4（人工决策升级）**；
   - 否则，先处理全部可解决的 `P0`，再处理 `P1`，最后处理 `P2`。
2. 修改完成后，主控必须运行本地静态自检（如 DDL 语法/外键校验、HTML 锚点与样式检查、单测/Lint），确保不引入低级错误。

### Step 3：同会话全量复检
使用 `codex exec resume <SESSION_ID>` 让审查 Agent 在原上下文中严格对照 `.collab/codex_review.json` 的每条 `acceptance_criteria` 逐项判定 `PASS` 或 `FAIL`：
```bash
codex exec -s read-only --skip-git-repo-check -C <WORKDIR> \
  --output-schema .collab/verify_schema.json \
  -o .collab/codex_verify_r<N>.json \
  resume <SESSION_ID> \
  "主控已完成 P0->P1->P2 修订，请严格对照 .collab/codex_review.json 的全部 acceptance_criteria 逐项判定 PASS/FAIL，输出中文 JSON。"
```

### Step 4：收敛判定与“无法解决”人工决策升级
读取 `.collab/codex_verify_r<N>.json`：
- **情况 A（全部达标）**：`failed_count == 0`（或仅剩可豁免的 `P2`）→ 输出最终总结与产物链接，**结束流程**。
- **情况 B（未达标且当前轮次 `< max_rounds` 且无 `blocked_needs_human`）**：针对 `FAIL` 项补充修改，回到 **Step 3**（轮次 `+1`）。
- **情况 C（触发升级条件）**：只要满足以下任一条件，**立即停止自动循环**：
  1. 已达到最大轮次（`round >= max_rounds`）仍有 `FAIL`；
  2. 存在 `blocked_needs_human == true` 的阻塞项；
  3. 主控评估审查方的建议与核心业务约束冲突、无法在不破坏原需求的情况下修改。

  **升级动作（必须同时执行以下两步）**：
  1. 在对话中向用户展示**「问题决策与风险对照表」**，固定包含 3 项内容：
     - **1. 不同的可选方案**（Option A / B / C，写清各自改法与影响范围）；
     - **2. 每个建议方案及对应风险**（标出推荐项及二阶影响）；
     - **3. 需你方提供的支持**（如业务口径拍板、提供官方原件/环境权限、或确认豁免）。
  2. 调用 **`ask_question`** 工具弹出交互式多选确认卡（第一项标 `(Recommended)`），等用户点击确认后再继续执行或收尾。

---

## 三、审查器切换与跨工具迁移

- 需要把 Codex 换成 **Claude Code** 或其他 CLI 工具，或把本 Skill 迁移到 **Workbuddy / Codex / Claude** 中使用时，直接参考 [reviewer_adapters.md](references/reviewer_adapters.md) 替换调用命令即可，`.collab/` 协议与 JSON Schema 无需改动。
