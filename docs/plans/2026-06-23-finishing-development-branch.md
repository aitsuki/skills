# 开发分支收尾实施计划

> **For agentic workers:** REQUIRED SKILL: Use $executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 新增可独立调用的 `finishing-a-development-branch` skill，并让 `executing-plans` 在实施完成后携带明确的提交边界交接给它。

**Architecture:** `executing-plans` 在首个实施任务前记录 `implementation-base`，完成验证后强制调用新的收尾 skill。收尾 skill 负责安全检查、用户批准、备份分支、单提交或多语义提交重建、等价性验证，以及后续分支处置；手动调用时必须先让用户确认 Agent 推断的提交边界。

**Tech Stack:** Markdown skill 定义、YAML agent 元数据、Git 命令、PowerShell/rg 文本验证。

## Global Constraints

- 设计文档和实施计划提交必须位于 `implementation-base` 之前，不得进入实施提交整理范围。
- 只有工作区和暂存区干净时才允许重写历史。
- 禁止在 `main`、`master` 等受保护主分支上重写历史。
- 无法刷新远端引用或无法证明候选提交尚未推送时，必须拒绝重写。
- 单提交和多语义提交两种整理模式都必须先创建并验证备份分支。
- 备份分支命名为 `backup/<current-branch>-before-finish-<YYYYMMDD-HHMMSS>`，任何路径都不得自动删除它。
- 多语义提交方案必须先展示提交标题、文件、摘要及必要的依赖顺序，并获得用户批准。
- 重写前后必须验证 Git tree、diff 等价，并再次运行实施相关测试。
- 创建备份后发生失败时必须停止，不得自动回滚、删除引用或更换策略重试。
- 恢复命令只能展示，不能自动执行。

---

### Task 1: 新增开发分支收尾 Skill

**Files:**
- Create: `skills/finishing-a-development-branch/SKILL.md`

**Interfaces:**
- Consumes: `implementation-base`（由 `executing-plans` 传入）或手动调用时经用户确认的候选基准。
- Produces: 三种提交处理路径、安全备份和验证结果，以及本地合并、创建 PR、保留分支三种后续处置选择。

- [ ] **Step 1: 写出结构检查并确认当前失败**

Run:

```powershell
$path = "skills/finishing-a-development-branch/SKILL.md"
$required = @(
  "name: finishing-a-development-branch",
  "implementation-base..HEAD",
  "git fetch",
  "backup/<current-branch>-before-finish-<YYYYMMDD-HHMMSS>",
  "Keep existing commits",
  "Consolidate into one commit",
  "Reorganize into semantic commits",
  "git reset --hard <backup-branch>"
)
$text = if (Test-Path $path) { Get-Content -Raw $path } else { "" }
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: FAIL，并列出所有缺失文本，因为 skill 尚未创建。

- [ ] **Step 2: 创建完整的收尾 Skill**

Create `skills/finishing-a-development-branch/SKILL.md`:

````markdown
---
name: finishing-a-development-branch
description: Use after implementation is complete, or manually when local development commits need safe consolidation and branch disposition.
---

# Finishing a Development Branch

## Overview

Finish a development branch safely: verify the repository, optionally
consolidate unpublished implementation commits, verify equivalence, then offer
branch disposition choices.

**Announce at start:** "I'm using the finishing-a-development-branch skill to safely finish this development branch."

## Inputs

Prefer a trusted `implementation-base` passed by `executing-plans`. The eligible
range is exactly:

```text
implementation-base..HEAD
```

This boundary excludes the already committed spec and implementation plan.

If invoked manually, inspect recent local history and propose a candidate base.
Show the candidate base, commits that remain unchanged, and commits eligible
for rewriting. Obtain explicit user confirmation or correction before
continuing. Never rewrite from an inferred boundary without approval.

## Step 1: Verify the Starting State

1. Run the implementation test suite. Stop if it fails.
2. Run `git status --short`. Stop unless the working tree and index are clean.
3. Determine the current branch. Stop on `main`, `master`, or another protected
   primary branch.
4. Verify the base is an ancestor of `HEAD`.
5. Run `git fetch --all --prune`. If remote references cannot be refreshed,
   stop because unpublished status cannot be proved.
6. List the eligible commits in oldest-first order.
7. Check whether any eligible commit is reachable from any remote-tracking
   branch. If any is reachable, stop and report that published history will not
   be rewritten.

Useful commands:

```bash
git status --short
git branch --show-current
git merge-base --is-ancestor <implementation-base> HEAD
git fetch --all --prune
git log --reverse --oneline <implementation-base>..HEAD
git branch -r --contains <commit>
```

If the eligible range is empty, skip commit consolidation and proceed to
branch disposition.

## Step 2: Choose Commit Handling

Show the confirmed eligible range and ask the user to choose:

1. **Keep existing commits**
2. **Consolidate into one commit**
3. **Reorganize into semantic commits**

Keeping existing commits does not create a backup and does not rewrite
history. Proceed directly to branch disposition.

## Step 3A: Propose One Commit

Inspect the full eligible diff and propose one commit subject and optional
body. Show:

- the base and current `HEAD`;
- every commit that will be replaced;
- the proposed final commit message;
- confirmation that a backup branch will be retained.

Obtain explicit approval before creating a backup or rewriting history.

## Step 3B: Propose Multiple Semantic Commits

Group changes by independently explainable behavior, not by plan task number
or file type. Keep tests with the behavior they verify. Each commit should
leave the repository valid and preferably test-passing.

For every proposed commit show:

- commit subject;
- affected files;
- concise change summary;
- ordering or dependency rationale where needed.

Show the complete ordered proposal and obtain explicit user approval. Revise
the proposal when requested. Do not rewrite until the complete grouping is
approved.

## Step 4: Record State and Create the Backup

For either rewrite mode, record:

```bash
git rev-parse HEAD
git rev-parse HEAD^{tree}
git diff --binary <implementation-base> HEAD
```

Create a unique backup named:

```text
backup/<current-branch>-before-finish-<YYYYMMDD-HHMMSS>
```

Verify it resolves to the recorded original `HEAD`:

```bash
git branch <backup-branch> <original-head>
git rev-parse <backup-branch>
```

If backup creation or verification fails, stop without rewriting history.
Never delete the backup branch automatically.

## Step 5: Rebuild the Approved History

Rebuild from `implementation-base` according to the approved proposal. Do not
use an opaque interactive rebase whose result was not shown in advance.

For one commit, move the branch to the base while keeping the final content,
then commit with the approved message:

```bash
git reset --soft <implementation-base>
git commit
```

For multiple semantic commits, first keep the final content in the working tree
without staging it:

```bash
git reset --mixed <implementation-base>
```

Then reconstruct each approved group in order:

1. For files wholly owned by the current group, stage exact paths with
   `git add -- <paths>`.
2. For a file shared by multiple groups, build a patch containing only the
   approved hunks and stage it with `git apply --cached <group.patch>`.
3. Inspect `git diff --cached --stat` and `git diff --cached`.
4. Confirm the staged diff contains only the approved group.
5. Commit with the approved message.
6. Run the tests assigned to that semantic commit before continuing.

After the final group, `git status --short` must show no remaining changes. Do
not silently change the approved grouping.

If exact grouping cannot be reproduced safely because changes overlap at hunk
level, stop and ask the user whether to revise the grouping. Do not guess.

## Step 6: Verify the Rewrite

Verify all of the following:

```bash
git rev-parse <backup-branch>
git rev-parse HEAD^{tree}
git diff --exit-code <backup-branch> HEAD
git status --short
```

- the backup still points to the original `HEAD`;
- new `HEAD^{tree}` equals the recorded original tree;
- the final diff between the backup and new `HEAD` is empty;
- the working tree and index are clean;
- the implementation tests pass again.

If any check fails, stop immediately. Do not automatically roll back, delete
references, continue to branch disposition, or retry with another strategy.
Report:

- failed operation;
- current branch and `HEAD`;
- backup branch and commit;
- completed verification results;
- informational recovery command:

```bash
git reset --hard <backup-branch>
```

Never execute the recovery command automatically.

## Step 7: Choose Branch Disposition

After successful verification, identify the target branch and ask the user to
confirm it, then choose:

1. Merge the branch locally.
2. Push the branch and create a pull request.
3. Keep the branch as-is.

For a local merge:

```bash
git checkout <target-branch>
git pull --ff-only
git merge <development-branch>
<run implementation tests>
```

If checkout, pull, merge, or tests fail, stop and report the state. Do not
delete the development branch. Offer deletion only as a separate action after
successful tests and explicit confirmation.

For a pull request:

```bash
git push -u origin <development-branch>
gh pr create --base <target-branch> --head <development-branch>
```

Show the proposed PR title and body before running `gh pr create`. If `gh` is
unavailable or either command fails, stop and report the failure.

Keeping the branch performs no Git mutation.

Before any action that deletes a commit, branch, or worktree, explain the exact
destructive effect and obtain separate explicit confirmation. The
consolidation backup branch must remain intact in every path.

At completion, report:

- final branch and `HEAD`;
- tests run and results;
- selected disposition;
- backup branch, if created;
- optional manual cleanup command:

```bash
git branch -D <backup-branch>
```

Do not run the cleanup command.

## Safety Rules

- Never rewrite a dirty working tree.
- Never rewrite a protected primary branch.
- Never rewrite commits whose unpublished status cannot be proved.
- Never include commits before the confirmed implementation base.
- Never create a backup for the keep-existing-commits path.
- Always create and verify a backup before either rewrite mode.
- Never delete the backup automatically.
- Never rewrite before the user approves the exact proposal.
- Never continue after failed post-rewrite verification.
````

- [ ] **Step 3: 运行结构检查**

Run:

```powershell
$path = "skills/finishing-a-development-branch/SKILL.md"
$required = @(
  "name: finishing-a-development-branch",
  "implementation-base..HEAD",
  "git fetch",
  "backup/<current-branch>-before-finish-<YYYYMMDD-HHMMSS>",
  "Keep existing commits",
  "Consolidate into one commit",
  "Reorganize into semantic commits",
  "git reset --hard <backup-branch>"
)
$text = Get-Content -Raw $path
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 4: 检查危险路径均有明确禁令**

