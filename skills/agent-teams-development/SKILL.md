---
name: agent-teams-development
description: Use when executing an implementation plan using Claude Code's native agent teams feature — tasks span independent domains that benefit from parallel execution by domain-specialist teammates
---

# Agent Teams Development

Analyze an implementation plan, propose a team of domain-specialist teammates, get user approval, then hand off to Claude Code's native agent teams feature.

**Announce at start:** "Using agent-teams-development skill to propose an agent team composition and kick off implementation."

## When to Use

Best when the plan has tasks across independent domains that benefit from parallel execution — e.g. frontend, backend, and tests that can proceed without waiting on each other. Teammates communicate peer-to-peer during implementation, which helps when domains share interfaces or contracts.

Not ideal for small plans or tightly coupled tasks — subagent-driven-development is more efficient there.

Agent teams use significantly more tokens than a single session. Each teammate is a separate Claude instance with its own context window.

<HARD-GATE>
Do NOT kick off the agent team or issue any kickoff prompt until the user has explicitly accepted the proposed team composition. This applies regardless of how clear the task breakdown seems.
</HARD-GATE>

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

Add this to `~/.claude/settings.json` or pass as an environment variable. Ask the user to confirm it is set before continuing. If it is not set, stop and direct the user to add it — do not attempt to set it automatically.

Do NOT proceed to Phase 1 until both prerequisites are confirmed.

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

## Red Flags

**Never:**

- Call TeamCreate, Task (with team_name), SendMessage, TaskCreate, or any agent team tools directly
- Orchestrate or monitor the team after kickoff
- Dispatch reviewers or track task completion

## After Teams Complete

Add the following to the end of the kickoff prompt so the team lead handles this when work is done:

```
When all teammates have finished and the team is cleaned up, invoke `superpowers:finishing-a-development-branch`.
```

## Integration

**Required before this skill:**
- **superpowers:using-git-worktrees** — REQUIRED: active worktree must exist before starting
- **superpowers:writing-plans** — creates the implementation plan this skill executes

**Invoked by:**
- **superpowers:writing-plans** — offers this skill as the "Agent Teams" execution option

**After teams complete:**
- **superpowers:finishing-a-development-branch** — included in kickoff prompt for team lead to invoke when done
