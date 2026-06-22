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

Before starting implementation, report the current branch and working tree status.

### Step 2: Execute Tasks

For each task:
1. Mark as in_progress
2. Follow each step exactly (plan has bite-sized steps)
3. Run verifications as specified
4. Mark as completed

### Step 3: Complete Development

After all tasks complete and verified:
1. Run the plan-specified final verification if present; otherwise run the relevant full test suite and report that no final verification was specified
2. Check `git status`
3. Summarize changes, tests, and remaining risks
4. Report when complete

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
- Report current branch and working tree status before implementation
- Follow plan steps exactly
- Don't skip verifications
- Reference available skills when plan says to
- Stop when blocked, don't guess
