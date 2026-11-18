# 审查器适配器与跨工具迁移指南

本 Skill 将 **「协作协议（`.collab/` + JSON Schema + 优先级闭环）」** 与 **「审查器 CLI 启动命令」** 解耦。迁移到其他 Agent（如 Workbuddy、Claude Code、Codex 互相调用）时，只需复制本 Skill 文件夹并按下方模板替换调用命令即可。

---

## 1. 默认适配器：OpenAI Codex CLI（已实测）

- **首轮全量审计（只读沙箱 + 结构化 JSON）**：
  ```bash
  codex exec -s read-only --skip-git-repo-check -C <WORKDIR> \
    --output-schema .collab/review_schema.json \
    -o .collab/codex_review.json \
    "请严格读取并执行 .collab/codex_prompt.txt 中的全部审计要求，输出中文 JSON。"
  ```
- **同会话复检（延续首轮上下文）**：
  ```bash
  codex exec -s read-only --skip-git-repo-check -C <WORKDIR> \
    --output-schema .collab/verify_schema.json \
    -o .collab/codex_verify_r<N>.json \
    resume <SESSION_ID> \
    "主控已完成 P0->P1->P2 修订，请严格对照 .collab/codex_review.json 的全部 acceptance_criteria 逐项判定 PASS/FAIL。"
  ```
- **注意事项**：
  - 若工作区不是 Git 仓库，必须加 `--skip-git-repo-check`。
  - 需要联网调用模型接口时，若本地沙箱拦截配置文件或网络，需请求用户授权在沙箱外执行。

---

## 2. 切换其他审查器 CLI（可插拔模板）

若希望把审查方从 Codex 切换为 **Claude Code** 或其他支持无头模式（Headless CLI）的 Agent：

1. **只读约束**：优先使用目标 CLI 的只读模式或只读权限参数，确保审查方**绝不直接修改源码/方案文件**。
2. **结构化输出约束**：若目标 CLI 支持 JSON Schema 参数，直接传入 `schemas/review_schema.json` 与 `schemas/verify_schema.json`；若不支持，在 Prompt 中强制要求输出严格符合该 Schema 的纯 JSON 并由主控用 Python `jsonschema` 校验。
3. **上下文延续**：优先使用目标 CLI 的会话恢复命令（如 `resume` / `--resume` / `--continue`）；若无会话恢复功能，则在复检 Prompt 中同时传入 `.collab/codex_review.json` 与修改后的目标文件。

---

## 3. 反向迁移到 Workbuddy / Codex / Claude

- **Workbuddy → 调 Codex / Claude**：将本 Skill 放入 Workbuddy 可读取的技能目录（或将其内容写入项目指令文件），由 Workbuddy 担任主控、按本 Skill 步骤调用 `codex exec`。
- **Codex → 调其他 CLI**：将本 Skill 放入 Codex 识别的技能/指令目录（如 `AGENTS.md` 引用），由 Codex 担任主控、通过终端调用外部无头审查 CLI 并读取 `.collab/` 产出的 JSON。
- **核心不变部分**：`.collab/` 共享目录、`P0 → P1 → P2` 修复顺序、最大 `3` 轮上限、以及无法解决时必须输出的 **「多选项 + 风险对照 + 需支持项」** 人工决策包。
