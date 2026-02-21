# Agent Teams Development Skill Design

**Date:** 2026-02-21
**Status:** Design Complete, Awaiting Implementation

## Overview

Add Claude Code's native agent teams feature as a third execution option in Superpowers. When a plan is ready to execute, the user is offered agent teams alongside the existing subagent-driven and parallel-session approaches. The new skill analyzes the plan, proposes a team composition, lets the user accept or adjust it, then hands off to Claude Code's native agent teams feature.

## Background

Claude Code's experimental agent teams feature lets multiple persistent Claude Code sessions work together as a team. Unlike subagents (which only report results back to the lead), team members share a task list, communicate peer-to-peer via SendMessage, and coordinate directly with each other. This makes agent teams better suited for work that spans independent domains that benefit from parallel execution and cross-agent collaboration.

The feature requires `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` in settings or environment.

## What Changes

### 1. `skills/writing-plans/SKILL.md` — Execution Handoff section

The current two-option handoff becomes three options:

```
Plan complete and saved to `docs/plans/<filename>.md`. Three execution options:

1. Subagent-Driven (this session) — fresh subagent per task, spec+quality review between tasks
2. Parallel Session (separate) — open new session with executing-plans, batch execution with checkpoints
3. Agent Teams (parallel, collaborative) — teammates own domains and work concurrently with
   peer-to-peer communication. Requires CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1.

Which approach?
```

If agent teams is chosen → invoke `superpowers:agent-teams-development`.

### 2. `skills/agent-teams-development/SKILL.md` — new skill

## New Skill Design

### When to Use

Agent teams are best when the implementation plan has tasks that fall into independent domain clusters — each cluster can be worked on in parallel without waiting on the others, and where teammates may need to coordinate on shared interfaces or contracts (e.g. frontend and backend agreeing on an API shape).

Agent teams add coordination overhead and use significantly more tokens than a single session. For small plans with tightly coupled tasks, subagent-driven development is more efficient.

### Phase 1: Prerequisites

On invocation the skill:

1. Announces: "Using agent-teams-development skill to set up an agent team for this plan."
2. Verifies a git worktree is active (same requirement as subagent-driven-development).
3. Reminds the user that `CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS=1` must be set in `settings.json` or environment before proceeding.

### Phase 2: Team Proposal

1. Read the implementation plan and extract all tasks.
2. Identify independent domain clusters — groups of tasks that can proceed in parallel without blocking each other.
3. Propose one teammate per domain cluster. Each teammate gets:
   - A role label describing their domain (e.g. "Backend/API", "Frontend", "Tests & Integration")
   - The list of tasks assigned to them
   - Any dependencies on other teammates' work called out explicitly
4. Present the proposal to the user. Example:

```
Proposed team for "Payment System":

  Teammate 1 (Backend/API)        — Tasks 1-3: payment gateway, transaction model, webhook handler
  Teammate 2 (Frontend)           — Tasks 4-5: checkout UI, payment form
  Teammate 3 (Tests/Integration)  — Task 6: integration tests (starts after Teammate 1 finishes)

Accept this team, or describe how you'd like it changed?
```

5. Iterate until the user explicitly accepts. The user can say anything: "merge teammates 2 and 3", "add a security reviewer", "split task 3 off into its own teammate", etc.

The skill does **not** create any team or write any code until the user approves.

### Phase 3: Kickoff

Once approved, craft a natural language kickoff prompt that describes:

- The feature goal and architecture (from the plan header)
- Each teammate's role and assigned tasks with enough context to work independently
- Task dependencies between teammates
- Quality requirements: require plan approval before each teammate begins implementing

Hand off to Claude Code's native agent teams by issuing the kickoff prompt. The skill's job ends here. Claude Code handles team creation, spawning teammates, the shared task list, peer-to-peer messaging, and all coordination.

Example kickoff prompt:

```
Create an agent team to implement the payment system. Spawn three teammates:

- Teammate "Backend" (Tasks 1-3): implement the payment gateway integration, transaction
  model, and webhook handler. Follow TDD. Commit after each task.
- Teammate "Frontend" (Tasks 4-5): implement the checkout UI and payment form. Follow TDD.
  Commit after each task. Coordinate with Backend on the API contract.
- Teammate "Tests" (Task 6): implement integration tests and error scenarios. Wait for
  Backend tasks to complete before starting. Coordinate with both other teammates.

Require plan approval before each teammate begins implementing.
```

### What the Skill Does NOT Do

- Manually call TeamCreate, Task, SendMessage, TaskCreate, or any other agent team tools
- Orchestrate the team after kickoff
- Dispatch subagent reviewers
- Monitor task completion

All of that is handled by Claude Code's native agent teams feature.

## Skill File Location

```
skills/agent-teams-development/SKILL.md
```

No subdirectory prompt templates needed — unlike subagent-driven-development, this skill does not dispatch its own subagents.

## Integration

The skill fits into the existing Superpowers workflow as a peer execution option:

```
brainstorming → writing-plans → [choose execution]
                                  ├── subagent-driven-development  (in-session, sequential)
                                  ├── executing-plans              (parallel session)
                                  └── agent-teams-development      (native agent teams)  ← NEW
```

After agent teams completes, the lead invokes `superpowers:finishing-a-development-branch` as usual.
