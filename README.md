# claude-review-loop

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet)](https://claude.ai/claude-code)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](https://github.com/stone16/claude-review-loop/releases)
[![Peer: Codex](https://img.shields.io/badge/peer-Codex_CLI-74aa9c)](https://github.com/openai/codex)
[![Peer: Gemini](https://img.shields.io/badge/peer-Gemini_CLI-4285F4)](https://github.com/google-gemini/gemini-cli)

**Cross-LLM iterative code review for Claude Code.**

Spawns a peer AI reviewer (OpenAI Codex or Google Gemini) to independently review your code. Claude evaluates findings, fixes accepted issues, and re-submits for re-review — looping until both agents reach consensus.

You don't participate. You watch.

| Variant | Peer Reviewer | Best For |
|---------|--------------|----------|
| `review loop` | Codex (default) | General code review, strong at logic/security |
| `review loop with gemini` | Gemini CLI | Alternative perspective, good at patterns/style |
| `review loop, max 3 rounds` | Configurable | Quick reviews with time constraints |
| `review loop for PR 42` | Auto-detect | PR-scoped review via `gh` CLI |
| `review loop for commit abc123` | Auto-detect | Single commit review |

---

## Why Cross-LLM Review?

Single-model code review has blind spots. Every LLM has biases — patterns it over-indexes on and issues it consistently misses. By having a *different* model review your code and then *debating* the findings, you get:

- **Diversity of perspective** — Codex and Claude catch different classes of bugs
- **Adversarial validation** — findings are challenged, not just listed
- **Higher signal-to-noise** — false positives get filtered through debate
- **Autonomous improvement** — code actually gets fixed, not just flagged

---

## How It Works

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Claude Code │────▶│  Peer (Codex │────▶│  Claude Code │
│  (orchestrator)    │  or Gemini)  │     │  (evaluator) │
│              │◀────│              │◀────│              │
│  1. Detect scope   │  2. Review   │     │  3. Evaluate │
│  4. Fix code │     │  5. Re-review│     │  6. Converge │
└─────────────┘     └──────────────┘     └─────────────┘
                         ↕ repeat until consensus
```

### Phase-by-Phase Breakdown

#### Phase 0+1: Preflight & Context Collection

A single `preflight.sh` execution handles everything — no back-and-forth API calls:

1. Reads `.review-loop/config.json` (project-level overrides)
2. Detects review scope automatically:
   - **Local diff** — unstaged/staged changes
   - **Branch commits** — commits ahead of base branch
   - **Pull Request** — via `gh` CLI
   - **Specific commit** — by SHA
3. Collects target file list and project context (CLAUDE.md, package.json, README)
4. Creates session directory with structured JSON log
5. Creates a git checkpoint commit for safe rollback

#### Phase 2: Code Evolution Loop

For each round:

1. **Claude evaluates** each peer finding → ACCEPT or REJECT with reasoning
2. **Implements fixes** for accepted findings (minimal, scoped changes)
3. **Checkpoint commits** changes
4. **Sends re-review prompt** to peer with:
   - List of changed files (peer reads them locally)
   - Rejected findings with Claude's reasoning
   - Summary of accepted/fixed items
5. **Peer responds** with: CONSENSUS, ACCEPTED_REJECTION, INSIST, or new FINDING
6. **Debate resolution**: findings debated for 2+ rounds → ESCALATED for human decision

#### Phase 3: Final Consensus

- Fresh peer session (not resumed) performs independent final check
- If new findings emerge, loops back to Phase 2
- Generates `summary.md` and completes `rounds.json`

### How Context Is Passed to the Peer

The peer reviewer does **not** receive embedded diffs or pasted code. Instead:

```
┌──────────────────────────────────────────────────┐
│              Prompt to Peer Reviewer             │
├──────────────────────────────────────────────────┤
│ • repo_root         → absolute path              │
│ • scope_type        → "local-diff" / "pr-42"     │
│ • target_files      → file list to inspect       │
│ • project_context   → brief project description  │
│                                                  │
│ The peer reads local files directly from the     │
│ workspace. No diff is embedded in the prompt.    │
│ This keeps prompts small and consistent.         │
└──────────────────────────────────────────────────┘
```

For Codex specifically:
- Runs in an **isolated CODEX_HOME** (no MCP servers, stripped API keys)
- Gets **full local filesystem access** via `--dangerously-bypass-approvals-and-sandbox`
- Session is **reused across re-review rounds** for context continuity
- Final consensus uses a **fresh session** for independence

---

## Scenarios

### Scenario 1: Local Diff Review

You've made changes but haven't committed yet.

```
You: "review loop"

→ Detects 5 files with local changes
→ Round 1: Codex finds 3 issues (1 critical SQL injection, 2 minor)
→ Claude accepts all 3, implements fixes, commits
→ Round 2: Codex confirms fixes, no new issues
→ Final: Fresh session confirms consensus
→ Status: ✅ consensus after 2 rounds
```

### Scenario 2: PR Review with Gemini

You want Gemini to review a specific PR.

```
You: "review loop with gemini for PR 42"

→ Scope: PR #42 (via gh CLI)
→ Round 1: Gemini finds 5 issues
→ Claude accepts 4, rejects 1 (stylistic, no project convention)
→ Round 2: Gemini insists on rejected finding with stronger argument
→ Claude re-evaluates, accepts — valid point about readability
→ Round 3: Gemini confirms all fixes
→ Status: ✅ consensus after 3 rounds
```

### Scenario 3: Branch Review with Max Rounds

You've been working on a feature branch.

```
You: "review loop, max 3 rounds"

→ Scope: 12 commits ahead of main
→ Round 1: Codex finds 8 issues across 6 files
→ Round 2: 5 resolved, 3 still debated
→ Round 3: 2 more resolved, 1 deadlocked (architectural disagreement)
→ Status: ⚠️ max_rounds — 1 escalated finding for human decision

Escalated: "Database pool size of 10 is too low"
  Codex says: should be 50 for expected load
  Claude says: matches current infrastructure limits
  → Needs human/team decision
```

### Scenario 4: Specific Commit Review

```
You: "review loop for commit abc1234"

→ Scope: single commit abc1234
→ Reviews only files touched in that commit
→ Round 1: no issues found
→ Status: ✅ consensus after 1 round
```

---

## Installation

### Via Claude Code Plugin (Recommended)

```bash
# Add the marketplace
claude plugin marketplace add stone16/claude-review-loop

# Install the plugin
claude plugin install stometa@claude-review-loop --scope user
```

### Via Git Clone (Manual)

```bash
git clone https://github.com/stone16/claude-review-loop.git
# Then add as local marketplace
claude plugin marketplace add /path/to/claude-review-loop
claude plugin install stometa@claude-review-loop --scope user
```

### Verify Installation

```bash
claude plugin list | grep stometa
```

---

## Prerequisites

| Requirement | How to Install |
|-------------|----------------|
| **Claude Code** | [claude.ai/claude-code](https://claude.ai/claude-code) |
| **Git** | Pre-installed on most systems |
| **Codex CLI** (peer option 1) | `npm install -g @openai/codex` |
| **Gemini CLI** (peer option 2) | [See Gemini CLI docs](https://github.com/google-gemini/gemini-cli) |
| **gh CLI** (optional, for PR scope) | `brew install gh` or [cli.github.com](https://cli.github.com) |

---

## Configuration

### Defaults

| Setting | Default | Options |
|---------|---------|---------|
| `peer_reviewer` | `codex` | `codex`, `gemini` |
| `max_rounds` | `5` | 1–10 |
| `timeout_per_round` | `600` | seconds |
| `scope_preference` | `auto` | `auto`, `diff`, `branch`, `pr` |

### Project-Level Config

Create `.review-loop/config.json` in your project root:

```json
{
  "peer_reviewer": "gemini",
  "max_rounds": 8,
  "timeout_per_round": 300
}
```

### Invocation-Time Override

```
"review loop with gemini, max 3 rounds"
```

**Precedence:** built-in defaults < `.review-loop/config.json` < invocation args

---

## Output

Each session creates a directory under `.review-loop/`:

```
.review-loop/
├── 2026-03-11-143025-local-diff/
│   ├── rounds.json         # Structured log of all rounds
│   ├── summary.md          # Human-readable summary
│   └── peer-output/        # Raw peer responses
│       ├── round-1-prompt.md
│       ├── round-1-raw.txt
│       ├── round-2-prompt.md
│       ├── round-2-raw.txt
│       └── peer-session-id.txt
└── latest -> 2026-03-11-143025-local-diff
```

### summary.md Example

```markdown
# Review Loop Summary

**Session**: 2026-03-11-143025-local-diff
**Peer**: codex CLI
**Scope**: local-diff (3 files changed, 42 insertions)
**Rounds**: 2 | **Status**: ✅ consensus

## Changes Made

- `src/auth.ts` — Fixed SQL injection in user lookup (parameterized query)
- `src/api/handler.ts` — Added missing error handling for null response

## Findings Resolution

| # | Finding | Severity | Action | Resolution |
|---|---------|----------|--------|------------|
| f1 | SQL injection in user lookup | critical | accept | fixed |
| f2 | Missing null check | major | accept | fixed |
| f3 | Variable naming style | suggestion | reject | peer accepted reasoning |

## Consensus

Both Claude Code and Codex agree the code is in good shape after 2 rounds.
```

---

## How It Differs From Standard Code Review

| Aspect | Standard Review | Review Loop |
|--------|----------------|-------------|
| Reviewer | Single model | Cross-LLM (Codex/Gemini + Claude) |
| Output | List of findings | Improved code + consensus report |
| False positives | Listed and ignored | Debated and resolved |
| Human effort | Must read + fix | Watch + decide escalations only |
| Iteration | One-shot | Multi-round convergence |

---

## Troubleshooting

| Issue | Solution |
|-------|----------|
| "codex CLI not found" | `npm install -g @openai/codex` |
| "gemini CLI not found" | Install Gemini CLI; or the loop falls back automatically |
| Peer times out | Increase `timeout_per_round` in config |
| "No changes detected" | Ensure you have uncommitted changes, unpushed commits, or an open PR |
| Plugin installed but skill not found | Verify with `claude plugin list`; ensure `--scope user` was used |

---

## Contributing

Contributions welcome! Please open an issue or PR.

- Bug reports and feature requests → [Issues](https://github.com/stone16/claude-review-loop/issues)
- Code contributions → Fork, branch, PR

---

## License

Apache 2.0 — see [LICENSE](LICENSE)

---

---

# 中文说明

## claude-review-loop

**Claude Code 的跨 LLM 迭代式代码审查插件。**

自动调用另一个 AI（OpenAI Codex 或 Google Gemini）独立审查你的代码。Claude 评估审查结果，修复被采纳的问题，然后重新提交审查 —— 循环直到两个 AI 达成共识。

你不需要参与，只需要观看。

### 调用方式一览

| 调用方式 | Peer Reviewer | 适用场景 |
|---------|--------------|---------|
| `review loop` | Codex（默认） | 通用代码审查，擅长逻辑/安全 |
| `review loop with gemini` | Gemini CLI | 替代视角，擅长模式/风格 |
| `review loop, max 3 rounds` | 可配置 | 有时间限制的快速审查 |
| `review loop for PR 42` | 自动检测 | 通过 `gh` CLI 限定 PR 范围 |
| `review loop for commit abc123` | 自动检测 | 单个 commit 审查 |

---

## 为什么需要跨 LLM 审查？

单模型代码审查有盲点。每个 LLM 都有偏见 —— 它会过度关注某些模式，同时持续遗漏其他问题。通过让一个 *不同的* 模型来审查代码，然后 *辩论* 发现的问题，你可以获得：

- **多样化视角** — Codex 和 Claude 能捕获不同类别的 bug
- **对抗性验证** — 发现的问题会被质疑，而不只是列出来
- **更高信噪比** — 误报通过辩论被过滤掉
- **自主改进** — 代码真正被修复，而不只是被标记

---

## 工作原理

### 上下文如何传递给 Peer Reviewer

Peer reviewer **不会**收到嵌入的 diff 或粘贴的代码。取而代之的是：

```
┌──────────────────────────────────────────────────┐
│         传递给 Peer Reviewer 的 Prompt            │
├──────────────────────────────────────────────────┤
│ • repo_root         → 仓库绝对路径               │
│ • scope_type        → "local-diff" / "pr-42"     │
│ • target_files      → 需要检查的文件列表          │
│ • project_context   → 简短的项目描述              │
│                                                  │
│ Peer 从工作区直接读取本地文件。                    │
│ Prompt 中不嵌入 diff，保持轻量和一致。            │
└──────────────────────────────────────────────────┘
```

对于 Codex：
- 在**隔离的 CODEX_HOME** 中运行（无 MCP 服务器，剥离 API 密钥）
- 通过 `--dangerously-bypass-approvals-and-sandbox` 获得**完整本地文件系统访问**
- 在重审轮次中**复用同一会话**以保持上下文连续性
- 最终共识检查使用**全新会话**以确保独立性

### 三阶段流程

**阶段 0+1：预检与上下文收集**
- 单次 `preflight.sh` 执行，替代 15+ 次工具调用
- 自动检测审查范围（本地 diff → 分支提交 → PR → 指定 commit）
- 收集目标文件列表和项目上下文
- 创建 git checkpoint commit 用于安全回滚

**阶段 2：代码演进循环**
- Claude 评估每个发现 → ACCEPT 或 REJECT（附理由）
- 实现已采纳的修复，创建 checkpoint commit
- 发送重审 prompt 给 peer（包含变更文件列表、被拒理由）
- Peer 回应：CONSENSUS / ACCEPTED_REJECTION / INSIST / 新发现
- 辩论超过 2 轮 → ESCALATED 升级给人工决策

**阶段 3：最终共识**
- 全新 peer 会话进行独立最终检查
- 生成 `summary.md` 和 `rounds.json`

---

## 使用场景

### 场景 1：本地修改审查

```
你: "review loop"

→ 检测到 5 个文件有本地修改
→ 第 1 轮：Codex 发现 3 个问题（1 个严重 SQL 注入，2 个次要）
→ Claude 全部采纳，实现修复，提交
→ 第 2 轮：Codex 确认修复，无新问题
→ 最终：全新会话确认共识
→ 状态：✅ 2 轮后达成共识
```

### 场景 2：指定 Gemini 审查 PR

```
你: "review loop with gemini for PR 42"

→ 范围：PR #42
→ 第 1 轮：Gemini 发现 5 个问题
→ Claude 采纳 4 个，拒绝 1 个（纯风格偏好）
→ 第 2 轮：Gemini 坚持被拒的发现，提出更强论据
→ Claude 重新评估，采纳 — 可读性方面的合理观点
→ 第 3 轮：Gemini 确认所有修复
→ 状态：✅ 3 轮后达成共识
```

### 场景 3：限制轮次的分支审查

```
你: "review loop, max 3 rounds"

→ 范围：领先 main 分支 12 个提交
→ 第 1 轮：Codex 在 6 个文件中发现 8 个问题
→ 第 2 轮：5 个已解决，3 个仍在辩论
→ 第 3 轮：又解决 2 个，1 个僵持（架构分歧）
→ 状态：⚠️ max_rounds — 1 个升级项需要人工决策
```

---

## 安装

```bash
# 添加 marketplace
claude plugin marketplace add stone16/claude-review-loop

# 安装插件
claude plugin install stometa@claude-review-loop --scope user

# 验证
claude plugin list | grep stometa
```

## 前置条件

| 依赖 | 安装方式 |
|------|---------|
| **Claude Code** | [claude.ai/claude-code](https://claude.ai/claude-code) |
| **Git** | 大多数系统预装 |
| **Codex CLI**（peer 选项 1） | `npm install -g @openai/codex` |
| **Gemini CLI**（peer 选项 2） | 参见 Gemini CLI 文档 |
| **gh CLI**（可选，用于 PR 范围检测） | `brew install gh` |

## 配置

在项目根目录创建 `.review-loop/config.json`：

```json
{
  "peer_reviewer": "gemini",
  "max_rounds": 8,
  "timeout_per_round": 300
}
```

调用时覆盖：`"review loop with gemini, max 3 rounds"`

**优先级**：内置默认值 < 配置文件 < 调用时参数

---

## 许可证

Apache 2.0 — 详见 [LICENSE](LICENSE)
