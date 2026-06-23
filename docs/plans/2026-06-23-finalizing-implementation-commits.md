# Finalizing Implementation Commits Implementation Plan

> **For agentic workers:** REQUIRED SKILL: Use $executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the branch-oriented finishing skill with a narrowly scoped skill that safely keeps, squashes, or semantically rebuilds local implementation commits and then stops.

**Architecture:** Rename the existing skill directory and rewrite its instructions around the single range `implementation-base..HEAD`. Preserve `executing-plans` as the trusted producer of `implementation-base`, while allowing manual invocation with a user-confirmed readable range. Keep all history-rewrite safety inside the renamed skill and remove push, PR, branch merge, discard, detached-HEAD, and worktree-cleanup behavior.

**Tech Stack:** Markdown skill definitions, YAML agent metadata, Git CLI, PowerShell assertions, Codex skill validation scripts.

## Global Constraints

- The runtime skill name is `finalizing-implementation-commits`.
- The skill may be invoked by `executing-plans` or explicitly by a user.
- The only eligible range is `implementation-base..HEAD`.
- Manual invocation shows only eligible commits in readable `git log --oneline` form and requires range confirmation.
- Initial test failure or a dirty working tree/index ends the workflow before commit analysis.
- Zero or one eligible commit produces an “无需整理” report and ends.
- Multiple commits produce two or three choices; a semantic option embeds only proposed commit subjects as an oldest-first sub-list.
- The user's choice is execution authorization; do not request a second confirmation.
- Keeping existing commits does not create a backup.
- Squashing or semantic rebuilding requires successful remote refresh, proof that no eligible commit is on a remote-tracking branch, and a verified backup of the original `HEAD`.
- Never push, force-push, create a PR, merge branches, delete work, or clean a worktree.
- Never automatically delete the backup; only show an optional manual deletion command after success.
- After reporting the result, the skill must end.

---

### Task 1: Rename and Rewrite the Commit Finalization Skill

**Files:**
- Move: `skills/finishing-a-development-branch/` → `skills/finalizing-implementation-commits/`
- Replace: `skills/finalizing-implementation-commits/SKILL.md`

**Interfaces:**
- Consumes: trusted `implementation-base` from `executing-plans`, or a manually inferred range explicitly confirmed by the user.
- Produces: one terminal result—unchanged commits, one squashed commit, or an approved oldest-first semantic commit sequence—plus a retained backup only when history was rewritten.

- [ ] **Step 1: Run the new-skill contract check and verify it fails**

Run:

```powershell
$new = "skills/finalizing-implementation-commits/SKILL.md"
$old = "skills/finishing-a-development-branch/SKILL.md"
if (-not (Test-Path $old)) { throw "expected existing skill at $old" }
if (Test-Path $new) { throw "new skill already exists before rename" }
Write-Error "finalizing-implementation-commits has not been created"
```

Expected: FAIL with `finalizing-implementation-commits has not been created`.

- [ ] **Step 2: Move the skill directory**

Run:

```powershell
$source = Resolve-Path "skills/finishing-a-development-branch"
$skillsRoot = Resolve-Path "skills"
$destination = Join-Path $skillsRoot "finalizing-implementation-commits"
if (-not $source.Path.StartsWith($skillsRoot.Path, [System.StringComparison]::OrdinalIgnoreCase)) {
  throw "source is outside skills root"
}
if (Test-Path $destination) { throw "destination already exists" }
Move-Item -LiteralPath $source.Path -Destination $destination
```

Expected: the old directory no longer exists and the new directory contains `SKILL.md` and `agents/openai.yaml`.

- [ ] **Step 3: Replace the skill definition**

Replace `skills/finalizing-implementation-commits/SKILL.md` with:

````markdown
---
name: finalizing-implementation-commits
description: Safely keep, squash, or semantically rebuild local implementation commits after tests pass. Use when executing-plans hands off a trusted implementation-base, or when a user explicitly invokes the skill later and confirms a readable local commit range.
---

# Finalizing Implementation Commits

## Overview

Finalize local implementation commits without publishing, merging branches, or
cleaning worktrees.

**Core principle:** Test -> Require clean state -> Confirm range -> Choose
history shape -> Prove unpublished -> Back up -> Rewrite -> Verify -> Stop.

**Announce at start:** "I'm using the finalizing-implementation-commits skill to finalize local implementation commits."

## Hard Scope