Run:

```powershell
$text = Get-Content -Raw "skills/finishing-a-development-branch/SKILL.md"
$required = @(
  "Never delete the backup automatically",
  "Never execute the recovery command automatically",
  "Never rewrite a dirty working tree",
  "Never rewrite a protected primary branch",
  "Never rewrite commits whose unpublished status cannot be proved",
  "Never rewrite before the user approves the exact proposal"
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 5: Commit**

```bash
git add skills/finishing-a-development-branch/SKILL.md
git commit -m "feat: add development branch finishing skill"
```

### Task 2: 添加 Skill 元数据并更新清单

**Files:**
- Create: `skills/finishing-a-development-branch/agents/openai.yaml`
- Modify: `README.md`

**Interfaces:**
- Consumes: Task 1 新增的 `$finishing-a-development-branch` skill。
- Produces: 手动调用入口和仓库级 skill 清单说明。

- [ ] **Step 1: 写出元数据检查并确认当前失败**

Run:

```powershell
$yaml = "skills/finishing-a-development-branch/agents/openai.yaml"
$readme = Get-Content -Raw README.md
if (-not (Test-Path $yaml)) { Write-Output "missing agent metadata"; exit 1 }
if (-not $readme.Contains("finishing-a-development-branch")) {
  Write-Output "missing README entry"
  exit 1
}
```

Expected: FAIL，报告缺少 agent 元数据。

- [ ] **Step 2: 创建 Agent 元数据**

Create `skills/finishing-a-development-branch/agents/openai.yaml`:

```yaml
interface:
  display_name: "Finishing a Development Branch"
  short_description: "Safely consolidate commits and finish a branch"
  default_prompt: "Use $finishing-a-development-branch to safely finish this development branch."

policy:
  allow_implicit_invocation: false
