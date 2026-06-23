---
name: finishing-a-development-branch
description: Use when implementation is complete, tests pass, and local commits or branch integration need to be finished safely.
---

# Finishing a Development Branch

## Overview

Guide completion of development work by safely handling local implementation
commits, presenting clear integration options, and executing the chosen
workflow.

**Core principle:** Verify tests -> Detect environment -> Handle commits -> Present options -> Execute choice -> Clean up.

**Announce at start:** "I'm using the finishing-a-development-branch skill to complete this work."

## The Process

### Step 1: Verify Tests

**Before presenting options, verify tests pass:**

```bash
# Run project's test suite
npm test / cargo test / pytest / go test ./...
```

**If tests fail:**
```
Tests failing (<N> failures). Must fix before completing:

[Show failures]

Cannot proceed with commit rewriting, merge, or PR until tests pass.
```

Stop. Don't proceed to Step 2.

**If tests pass:** Continue to Step 2.

### Step 2: Detect Environment

**Determine workspace state before presenting options:**

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
git status --short
git branch --show-current
```

The working tree and index must be clean before commit rewriting. Detect
whether this is a normal repository, a named-branch worktree, or a detached
HEAD so integration and cleanup options match the environment.

### Step 3: Determine Base Branch and Implementation Range

```bash
# Try common base branches
git merge-base HEAD main 2>/dev/null || git merge-base HEAD master 2>/dev/null
```

Or ask: "This branch split from main - is that correct?"

If `$executing-plans` supplied `implementation-base`, use it. The eligible
implementation range is:

```text
implementation-base..HEAD
```

This excludes the committed spec and implementation plan.

For a manual invocation, inspect recent history and propose a candidate base.
Show the candidate base, commits that remain unchanged, and commits eligible
for rewriting. Obtain explicit confirmation before using an inferred boundary.

### Step 4: Handle Implementation Commits

If the eligible range is empty, skip this step.

If it contains exactly one commit, report that no consolidation is needed and
continue to Step 5.

For multiple commits, first analyze each commit's purpose, files, diff, tests,
and dependencies. If there is a useful semantic grouping, show the complete
ordered proposal before asking the user. For each proposed commit show:

- commit subject
- affected files
- concise change summary
- ordering or dependency rationale when needed

Keep tests with the behavior they verify. Group by independently explainable
behavior, not by plan task number or file type.

If no useful semantic grouping exists, present:

```
Implementation has multiple local commits. How should they be handled?

1. Keep existing commits
2. Consolidate into one commit

Which option?
```

If a useful semantic grouping exists, show the proposal, then present:

```
Implementation has multiple local commits. How should they be handled?

1. Keep existing commits
2. Consolidate into one commit
3. Apply the semantic commit plan shown above

Which option?
```

Never offer to analyze a semantic plan later. The proposal must already be
visible when the user chooses.

#### Option 1: Keep Existing Commits

Do not create a backup or rewrite history. Continue to Step 5.

#### Options 2 and 3: Rewrite Commits

Before proposing or executing a rewrite:

1. Refresh remote references:

```bash
git fetch --all --prune
```

2. For every eligible commit, verify no remote-tracking branch contains it:

```bash
git branch -r --contains <commit>
```

If remote state cannot be refreshed or any eligible commit has been published,
do not rewrite. Explain why and return to the keep-existing-commits path.

Publication status, not the branch name, is the safety boundary. Unpublished commits
may be rewritten on `main`, `master`, or another primary branch.

Show the exact rewrite before proceeding:

- confirmed base and current `HEAD`
- commits that will be replaced
- final commit message, or complete semantic grouping
- confirmation that a backup branch will be retained

Obtain explicit approval.

Record the original state:

```bash
git rev-parse HEAD
git rev-parse HEAD^{tree}
git diff --binary <implementation-base> HEAD
```

Create and verify:

```text
backup/<current-branch>-before-finish-<YYYYMMDD-HHMMSS>
```

```bash
git branch <backup-branch> <original-head>
git rev-parse <backup-branch>
```

If backup creation or verification fails, stop without rewriting.

For one commit:

```bash
git reset --soft <implementation-base>
git commit
```

For multiple semantic commits:

```bash
git reset --mixed <implementation-base>
```

Rebuild each approved group in order. Stage whole files with
`git add -- <paths>`; for shared files, stage exact approved hunks with
`git apply --cached <group.patch>`. Inspect the staged diff before every
commit, and run the tests assigned to each group. If the approved grouping
cannot be reproduced safely, stop and ask the user to revise it.

After rewriting, verify:

```bash
git rev-parse <backup-branch>
git rev-parse HEAD^{tree}
git diff --exit-code <backup-branch> HEAD
git status --short
```

The backup must still point to the original `HEAD`, the final tree and content
must be unchanged, the working tree must be clean, and the implementation tests
must pass again.

If verification fails, stop. Do not automatically roll back, delete
references, retry with another strategy, or continue to Step 5. Report the
failure and show, but never execute:

```bash
git reset --hard <backup-branch>
```

Never delete the backup automatically.

### Step 5: Present Integration Options

If the current branch is already the confirmed base/target branch, present:

```
Implementation complete. Changes are already on <target-branch>.

