# Agent Teams Development — Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add Claude Code's native agent teams feature as a third execution option in Superpowers, offered alongside subagent-driven-development and executing-plans after a plan is written.

**Architecture:** Two files change. `skills/writing-plans/SKILL.md` gains a third option in its Execution Handoff section. A new `skills/agent-teams-development/SKILL.md` is created with the full skill: analyze plan → propose team → get user approval → craft kickoff prompt → hand off to native agent teams. The skill does not manually orchestrate agents; it delegates entirely to Claude Code's native feature.

**Tech Stack:** Markdown skill files with YAML frontmatter. No code. No dependencies.

---

### Task 1: Update `skills/writing-plans/SKILL.md` — Execution Handoff

**Files:**
- Modify: `skills/writing-plans/SKILL.md:97-116`

**Step 1: Replace the Execution Handoff section**

Open `skills/writing-plans/SKILL.md`. Replace the entire `## Execution Handoff` section (lines 97–116) with:

```markdown
## Execution Handoff

After saving the plan, offer execution choice:

**"Plan complete and saved to `docs/plans/<filename>.md`. Three execution options:**

**1. Subagent-Driven (this session)** - I dispatch fresh subagent per task, review between tasks, fast iteration

**2. Parallel Session (separate)** - Open new session with executing-plans, batch execution with checkpoints

**3. Agent Teams (parallel, collaborative)** - Teammates own domains and work concurrently with peer-to-peer communication. Requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`.

**Which approach?"**

**If Subagent-Driven chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:subagent-driven-development
- Stay in this session
- Fresh subagent per task + code review

**If Parallel Session chosen:**
- Guide them to open new session in worktree
- **REQUIRED SUB-SKILL:** New session uses superpowers:executing-plans

**If Agent Teams chosen:**
- **REQUIRED SUB-SKILL:** Use superpowers:agent-teams-development
```

**Step 2: Verify the change**

Read `skills/writing-plans/SKILL.md` and confirm:
- The heading says "Three execution options"
- Option 3 mentions agent teams and `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1`
- The "If Agent Teams chosen" block is present with `superpowers:agent-teams-development`
- Options 1 and 2 are unchanged

**Step 3: Commit**

```bash
git add skills/writing-plans/SKILL.md
git commit -m "feat: add agent teams as third execution option in writing-plans"
```

---

### Task 2: Create `skills/agent-teams-development/SKILL.md`

**Files:**
- Create: `skills/agent-teams-development/SKILL.md`

**Step 1: Create the directory and skill file**

Create `skills/agent-teams-development/SKILL.md` with this exact content:

````markdown
---
name: agent-teams-development
description: Use when executing an implementation plan using Claude Code's native agent teams — propose a team composition, get user approval, then kick off the native agent teams feature
---

# Agent Teams Development

Analyze an implementation plan, propose a team of domain-specialist teammates, get user approval, then hand off to Claude Code's native agent teams feature.

**Announce at start:** "Using agent-teams-development skill to propose an agent team for this plan."

## When to Use

Best when the plan has tasks across independent domains that benefit from parallel execution — e.g. frontend, backend, and tests that can proceed without waiting on each other. Teammates communicate peer-to-peer during implementation, which helps when domains share interfaces or contracts.

Not ideal for small plans or tightly coupled tasks — subagent-driven-development is more efficient there.

Agent teams use significantly more tokens than a single session. Each teammate is a separate Claude instance with its own context window.

## Prerequisites

Before proposing a team:

1. Verify a git worktree is active. If not, stop and use `superpowers:using-git-worktrees` first.
2. Remind the user that `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` must be set before proceeding:

```json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

Add this to `~/.claude/settings.json` or pass as an environment variable. Ask the user to confirm it is set before continuing.

## Phase 1: Propose a Team

1. Read the implementation plan. Extract all tasks and the plan header (goal, architecture, tech stack).
2. Identify independent domain clusters — groups of tasks that can proceed in parallel without blocking each other.
3. Assign one teammate per domain cluster. Each teammate gets:
   - A role label describing their domain (e.g. "Backend/API", "Frontend", "Tests")
   - The tasks assigned to them by number and brief description
   - Any dependency on another teammate's work called out explicitly