```

- [ ] **Step 3: 更新 README**

Append this entry to the skill list in `README.md`:

```markdown
- **finishing-a-development-branch**: 安全整理未推送的实施提交，保留备份分支，并引导完成开发分支处置。
```

- [ ] **Step 4: 验证元数据和清单**

Run:

```powershell
$yaml = Get-Content -Raw "skills/finishing-a-development-branch/agents/openai.yaml"
$readme = Get-Content -Raw README.md
$checks = @(
  $yaml.Contains('display_name: "Finishing a Development Branch"'),
  $yaml.Contains('allow_implicit_invocation: false'),
  $readme.Contains("finishing-a-development-branch")
)
if ($checks -contains $false) { exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 5: Commit**

```bash
git add skills/finishing-a-development-branch/agents/openai.yaml README.md
git commit -m "docs: register development branch finishing skill"
```

### Task 3: 将 Executing Plans 接入收尾流程

**Files:**
- Modify: `skills/executing-plans/SKILL.md`

**Interfaces:**
- Consumes: 当前 `HEAD`、计划任务和最终验证结果。
- Produces: 在实施前记录的 `implementation-base`，以及成功后对 `$finishing-a-development-branch` 的强制交接。

- [ ] **Step 1: 写出交接检查并确认当前失败**

Run:

```powershell
$text = Get-Content -Raw "skills/executing-plans/SKILL.md"
$required = @(
  'implementation-base',
  'git rev-parse HEAD',
  '$finishing-a-development-branch',
  'implementation-base..HEAD'
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: FAIL，并列出缺失文本。

- [ ] **Step 2: 在开始实施前记录可信边界**

In `skills/executing-plans/SKILL.md`, replace the paragraph after Step 1:

````markdown
Before starting implementation, report the current branch and working tree
status. After confirming the plan commit is already present and the working
tree is in an acceptable starting state, record:

```bash
git rev-parse HEAD
```

Treat this exact commit as `implementation-base`. Record it immediately before
the first implementation task. Do not move the boundary later, even when tasks
create intermediate commits. The implementation range handed to the finishing
skill is `implementation-base..HEAD`, which keeps the committed spec and plan
outside the rewrite range.
````

- [ ] **Step 3: 将完成阶段改为强制交接**

Replace the existing `### Step 3: Complete Development` section with:

```markdown
### Step 3: Complete Development

After all tasks complete:

1. Run the plan-specified final verification if present; otherwise run the
   relevant full test suite and report that no final verification was
   specified.
2. Check `git status`.
3. If verification fails or the working tree is not ready for finishing, stop
   and report the issue.
4. Summarize implementation changes, tests, and remaining risks.
5. Invoke `$finishing-a-development-branch` and pass the recorded
   `implementation-base`. State that the eligible implementation range is
   exactly `implementation-base..HEAD`.

Do not rewrite commits inside `executing-plans`. Commit consolidation, backup
creation, equivalence verification, and branch disposition belong exclusively
to `finishing-a-development-branch`.
```

- [ ] **Step 4: 更新 Remember 约束**

Append these entries under `## Remember`:

```markdown
- Record `implementation-base` immediately before the first implementation task
- Never move the implementation boundary after execution starts
- Always hand successful execution to `$finishing-a-development-branch`
- Never rewrite implementation commits inside this skill
```

- [ ] **Step 5: 验证交接定义**

Run:

```powershell
$text = Get-Content -Raw "skills/executing-plans/SKILL.md"
$required = @(
  'implementation-base',
  'git rev-parse HEAD',
  '$finishing-a-development-branch',
  'implementation-base..HEAD',
  'Do not rewrite commits inside `executing-plans`',
  'Never move the implementation boundary'
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 6: Commit**

```bash
git add skills/executing-plans/SKILL.md
git commit -m "feat: hand completed plans to branch finishing"
```

### Task 4: 完成跨 Skill 验收

**Files:**
- Verify: `skills/finishing-a-development-branch/SKILL.md`
- Verify: `skills/finishing-a-development-branch/agents/openai.yaml`
- Verify: `skills/executing-plans/SKILL.md`
- Verify: `skills/writing-plans/SKILL.md`
- Verify: `README.md`

**Interfaces:**
- Consumes: Tasks 1-3 的全部变更。
- Produces: 对批准 spec 的覆盖证明，以及干净、可交付的仓库状态。

- [ ] **Step 1: 检查 spec 与 plan 提交仍由边界排除**

Run:

```powershell
$executing = Get-Content -Raw "skills/executing-plans/SKILL.md"
$writing = Get-Content -Raw "skills/writing-plans/SKILL.md"
$checks = @(
  $executing.Contains("record it immediately before"),
  $executing.Contains("implementation-base..HEAD"),
  $executing.Contains("keeps the committed spec and plan outside"),
  $writing.Contains("commit the plan before offering")
)
if ($checks -contains $false) { exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 2: 检查三条用户路径及两条备份路径**

Run:

```powershell
$text = Get-Content -Raw "skills/finishing-a-development-branch/SKILL.md"
$required = @(
  "Keep existing commits",
  "does not create a backup",
  "Consolidate into one commit",
  "Reorganize into semantic commits",
  "For either rewrite mode",
  "Never delete the backup branch automatically"
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 3: 检查远端、工作区和受保护分支安全门**

Run:

```powershell
$text = Get-Content -Raw "skills/finishing-a-development-branch/SKILL.md"
$required = @(
  "git status --short",
  "git fetch --all --prune",
  "git branch -r --contains <commit>",
  "Never rewrite a dirty working tree",
  "Never rewrite a protected primary branch",
  "Never rewrite commits whose unpublished status cannot be proved"
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 4: 检查重写后验证和失败恢复约束**

Run:

```powershell
$text = Get-Content -Raw "skills/finishing-a-development-branch/SKILL.md"
$required = @(
  "git rev-parse HEAD^{tree}",
  "git diff --exit-code <backup-branch> HEAD",
  "the implementation tests pass again",
  "Do not automatically roll back",
  "git reset --hard <backup-branch>",
  "Never execute the recovery command automatically"
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing.Count -gt 0) { $missing; exit 1 }
```

Expected: PASS，无输出。

- [ ] **Step 5: 运行格式和仓库状态检查**

Run:

```powershell
git diff --check
git status --short
```

Expected: `git diff --check` 无输出；`git status --short` 无输出，表示所有任务提交均已完成且工作区干净。

- [ ] **Step 6: 查看最终提交序列**

Run:

```powershell
git log --oneline --decorate -8
```

Expected: 从新到旧依次包含：

```text
feat: hand completed plans to branch finishing
docs: register development branch finishing skill
feat: add development branch finishing skill
```

本任务仅做最终验证，不额外创建空提交。