This skill may keep existing commits, squash eligible commits into one commit,
or rebuild them as an oldest-first semantic sequence.

Never:

- push or force-push;
- create a pull request;
- merge branches;
- delete commits, branches, or worktrees;
- clean up a worktree;
- automatically delete a backup branch;
- continue into another Git workflow after reporting the result.

## Step 1: Run Initial Tests

Run the implementation tests appropriate to the repository.

If any test fails, show the failure and stop immediately. Do not inspect,
analyze, or rewrite implementation commits.

## Step 2: Require a Clean Repository

Run:

```bash
git status --short
```

If it prints anything, show the status and stop. Do not stash, commit, reset,
checkout, or discard changes automatically. The user may clean the repository
and invoke the skill again.

## Step 3: Establish the Eligible Range

When `executing-plans` supplies `implementation-base`, use that exact commit.
The eligible range is:

```text
implementation-base..HEAD
```

When invoked manually, inspect recent local history and infer a candidate
`implementation-base`. Show only the commits eligible for rewriting, using
readable `git log --oneline` formatting:

```text
4ab82d1 feat: add parser
703cd92 test: cover parser errors
cf801a4 refactor: simplify parser state
```

Do not show bare hashes without subjects. Ask the user to confirm or correct
the candidate range before continuing.

For either source, verify:

```bash
git cat-file -e <implementation-base>^{commit}
git merge-base --is-ancestor <implementation-base> HEAD
git log --reverse --oneline <implementation-base>..HEAD
```

If the base is invalid or is not an ancestor of `HEAD`, report the failure and
stop without changing history.

## Step 4: Count Eligible Commits

Count commits in `implementation-base..HEAD`.

If the range contains zero or one commit, report that no organization is
needed and stop. Do not create a backup.

For multiple commits, inspect each commit's purpose, files, diff, tests, and
dependencies before presenting choices.

## Step 5: Present Commit Choices

If no useful semantic grouping exists, present:

```text
1. Keep existing commits
2. Squash into one commit
```

If a useful semantic grouping exists, present the proposed new commit subjects
directly beneath the third choice. Preserve the original oldest-first
`git log` order:

```text
1. Keep existing commits
2. Squash into one commit
3. Rebuild as semantic commits
   - feat: add parser
   - test: cover parser errors
```

Before the user chooses, do not show per-group files, summaries, or dependency
explanations. Never offer to analyze or plan the grouping later.

The user's choice is execution authorization. Do not request a second
confirmation.

If the user keeps existing commits, report the unchanged `HEAD`, initial test
result, and that no backup was created. Then stop.

## Step 6: Apply Rewrite Safety Gates

For squash or semantic rebuild, recheck:

```bash
test -z "$(git status --porcelain)"
git cat-file -e <implementation-base>^{commit}
git merge-base --is-ancestor <implementation-base> HEAD
```

Then refresh remote references:

```bash
git fetch --all --prune
```

If fetch fails, report that publication status cannot be verified and stop.

For every eligible commit, run:

```bash
git branch -r --contains <commit>
```

If any remote-tracking branch contains an eligible commit, report the commit
and containing references, refuse to rewrite, and stop. Never suggest
force-push as a workaround.

The current branch name, including `main` or `master`, does not prevent a
rewrite. Publication status and the safety gates are the boundary.

## Step 7: Record State and Create the Backup

Record:

```bash
ORIGINAL_HEAD=$(git rev-parse HEAD)
ORIGINAL_TREE=$(git rev-parse HEAD^{tree})
ORIGINAL_DIFF=$(mktemp)
git diff --binary <implementation-base> "$ORIGINAL_HEAD" > "$ORIGINAL_DIFF"
```

Create a unique backup:

```text
backup/<current-branch>-before-finalize-<YYYYMMDD-HHMMSS>
```

Create and verify it:

```bash
git branch <backup-branch> "$ORIGINAL_HEAD"
test "$(git rev-parse <backup-branch>)" = "$ORIGINAL_HEAD"
```

If creation or verification fails, stop before rewriting. Never delete the
backup automatically.

## Step 8A: Squash Into One Commit

Generate an accurate final message from the complete eligible change, then:

```bash
git reset --soft <implementation-base>
git commit -m "<generated-final-message>"
```

The choice to squash already authorized this message and operation.

## Step 8B: Rebuild Semantic Commits

Reset while preserving final content:

```bash
git reset --mixed <implementation-base>
```

Rebuild exactly the semantic subjects shown in the selected option, in that
order.

- Stage whole files with `git add -- <paths>` when a file belongs entirely to
  one group.
- For files shared by groups, stage only the current group's exact hunks.
- Keep tests with the behavior they verify.
- Inspect the staged diff before every commit.
- Keep every intermediate commit valid.
- Run tests associated with the group whenever practical.

If the displayed grouping cannot be reproduced safely, stop. Do not switch to
squash or invent a different grouping.

## Step 9: Verify the Rewrite

Run:

```bash
test "$(git rev-parse <backup-branch>)" = "$ORIGINAL_HEAD"
test "$(git rev-parse HEAD^{tree})" = "$ORIGINAL_TREE"
NEW_DIFF=$(mktemp)
git diff --binary <implementation-base> HEAD > "$NEW_DIFF"
cmp -s "$ORIGINAL_DIFF" "$NEW_DIFF"
git diff --exit-code <backup-branch> HEAD
test -z "$(git status --porcelain)"
<run implementation tests>
rm -- "$ORIGINAL_DIFF" "$NEW_DIFF"
```

All checks and tests must pass. Remove temporary diff files only after every
verification succeeds.

If any operation or verification fails after backup creation, stop
immediately. Do not roll back, delete or move references, retry, or use another
rewrite strategy. Report:

- the failed operation;
- current branch and `HEAD`;
- backup branch and target commit;
- completed checks and results;
- this informational recovery command:

```bash
git reset --hard <backup-branch>
```

Never execute the recovery command automatically.

## Step 10: Report and Stop

On successful rewrite, report:

- new `HEAD`;
- squash or semantic-rebuild result;
- initial and final test results;
- retained backup branch;
- optional manual cleanup command:

```bash
git branch -D <backup-branch>
```

Do not run the cleanup command. End immediately without offering push, PR,
branch merge, deletion, or worktree actions.
````

- [ ] **Step 4: Run structural and prohibited-behavior checks**

Run:

```powershell
$path = "skills/finalizing-implementation-commits/SKILL.md"
$text = Get-Content -Raw $path
$required = @(
  "name: finalizing-implementation-commits",
  "implementation-base..HEAD",
  "git log --reverse --oneline",
  "zero or one commit",
  "The user's choice is execution authorization",
  "git fetch --all --prune",
  "git branch -r --contains <commit>",
  "backup/<current-branch>-before-finalize-<YYYYMMDD-HHMMSS>",
  "git reset --soft <implementation-base>",
  "git reset --mixed <implementation-base>",
  "cmp -s",
  "End immediately"
)
$forbidden = @(
  "gh pr create",
  "git push -u",
  "git pull --ff-only",
  "git worktree remove",
  "Type 'discard' to confirm"
)
$missing = $required | Where-Object { -not $text.Contains($_) }
$present = $forbidden | Where-Object { $text.Contains($_) }
if ($missing) { Write-Output "Missing:"; $missing }
if ($present) { Write-Output "Forbidden:"; $present }
if ($missing -or $present) { exit 1 }
```

Expected: PASS with no output.

- [ ] **Step 5: Validate the renamed skill**

Run:

```powershell
python "C:\Users\as\.codex\skills\.system\skill-creator\scripts\quick_validate.py" "skills\finalizing-implementation-commits"
```

Expected: `Skill is valid!`

- [ ] **Step 6: Commit**

```bash
git add -- skills/finishing-a-development-branch skills/finalizing-implementation-commits
git commit -m "feat: narrow finishing workflow to implementation commits"
```

### Task 2: Update Skill Metadata and Repository Registration

**Files:**
- Replace: `skills/finalizing-implementation-commits/agents/openai.yaml`
- Modify: `README.md`

**Interfaces:**
- Consumes: the renamed `$finalizing-implementation-commits` skill.
- Produces: an explicit-only manual invocation surface and accurate repository documentation.

- [ ] **Step 1: Run metadata and registration checks and verify they fail**

Run:

```powershell
$yaml = Get-Content -Raw "skills/finalizing-implementation-commits/agents/openai.yaml"
$readme = Get-Content -Raw "README.md"
$checks = @(
  $yaml.Contains('display_name: "Finalizing Implementation Commits"'),
  $yaml.Contains('$finalizing-implementation-commits'),
  $readme.Contains("finalizing-implementation-commits"),
  -not $readme.Contains("finishing-a-development-branch")
)
if ($checks -contains $false) { Write-Error "metadata and README still describe the old skill"; exit 1 }
```

