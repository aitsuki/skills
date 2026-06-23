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

If creation or verification fails, stop before rewriting. Never delete the backup automatically.

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
