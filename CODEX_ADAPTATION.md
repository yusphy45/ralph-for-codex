# Codex Adaptation Notes

本文记录 Ralph 当前项目机制，以及将它扩展为支持 Codex 的改造方案。

## 项目是做什么的

Ralph 是一个自治 AI coding agent 外层循环。它不会一次性把完整 PRD 交给同一个长上下文 agent，而是把需求拆成 `prd.json` 里的多个 user story，然后反复启动全新的 AI coding agent 实例。

每一轮 agent 只做一件事：

1. 读取 `prd.json` 和 `progress.txt`
2. 切换或创建 PRD 指定的 feature branch
3. 找到最高优先级且 `passes: false` 的 user story
4. 实现这个单独 story
5. 运行项目质量检查
6. 更新可复用经验到 `AGENTS.md` 或对应工具的说明文件
7. 提交代码
8. 把完成的 story 标记为 `passes: true`
9. 向 `progress.txt` 追加本轮进展和学习
10. 如果所有 story 都完成，输出 `<promise>COMPLETE</promise>`

Ralph 的跨轮记忆主要来自三处：

- Git history：上一轮提交的代码和提交信息
- `progress.txt`：上一轮追加的进展、踩坑和代码库模式
- `prd.json`：每个 story 的完成状态

## 当前工具适配方式

当前仓库主要支持 Amp 和 Claude Code：

- `ralph.sh` 是外层循环脚本
- `prompt.md` 是 Amp 使用的 agent 指令
- `CLAUDE.md` 是 Claude Code 使用的 agent 指令
- `prd.json.example` 定义 Ralph 所需的 PRD JSON 格式
- `skills/prd/` 用于生成 PRD
- `skills/ralph/` 用于把 PRD 转换成 Ralph JSON

其中 `prd.json` 格式本身和具体 AI coding 工具无关。真正和工具耦合的部分是：

- `ralph.sh` 如何启动对应 CLI
- 每种工具读取哪份提示词
- 每种工具使用哪种长期说明文件，例如 `AGENTS.md` 或 `CLAUDE.md`
- 是否需要特殊权限参数，例如 Amp 的 `--dangerously-allow-all`、Claude 的 `--dangerously-skip-permissions`

## Codex 适配思路

Codex CLI 支持非交互执行：

```bash
codex exec [OPTIONS] [PROMPT]
```

适合 Ralph 的默认调用方式是：

```bash
codex exec --full-auto -C <project-root> -
```

其中：

- `exec` 让 Codex 非交互执行一次任务
- `--full-auto` 使用低摩擦的自动执行模式，默认仍保留工作区沙箱
- `-C <project-root>` 明确 Codex 的工作目录
- `-` 表示从 stdin 读取 Ralph 提示词

Codex 也支持更激进的：

```bash
codex exec --dangerously-bypass-approvals-and-sandbox -C <project-root> -
```

但这会跳过审批和沙箱，风险更高，因此不建议作为默认选项。

## 需要改造的文件

### `ralph.sh`

需要把工具选项从：

```text
amp | claude
```

扩展为：

```text
amp | claude | codex
```

新增 Codex 分支后，每一轮可通过类似方式启动：

```bash
codex exec --full-auto -C "$PROJECT_ROOT" -
```

同时给 Codex 的 stdin 注入 Ralph 文件路径，避免提示词只能说“同目录下的 `prd.json`”而产生歧义。

### `CODEX.md`

新增 Codex 专用提示词模板，基于 `prompt.md` 调整：

- 使用 `AGENTS.md` 作为可复用项目知识的落点
- 不写 Amp thread URL
- 明确 `prd.json`、`progress.txt`、project root 会由 `ralph.sh` 注入
- 保留单 story、质量检查、commit、更新 PRD、追加 progress、完成时输出 `<promise>COMPLETE</promise>` 等 Ralph 核心约定

### `README.md`

需要把文档从“双工具支持”更新为“三工具支持”：

```bash
./ralph.sh --tool amp
./ralph.sh --tool claude
./ralph.sh --tool codex
```

并在 prerequisites、setup、key files、workflow、references 中加入 Codex。

### `AGENTS.md`

Codex 会读取 `AGENTS.md`，因此项目自身的 agent 说明也应写明 Ralph 现在支持 Codex，并说明 `CODEX.md` 的作用。

### `skills/ralph/SKILL.md`

该 skill 中原本写着 “fresh Amp instance”，应改成更通用的 “fresh AI coding agent instance”，避免 PRD 转换说明和 Codex 支持相冲突。

## 推荐行为

默认运行：

```bash
./ralph.sh --tool codex 10
```

Codex 分支默认使用：

```bash
codex exec --full-auto
```

如果用户明确知道风险，并且外部环境已经有隔离机制，再考虑手动把脚本改成使用：

```bash
codex exec --dangerously-bypass-approvals-and-sandbox
```

## 关键注意事项

`ralph.sh` 可能被复制到目标项目的 `scripts/ralph/` 目录。此时：

- Ralph 文件目录是 `scripts/ralph/`
- 真正代码库根目录通常是 Git root
- `prd.json` 和 `progress.txt` 仍在 Ralph 文件目录
- Codex 的 `-C` 应该指向项目根目录，而不是 Ralph 文件目录

因此 Codex 适配时应显式区分：

```bash
SCRIPT_DIR=<ralph.sh 所在目录>
PROJECT_ROOT=<当前 Git 仓库根目录，失败时回退到当前目录>
PRD_FILE="$SCRIPT_DIR/prd.json"
PROGRESS_FILE="$SCRIPT_DIR/progress.txt"
```

这样无论 Ralph 在仓库根目录运行，还是从 `scripts/ralph/` 被调用，都能更稳定地工作。
