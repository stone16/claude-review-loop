# claude-review-loop

<p align="center">
  <a href="README.md">English</a> | <a href="README.zh-CN.md">中文</a>
</p>

[![License](https://img.shields.io/badge/License-Apache_2.0-blue.svg)](https://opensource.org/licenses/Apache-2.0)
[![Claude Code](https://img.shields.io/badge/Claude_Code-Plugin-blueviolet)](https://claude.ai/claude-code)
[![Version](https://img.shields.io/badge/version-1.0.0-green.svg)](https://github.com/stone16/claude-review-loop/releases)
[![Peer: Codex](https://img.shields.io/badge/peer-Codex_CLI-74aa9c)](https://github.com/openai/codex)
[![Peer: Gemini](https://img.shields.io/badge/peer-Gemini_CLI-4285F4)](https://github.com/google-gemini/gemini-cli)

**Claude Code 的跨 LLM 迭代式代码审查插件。**

自动调用另一个 AI（OpenAI Codex 或 Google Gemini）独立审查你的代码。Claude 评估审查结果，修复被采纳的问题，然后重新提交审查 —— 循环直到两个 AI 达成共识。

你不需要参与，只需要观看。

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

```
┌─────────────┐     ┌──────────────┐     ┌─────────────┐
│  Claude Code │────▶│  Peer (Codex │────▶│  Claude Code │
│  (编排器)     │    │  or Gemini)  │     │  (评估器)    │
│              │◀────│              │◀────│              │
│  1. 检测范围  │     │  2. 审查     │     │  3. 评估     │
│  4. 修复代码  │     │  5. 重审     │     │  6. 收敛     │
└─────────────┘     └──────────────┘     └─────────────┘
                         ↕ 重复直到达成共识
```

### 分阶段详解

#### 阶段 0+1：预检与上下文收集

单次 `preflight.sh` 执行完成所有准备工作，无需多次 API 调用：

1. 读取 `.review-loop/config.json`（项目级覆盖配置）
2. 自动检测审查范围：
   - **本地 diff** — 未暂存/已暂存的修改
   - **分支提交** — 领先于基础分支的提交
   - **Pull Request** — 通过 `gh` CLI 检测
   - **指定 commit** — 通过 SHA
3. 收集目标文件列表和项目上下文（CLAUDE.md、package.json、README）
4. 创建带有结构化 JSON 日志的会话目录
5. 创建 git checkpoint commit 用于安全回滚

#### 阶段 2：代码演进循环

每一轮：

1. **Claude 评估**每个 peer 发现 → ACCEPT 或 REJECT（附理由）
2. **实现修复** — 对已采纳的发现做最小化修改
3. **Checkpoint commit** — 记录变更
4. **发送重审 prompt** 给 peer：
   - 变更文件列表（peer 直接读取本地文件）
   - 被拒发现及 Claude 的理由
   - 已采纳/已修复项的摘要
5. **Peer 回应**：CONSENSUS / ACCEPTED_REJECTION / INSIST / 新发现
6. **辩论解决**：同一发现辩论超过 2 轮 → ESCALATED 升级给人工决策

#### 阶段 3：最终共识

- 全新 peer 会话（不复用之前的）进行独立最终检查
- 如果发现新问题，回到阶段 2 继续
- 生成 `summary.md` 和完整的 `rounds.json`

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

---

## 使用场景

### 场景 1：本地修改审查

你做了一些修改但还没提交。

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

你想让 Gemini 来审查一个特定的 PR。

```
你: "review loop with gemini for PR 42"

→ 范围：PR #42（通过 gh CLI）
→ 第 1 轮：Gemini 发现 5 个问题
→ Claude 采纳 4 个，拒绝 1 个（纯风格偏好，无项目规范要求）
→ 第 2 轮：Gemini 坚持被拒的发现，提出更强论据
→ Claude 重新评估，采纳 — 可读性方面的合理观点
→ 第 3 轮：Gemini 确认所有修复
→ 状态：✅ 3 轮后达成共识
```

### 场景 3：限制轮次的分支审查

你在一个 feature 分支上工作了一段时间。

```
你: "review loop, max 3 rounds"

→ 范围：领先 main 分支 12 个提交
→ 第 1 轮：Codex 在 6 个文件中发现 8 个问题
→ 第 2 轮：5 个已解决，3 个仍在辩论
→ 第 3 轮：又解决 2 个，1 个僵持（架构分歧）
→ 状态：⚠️ max_rounds — 1 个升级项需要人工决策

升级项："数据库连接池大小为 10 太低"
  Codex 认为：应该调到 50 以应对预期负载
  Claude 认为：当前值匹配现有基础设施限制
  → 需要人工/团队决策
```

### 场景 4：指定 Commit 审查

```
你: "review loop for commit abc1234"

→ 范围：单个 commit abc1234
→ 仅审查该 commit 涉及的文件
→ 第 1 轮：未发现问题
→ 状态：✅ 1 轮后达成共识
```

---

## 安装

### 通过 Claude Code Plugin 安装（推荐）

```bash
# 添加 marketplace
claude plugin marketplace add stone16/claude-review-loop

# 安装插件
claude plugin install stometa@claude-review-loop --scope user
```

### 通过 Git Clone 安装（手动）

```bash
git clone https://github.com/stone16/claude-review-loop.git
# 添加为本地 marketplace
claude plugin marketplace add /path/to/claude-review-loop
claude plugin install stometa@claude-review-loop --scope user
```

### 验证安装

```bash
claude plugin list | grep stometa
```

---

## 前置条件

| 依赖 | 安装方式 |
|------|---------|
| **Claude Code** | [claude.ai/claude-code](https://claude.ai/claude-code) |
| **Git** | 大多数系统预装 |
| **Codex CLI**（peer 选项 1） | `npm install -g @openai/codex` |
| **Gemini CLI**（peer 选项 2） | [参见 Gemini CLI 文档](https://github.com/google-gemini/gemini-cli) |
| **gh CLI**（可选，用于 PR 范围检测） | `brew install gh` 或 [cli.github.com](https://cli.github.com) |

---

## 配置

### 默认值

| 设置项 | 默认值 | 可选值 |
|-------|-------|-------|
| `peer_reviewer` | `codex` | `codex`, `gemini` |
| `max_rounds` | `5` | 1–10 |
| `timeout_per_round` | `600` | 秒 |
| `scope_preference` | `auto` | `auto`, `diff`, `branch`, `pr` |

### 项目级配置

在项目根目录创建 `.review-loop/config.json`：

```json
{
  "peer_reviewer": "gemini",
  "max_rounds": 8,
  "timeout_per_round": 300
}
```

### 调用时覆盖

```
"review loop with gemini, max 3 rounds"
```

**优先级**：内置默认值 < `.review-loop/config.json` < 调用时参数

---

## 输出

每次会话在 `.review-loop/` 下创建一个目录：

```
.review-loop/
├── 2026-03-11-143025-local-diff/
│   ├── rounds.json         # 结构化的轮次日志
│   ├── summary.md          # 人类可读的摘要
│   └── peer-output/        # 原始 peer 响应
│       ├── round-1-prompt.md
│       ├── round-1-raw.txt
│       ├── round-2-prompt.md
│       ├── round-2-raw.txt
│       └── peer-session-id.txt
└── latest -> 2026-03-11-143025-local-diff
```

### summary.md 示例

```markdown
# Review Loop 摘要

**会话**: 2026-03-11-143025-local-diff
**Peer**: codex CLI
**范围**: local-diff (3 个文件变更, 42 处新增)
**轮次**: 2 | **状态**: ✅ 共识

## 变更内容

- `src/auth.ts` — 修复用户查询中的 SQL 注入（参数化查询）
- `src/api/handler.ts` — 添加 null 响应的错误处理

## 发现解决情况

| # | 发现 | 严重度 | 操作 | 结果 |
|---|------|--------|------|------|
| f1 | 用户查询 SQL 注入 | critical | 采纳 | 已修复 |
| f2 | 缺少 null 检查 | major | 采纳 | 已修复 |
| f3 | 变量命名风格 | suggestion | 拒绝 | peer 接受理由 |

## 共识

Claude Code 和 Codex 在 2 轮后一致认为代码状态良好。
```

---

## 与传统代码审查的区别

| 维度 | 传统审查 | Review Loop |
|------|---------|-------------|
| 审查者 | 单模型 | 跨 LLM（Codex/Gemini + Claude） |
| 输出 | 问题列表 | 改进后的代码 + 共识报告 |
| 误报处理 | 列出后忽略 | 通过辩论解决 |
| 人工投入 | 需要阅读 + 修复 | 观看 + 仅决策升级项 |
| 迭代方式 | 一次性 | 多轮收敛 |

---

## 故障排除

| 问题 | 解决方案 |
|------|---------|
| "codex CLI not found" | `npm install -g @openai/codex` |
| "gemini CLI not found" | 安装 Gemini CLI；或者系统会自动回退到可用的 peer |
| Peer 超时 | 在配置中增加 `timeout_per_round` |
| "No changes detected" | 确保有未提交的修改、未推送的提交或打开的 PR |
| 插件已安装但 skill 未找到 | 用 `claude plugin list` 验证；确保使用了 `--scope user` |

---

## 贡献

欢迎贡献！请提交 issue 或 PR。

- Bug 报告和功能请求 → [Issues](https://github.com/stone16/claude-review-loop/issues)
- 代码贡献 → Fork, branch, PR

---

## 许可证

Apache 2.0 — 详见 [LICENSE](LICENSE)
