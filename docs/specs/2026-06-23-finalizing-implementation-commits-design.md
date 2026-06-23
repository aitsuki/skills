# 实施提交整理设计

## 目标

新增 `finalizing-implementation-commits` skill，用于安全整理实施过程中产生的本地提交。
它既可以由 `executing-plans` 在实施完成后调用，也可以由用户稍后手动调用。

该 skill 只负责提交整理，不负责开发分支生命周期或远程发布。无论当前工作位于
`main`、`master` 还是其他分支，其行为都只取决于可整理提交范围和安全门。

skill 明确禁止：

- push 或 force-push；
- 创建 PR；
- 合并分支；
- 删除提交、分支或 worktree；
- 清理 worktree；
- 自动删除 backup 分支。

完成提交处理和验证后，skill 报告结果并立即结束。

## 与现有工作流的关系

`executing-plans` 在第一个实施任务开始前记录当前 `HEAD`，并将其作为
`implementation-base`。实施完成且最终测试通过后，它调用
`finalizing-implementation-commits` 并传递该边界。

设计文档和实施计划的提交位于 `implementation-base` 之前，因此不会进入整理范围。
`executing-plans` 不自行压缩或重建实施提交。

用户也可以单独调用 `finalizing-implementation-commits`。这适用于此前选择保持现有
提交、之后又决定整理提交历史等场景。

## 主流程

```text
运行测试
→ 检查工作区和暂存区
→ 确定并验证 implementation-base
→ 展示可整理提交
→ 统计提交数量
→ 0 或 1 个提交：提示无需整理并结束
→ 多个提交：分析并展示整理选项
→ 用户选择保持、压缩或语义重建
→ 需要重写时执行远端与 backup 安全门
→ 重写提交
→ 验证内容、状态和测试
→ 报告结果并结束
```

## 可整理范围

由 `executing-plans` 调用时，使用其传入的 `implementation-base`。允许整理的范围始终
为：

```text
implementation-base..HEAD
```

手动调用时，Agent 分析近期本地提交历史并提出候选 `implementation-base`。继续前，
必须以类似 `git log --oneline` 的可读格式，只展示候选范围内允许整理的提交，例如：

```text
4ab82d1 feat: add parser
703cd92 test: cover parser errors
cf801a4 refactor: simplify parser state
```

不得只向用户展示裸 commit hash。用户必须明确确认该范围，或者修正候选边界。

无论边界来自 `executing-plans` 还是手动确认，Agent 都必须验证：

```bash
git cat-file -e <implementation-base>^{commit}
git merge-base --is-ancestor <implementation-base> HEAD
```

边界无效时立即结束，不修改提交历史。

## 初始安全门

提交分析前，必须运行实施相关测试。测试失败时展示失败信息并立即结束，不分析或整理
提交。

测试通过后检查：

```bash
git status --short
```

工作区或暂存区不干净时，展示状态并立即结束。skill 不得自动 stash、commit、reset、
checkout 或丢弃修改。用户清理工作区后可以重新调用。

## 提交数量

验证并展示可整理范围后，统计 `implementation-base..HEAD` 中的提交：

- 没有提交时，提示无需整理并结束；
- 只有一个提交时，提示无需整理并结束；
- 有多个提交时，进入整理决策。

以上两种无需整理的路径都不创建 backup 分支。

## 多提交整理决策

Agent 在展示选项前分析每个提交的目的、文件、diff、测试和依赖关系，判断是否存在比
全部压缩更有价值的语义分组。

如果不存在有价值的语义分组，展示：

```text
1. 保持现有提交
2. 压缩成一个提交
```

如果存在语义分组，第三个选项直接包含拟议的新提交标题。子列表必须按照原始
`git log` 的提交顺序排列：

```text
1. 保持现有提交
2. 压缩成一个提交
3. 按语义重新整理
   - feat: add parser
   - test: cover parser errors
```

选择前不展示每组的文件、摘要或依赖说明。不得提供“稍后分析”“由 Agent 再规划”等
没有实际方案的选项。

用户选择即为执行授权，不要求第二次确认：

- 选择保持现有提交：不创建 backup，不修改历史，报告结果并结束；
- 选择压缩成一个提交：通过重写安全门后，将范围重建为一个提交；
- 选择按语义重新整理：通过重写安全门后，严格按照第三项列出的顺序重建提交。

## 历史重写安全门

压缩或语义重建前，必须依次完成：

1. 确认初始测试已经通过；
2. 再次确认工作区和暂存区干净；
3. 确认 `implementation-base` 仍有效且是当前 `HEAD` 的祖先；
4. 执行 `git fetch --all --prune` 并确认成功；
5. 确认可整理范围中的每个提交都未被任何远端跟踪分支引用；
6. 记录原始状态；
7. 创建唯一 backup 分支并验证它指向原始 `HEAD`。

检查远端发布状态时，对每个可整理提交执行：

```bash
git branch -r --contains <commit>
```