Expected: FAIL with `metadata and README still describe the old skill`.

- [ ] **Step 2: Replace agent metadata**

Replace `skills/finalizing-implementation-commits/agents/openai.yaml` with:

```yaml
interface:
  display_name: "Finalizing Implementation Commits"
  short_description: "Safely organize unpublished implementation commits"
  default_prompt: "Use $finalizing-implementation-commits to safely finalize these local implementation commits."

policy:
  allow_implicit_invocation: false
```

- [ ] **Step 3: Replace the README entry**

Replace:

```markdown
- **finishing-a-development-branch**: 安全整理未推送的实施提交，保留备份分支，并引导完成开发分支处置。
```

With:

```markdown
- **finalizing-implementation-commits**: 安全保留、压缩或按语义重建未发布的本地实施提交，完成后立即结束。
```

- [ ] **Step 4: Validate metadata and registration**

Run:

```powershell
$yaml = Get-Content -Raw "skills/finalizing-implementation-commits/agents/openai.yaml"
$readme = Get-Content -Raw "README.md"
$checks = @(
  $yaml.Contains('display_name: "Finalizing Implementation Commits"'),
  $yaml.Contains('short_description: "Safely organize unpublished implementation commits"'),
  $yaml.Contains('$finalizing-implementation-commits'),
  $yaml.Contains('allow_implicit_invocation: false'),
  $readme.Contains("finalizing-implementation-commits"),
  -not $readme.Contains("finishing-a-development-branch")
)
if ($checks -contains $false) { exit 1 }
```

Expected: PASS with no output.

- [ ] **Step 5: Commit**

```bash
git add -- skills/finalizing-implementation-commits/agents/openai.yaml README.md
git commit -m "docs: register implementation commit finalization skill"
```

### Task 3: Hand Completed Plans to the Renamed Skill

**Files:**
- Modify: `skills/executing-plans/SKILL.md`

**Interfaces:**
- Consumes: the `implementation-base` captured immediately before the first implementation task.
- Produces: a mandatory handoff to `$finalizing-implementation-commits` after successful implementation verification.

- [ ] **Step 1: Run the renamed-handoff check and verify it fails**

Run:

```powershell
$text = Get-Content -Raw "skills/executing-plans/SKILL.md"
$checks = @(
  $text.Contains('$finalizing-implementation-commits'),
  -not $text.Contains('$finishing-a-development-branch'),
  $text.Contains("finalize implementation commits"),
  -not $text.Contains("integration options")
)
if ($checks -contains $false) { Write-Error "executing-plans still hands off to the old branch workflow"; exit 1 }
```

Expected: FAIL with `executing-plans still hands off to the old branch workflow`.

- [ ] **Step 2: Replace the completion handoff**

Replace the existing `### Step 3: Complete Development` section with:

```markdown
### Step 3: Complete Development

After all tasks complete and verify successfully:

- Announce: "I'm using the finalizing-implementation-commits skill to finalize local implementation commits."
- **REQUIRED SUB-SKILL:** Use `$finalizing-implementation-commits`
- Pass the recorded `implementation-base`
- State that the eligible range is exactly `implementation-base..HEAD`
- Follow that skill through commit analysis, any selected safe rewrite, final
  verification, and its mandatory terminal report

Do not rewrite implementation commits inside this skill. Do not push, create a
PR, merge branches, or continue with branch disposition after the finalizing
skill ends.
```

- [ ] **Step 3: Update surrounding terminology**

In `skills/executing-plans/SKILL.md`:

- Change `passed to the finishing skill` to `passed to the commit-finalization skill`.
- Replace the final Remember entry:

```markdown
- Leave commit rewriting to `$finishing-a-development-branch`
```

With:

```markdown
- Leave implementation commit rewriting to `$finalizing-implementation-commits`
- Treat the finalizing skill's report as the terminal state; never push or continue into branch disposition
```

- [ ] **Step 4: Validate the handoff and terminal boundary**

Run:

```powershell
$text = Get-Content -Raw "skills/executing-plans/SKILL.md"
$required = @(
  'implementation-base',
  'implementation-base..HEAD',
  '$finalizing-implementation-commits',
  'Do not rewrite implementation commits inside this skill',
  'Do not push, create a',
  'terminal state'
)
$forbidden = @(
  '$finishing-a-development-branch',
  'present integration options',
  'execute the user''s choice'
)
$missing = $required | Where-Object { -not $text.Contains($_) }
$present = $forbidden | Where-Object { $text.Contains($_) }
if ($missing) { Write-Output "Missing:"; $missing }
if ($present) { Write-Output "Forbidden:"; $present }
if ($missing -or $present) { exit 1 }
```

Expected: PASS with no output.

- [ ] **Step 5: Commit**

```bash
git add -- skills/executing-plans/SKILL.md
git commit -m "feat: hand plans to commit finalization"
```

### Task 4: Verify Cross-Skill Scope and Safety

**Files:**
- Verify: `skills/finalizing-implementation-commits/SKILL.md`
- Verify: `skills/finalizing-implementation-commits/agents/openai.yaml`
- Verify: `skills/executing-plans/SKILL.md`
- Verify: `README.md`
- Verify absence: `skills/finishing-a-development-branch/`

**Interfaces:**
- Consumes: Tasks 1–3.
- Produces: evidence that the runtime workflow matches the approved design and contains no branch-disposition behavior.

- [ ] **Step 1: Verify names and paths**

Run:

```powershell
$checks = @(
  (Test-Path "skills/finalizing-implementation-commits/SKILL.md"),
  (Test-Path "skills/finalizing-implementation-commits/agents/openai.yaml"),
  -not (Test-Path "skills/finishing-a-development-branch"),
  (Get-Content -Raw "skills/finalizing-implementation-commits/SKILL.md").Contains("name: finalizing-implementation-commits")
)
if ($checks -contains $false) { exit 1 }
```

Expected: PASS with no output.

- [ ] **Step 2: Verify decision shape and backup timing**

Run:

```powershell
$text = Get-Content -Raw "skills/finalizing-implementation-commits/SKILL.md"
$required = @(
  "zero or one commit",
  "Keep existing commits",
  "Squash into one commit",
  "Rebuild as semantic commits",
  "Preserve the original oldest-first",
  "The user's choice is execution authorization",
  "Do not create a backup",
  "Create a unique backup",
  "Never delete the backup automatically"
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing) { $missing; exit 1 }
```

Expected: PASS with no output.

- [ ] **Step 3: Verify hard safety gates**

Run:

```powershell
$text = Get-Content -Raw "skills/finalizing-implementation-commits/SKILL.md"
$required = @(
  "If any test fails",
  "If it prints anything",
  "git fetch --all --prune",
  "git branch -r --contains <commit>",
  "refuse to rewrite",
  'test "$(git rev-parse <backup-branch>)" = "$ORIGINAL_HEAD"',
  "cmp -s",
  "git diff --exit-code <backup-branch> HEAD",
  "Never execute the recovery command automatically"
)
$missing = $required | Where-Object { -not $text.Contains($_) }
if ($missing) { $missing; exit 1 }
```

Expected: PASS with no output.

- [ ] **Step 4: Verify prohibited runtime behavior is absent**

Run:

```powershell
$runtime = @(
  (Get-Content -Raw "skills/finalizing-implementation-commits/SKILL.md"),
  (Get-Content -Raw "skills/executing-plans/SKILL.md")
) -join "`n"
$forbidden = @(
  "gh pr create",
  "git push -u",
  "git push",
  "git pull --ff-only",
  "git merge <",
  "git worktree remove",
  "Discard this work",
  "detached HEAD"
)
$present = $forbidden | Where-Object { $runtime.Contains($_) }
if ($present) { $present; exit 1 }
```

Expected: PASS with no output.

- [ ] **Step 5: Run skill validation and repository checks**

Run:

```powershell
python "C:\Users\as\.codex\skills\.system\skill-creator\scripts\quick_validate.py" "skills\finalizing-implementation-commits"
git diff --check
git status --short
```

Expected:

```text
Skill is valid!
```

`git diff --check` prints nothing. `git status --short` prints nothing after all task commits.

- [ ] **Step 6: Review the implementation commit sequence**

Run:

```powershell
git log --oneline --decorate -6
```

Expected to include, newest first:

```text
feat: hand plans to commit finalization
docs: register implementation commit finalization skill
feat: narrow finishing workflow to implementation commits
```

This task performs verification only and creates no empty commit.