Present the proposal clearly:

```
Proposed team for "[Feature Name]":

  Teammate 1 ([Role])  — Tasks 1-3: [brief description]
  Teammate 2 ([Role])  — Tasks 4-5: [brief description]
  Teammate 3 ([Role])  — Task 6: [brief description] (starts after Teammate 1 finishes)

Accept this team, or describe how you'd like it changed?
```

4. Iterate until the user explicitly accepts. Revise and re-propose for any requested changes — merging teammates, splitting tasks off, adding a specialist, adjusting dependencies, etc.

**Do NOT proceed to Phase 2 until the user explicitly accepts the proposed team.**

## Phase 2: Kick Off

Once the user approves, craft a natural language kickoff prompt that includes:

- The feature goal and architecture context (from the plan header)
- Each teammate's role name, their assigned tasks with enough full-text context to work independently, TDD and commit instructions
- Dependencies between teammates stated explicitly
- "Require plan approval before each teammate begins implementing"

Then issue the kickoff prompt to start the native agent teams feature. Example:

```
Create an agent team to implement [feature name]. [Goal sentence from plan header.]

Spawn [N] teammates:

- Teammate "[Role 1]" (Tasks 1-3): [full task context from plan]. Follow TDD. Commit after each task.
- Teammate "[Role 2]" (Tasks 4-5): [full task context from plan]. Follow TDD. Commit after each task.
  Coordinate with "[Role 1]" on the API contract.
- Teammate "[Role 3]" (Task 6): [full task context from plan].
  Wait for "[Role 1]" tasks to complete before starting.

Require plan approval before each teammate begins implementing.
```

The skill's job ends here. Claude Code's native agent teams feature handles all team creation, teammate spawning, the shared task list, peer-to-peer communication, and coordination.

## What This Skill Does NOT Do

- Call TeamCreate, Task (with team_name), SendMessage, TaskCreate, or any agent team tools directly
- Orchestrate or monitor the team after kickoff
- Dispatch reviewers or track task completion

## After Teams Complete

When all teammates have finished and the team is cleaned up, invoke `superpowers:finishing-a-development-branch`.
````

**Step 2: Verify the file**

Read `skills/agent-teams-development/SKILL.md` and confirm:
- YAML frontmatter has correct `name` and `description`
- Prerequisites section mentions the env var and asks user to confirm it is set
- Phase 1 has the team proposal format and the hard gate ("Do NOT proceed until user explicitly accepts")
- Phase 2 has the kickoff prompt example
- "What This Skill Does NOT Do" section is present
- "After Teams Complete" mentions `superpowers:finishing-a-development-branch`

**Step 3: Commit**

```bash
git add skills/agent-teams-development/SKILL.md
git commit -m "feat: add agent-teams-development skill"
```

---

### Task 3: Add explicit skill request test prompt

**Files:**
- Create: `tests/explicit-skill-requests/prompts/agent-teams-development-please.txt`

**Step 1: Create the test prompt file**

Create `tests/explicit-skill-requests/prompts/agent-teams-development-please.txt` with this content:

```
agent-teams-development, please
```

**Step 2: Verify the file**

Read `tests/explicit-skill-requests/prompts/agent-teams-development-please.txt` and confirm it contains `agent-teams-development, please`.

**Step 3: Commit**

```bash
git add tests/explicit-skill-requests/prompts/agent-teams-development-please.txt
git commit -m "test: add explicit skill request prompt for agent-teams-development"
```

---

### Task 4: Add skill-triggering test prompt

**Files:**
- Create: `tests/skill-triggering/prompts/agent-teams-development.txt`

**Step 1: Create the test prompt file**

Create `tests/skill-triggering/prompts/agent-teams-development.txt` with this content:

```
I have a finished implementation plan with frontend and backend tasks. Let's use agent teams to execute it in parallel.
```

**Step 2: Verify the file**

Read `tests/skill-triggering/prompts/agent-teams-development.txt` and confirm it contains the prompt text above.

**Step 3: Commit**

```bash
git add tests/skill-triggering/prompts/agent-teams-development.txt
git commit -m "test: add skill-triggering prompt for agent-teams-development"
```