如果远端引用无法成功刷新，或任一待整理提交被远端跟踪分支引用，必须拒绝历史重写并
立即结束。skill 不得 push、force-push 或建议通过 force-push 绕过检查。

当前分支是否名为 `main`、`master` 或其他名称不影响判断。只有提交发布状态和其他
安全门决定是否允许重写。

## 原始状态与 backup

重写前记录：

- 原始 `HEAD`；
- 原始 `HEAD` 的 Git tree；
- `implementation-base` 到原始 `HEAD` 的 binary diff。

backup 分支仅在用户选择压缩或语义重建时创建。它用于在 AI 整理错误后提供人工恢复
锚点，必须指向重写前的原始 `HEAD`。

命名格式：

```text
backup/<current-branch>-before-finalize-<YYYYMMDD-HHMMSS>
```

创建后必须显式比较 backup 指向与原始 `HEAD`。创建或验证失败时立即结束，此时不得
开始重写。

skill 在任何路径都不得自动删除 backup。完成时只提供可选的手动删除命令。

## 压缩成一个提交

使用：

```bash
git reset --soft <implementation-base>
git commit -m "<final-message>"
```

Agent 根据可整理提交的整体行为生成准确的最终提交信息。用户选择“压缩成一个提交”
即授权使用该提交信息执行压缩，不再要求二次确认。

## 按语义重新整理

使用：

```bash
git reset --mixed <implementation-base>
```

然后严格按照选项中已列出的语义提交顺序重建历史：

- 按完整文件暂存时使用 `git add -- <paths>`；
- 同一文件跨多个语义提交时，只暂存属于当前组的精确 hunk；
- 测试与其验证的行为放在同一个提交；
- 每个中间提交必须保持仓库有效；
- 在实际可行时运行与该组对应的测试。

如果列出的分组无法安全复现、会产生无效的中间提交，或无法正确保留测试归属，立即
停止，不得自行改用压缩或其他重写策略。

## 重写后验证

重写完成后必须验证：

1. backup 仍然指向原始 `HEAD`；
2. 新 `HEAD` 与原始 `HEAD` 的 Git tree 相同；
3. `implementation-base` 到新旧 `HEAD` 的 binary diff 等价；
4. 原始 `HEAD` 与新 `HEAD` 的最终内容无差异；
5. 工作区和暂存区干净；
6. 实施相关测试再次通过。

只有所有检查通过，整理才算成功。

## 失败处理

创建 backup 前的失败不得修改提交历史。

创建 backup 后发生任何失败时，立即停止并保留当前现场。skill 不得：

- 自动回滚；
- 自动删除或移动引用；
- 自动重试；
- 改用另一种整理策略；
- 继续执行其他 Git 操作。

失败报告包含：

- 失败操作；
- 当前分支和 `HEAD`；
- backup 分支及其指向；
- 已完成检查及结果；
- 仅供人工参考的恢复命令，例如：

```bash
git reset --hard <backup-branch>
```

skill 只展示恢复命令，绝不执行。

## 成功报告与终点

保持现有提交时，报告：

- 最终 `HEAD`；
- 整理方式为保持现状；
- 初始测试结果；
- 未创建 backup。

重写成功时，报告：

- 新 `HEAD`；
- 整理方式为压缩或语义重建；
- 重写前后测试结果；
- 保留的 backup 分支；
- 可选的手动删除命令：

```bash
git branch -D <backup-branch>
```

报告后立即结束。不得继续提供 push、PR、分支合并、删除或 worktree 清理选项。

## 对现有 Skill 的修改

### `executing-plans`

- 继续在第一个实施任务开始前记录 `implementation-base`；
- 完成实施和最终验证后调用 `finalizing-implementation-commits`；
- 传递准确的 `implementation-base`；
- 不自行整理实施提交。

### 原 `finishing-a-development-branch`

由 `finalizing-implementation-commits` 替代。删除原 skill 中所有 push、PR、分支
合并、discard、detached HEAD 和 worktree 清理行为。

### `writing-plans`

保留任务级提交。它们在实施期间提供检查点，最终是否整理由
`finalizing-implementation-commits` 决定。

## 验收条件

- skill 不假设存在 feature branch；
- skill 在 `main` 上也能安全运行；
- skill 可以接受 `executing-plans` 传入的边界；
- skill 可以由用户手动调用并要求确认可读的候选范围；
- 0 或 1 个提交时提示无需整理并结束；
- 多提交时先完成内部分析，再动态展示两个或三个选项；
- 第三个选项直接包含按原提交顺序排列的拟议标题；
- 用户选择后不再要求二次确认；
- 测试失败或工作区不干净时立即结束；
- 远端状态不可验证或提交已发布时禁止重写；
- 保持现状时不创建 backup；
- 两种重写模式都在重写前创建并验证 backup；
- 重写后的 tree、diff、工作区和测试全部通过验证；
- backup 永不自动删除；
- skill 绝不 push、创建 PR、合并分支、删除工作或清理 worktree；
- 完成报告后立即结束。
