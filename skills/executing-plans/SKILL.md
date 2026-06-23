---
name: executing-plans
description: Use when you have a written implementation plan to execute task-by-task, either inline or in a fresh session.
---

# Executing Plans

## Overview

Load plan, review critically, execute all tasks, report when complete.

**Announce at start:** "I'm using the executing-plans skill to implement this plan."

## The Process

### Step 1: Load and Review Plan
1. Read plan file
2. Review critically - identify any questions or concerns about the plan
3. If concerns: Raise them with your human partner before starting
4. If no concerns: Create todos for the plan items and proceed

Before starting implementation, report the current branch and working tree
status. Confirm the spec and plan commits are already in history, then run:

```bash
git rev-parse HEAD
```

Record the result as `implementation-base` immediately before the first
implementation task. Do not move this boundary later. The implementation range
passed to the commit-finalization skill is `implementation-base..HEAD`, keeping the
committed spec and plan outside the rewrite range.

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

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

## When to Stop and Ask for Help

**STOP executing immediately when:**
- Hit a blocker (missing dependency, test fails, instruction unclear)
- Plan has critical gaps preventing starting
- You don't understand an instruction
- Verification fails repeatedly

**Ask for clarification rather than guessing.**

## When to Revisit Earlier Steps

**Return to Review (Step 1) when:**
- Partner updates the plan based on your feedback
- Fundamental approach needs rethinking

**Don't force through blockers** - stop and ask.

## Remember
- Review plan critically first
- Follow plan steps exactly
- Don't skip verifications
- Reference available skills when plan says to
- Stop when blocked, don't guess
- Record `implementation-base` immediately before implementation
- Leave implementation commit rewriting to `$finalizing-implementation-commits`
- Treat the finalizing skill's report as the terminal state; never push or continue into branch disposition