1. Push the current branch
2. Keep the branch local

Which option?
```

Otherwise, for a normal repository or named-branch worktree, present:

```
Implementation complete. What would you like to do?

1. Merge back to <base-branch> locally
2. Push and create a Pull Request
3. Keep the branch as-is (I'll handle it later)
4. Discard this work

Which option?
```

For detached HEAD, omit local merge:

```
Implementation complete. You're on a detached HEAD.

1. Push as a new branch and create a Pull Request
2. Keep as-is (I'll handle it later)
3. Discard this work

Which option?
```

### Step 6: Execute Choice

#### Merge Locally

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git checkout <base-branch>
git pull --ff-only
git merge <feature-branch>
<run implementation tests>
```

Only after merge and tests succeed, clean up an owned worktree and offer branch
deletion. Never delete a consolidation backup.

#### Push and Create PR

Show the proposed PR title and body, then:

```bash
git push -u origin <feature-branch>
gh pr create --base <base-branch> --head <feature-branch>
```

Do not clean up the worktree; it may be needed for PR feedback.

#### Push Current Target Branch

```bash
git push
```

#### Keep As-Is

Report the branch name and worktree path. Do not clean up.

#### Discard

**Confirm first:**
```
This will permanently delete:
- Branch <name>
- All commits: <commit-list>
- Worktree at <path>

Type 'discard' to confirm.
```

Wait for exact confirmation. Never delete a consolidation backup.

### Step 7: Cleanup Workspace

Only clean up after a successful local merge or confirmed discard.

```bash
GIT_DIR=$(cd "$(git rev-parse --git-dir)" 2>/dev/null && pwd -P)
GIT_COMMON=$(cd "$(git rev-parse --git-common-dir)" 2>/dev/null && pwd -P)
WORKTREE_PATH=$(git rev-parse --show-toplevel)
```

If `GIT_DIR == GIT_COMMON`, there is no worktree to remove.

If the worktree path is under `.worktrees/` or `worktrees/`, move to the main
repository root before cleanup:

```bash
MAIN_ROOT=$(git -C "$(git rev-parse --git-common-dir)/.." rev-parse --show-toplevel)
cd "$MAIN_ROOT"
git worktree remove "$WORKTREE_PATH"
git worktree prune
```

Otherwise, the host environment owns the workspace. Do not remove it.

At completion, report the final branch and `HEAD`, tests run, selected
workflow, and retained backup branch. If a backup exists, show the optional
manual cleanup command but do not run it:

```bash
git branch -D <backup-branch>
```

## Red Flags

**Never:**
- Proceed with failing tests
- Rewrite commits that cannot be proved unpublished
- Rewrite before the user approves the exact result
- Rewrite without a verified backup
- Delete the backup automatically
- Continue after failed rewrite verification
- Merge without verifying tests on the result
- Delete work without confirmation
- Force-push without explicit request
- Clean up worktrees you didn't create

**Always:**
- Confirm an inferred implementation boundary
- Analyze commits before presenting dynamic consolidation options
- Preserve final content when rewriting history
- Detect the workspace before integration or cleanup
- Require typed `discard` confirmation
