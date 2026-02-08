---
name: plan-to-action
description: Parse a development plan into TodoWrite tasks, then spawn CC subagent swarms to execute them. Replaces workflows:work.
---

# plan-to-action

**Purpose:** Turn a development plan into executable tasks, then use Claude Code's native subagent system to farm work out to parallel workers. The orchestrator stays lean.

## When to Use

- You have a plan file with phases/checkboxes
- The work is parallelizable (multiple independent features, files, or phases)
- You want subagent-level isolation per task

## Input

**If no plan document reference is provided**, ask: "Which plan file should I execute? Provide a path."

## Precondition: Dangerously Skip Permissions

**BLOCKING CHECK — do this before anything else.**

Ask the user: "Do you have `--dangerously-skip-permissions` enabled for this session?"

- If **yes**: proceed.
- If **no** or **unsure**: **STOP.** Tell the user:
  ```
  This skill spawns autonomous worker subagents that need to write files,
  run tests, and commit without permission prompts. It will not work
  without --dangerously-skip-permissions.

  Please quit and restart Claude Code with:
    claude --dangerously-skip-permissions
  ```
  Do NOT proceed. Wait for the user to restart and re-invoke.

## Workflow

### Phase 1: Parse Plan -> Tasks

1. Read the plan file completely
2. Review any references or links in the plan
3. If anything is unclear, ask clarifying questions now
4. Break into **atomic, independent tasks** using `TodoWrite`:
   - Each task = one logical unit (a file, a feature, a test suite)
   - Tag dependencies: if task B depends on task A, note it
   - Include acceptance criteria from the plan in each task description
   - Include relevant file references and pattern examples
5. Consider writing tests that fail _before_ writing a feature, then running the test again after writing the feature to ensure it succeeds.
5. Present task list to user for approval before spawning

**Task format in TodoWrite:**
```
[id: 1] feat: Add user model and migration
  deps: none
  files: src/models/user.ts, src/db/migrations/001_create_users.ts
  test: tests/models/user.test.ts
  accept: User model exists with email, name fields; migration runs clean

[id: 2] feat: Add authentication service
  deps: 1
  files: src/services/auth.ts
  test: tests/services/auth.test.ts
  accept: Login/logout works, returns JWT
```

### Phase 2: Spawn Worker Swarm

Identify groups of tasks that would benefit from being implemented by an agent swarm and spawn agent swarms to accomplish them.  As agents complete tasks, you should mark the corresponding task as complete.

Other tasks that can be completed independently might be spawned as subagents.

Agents spawned in either of these ways should be reminded that solutions to many problems can be found in ./docs/solutions/

The manager task should do very little work at all except to orchestrate the other agents.

**Key mechanics:**
- Workers can read the codebase but only modify their assigned scope
- workers should not ask for input. they should run completely autonomously.

### Phase 3: Orchestrator Loop

The orchestrator (main CC session) manages the swarm:

```
while tasks remain:
  1. Identify ready tasks (all deps marked complete)
  2. Spawn Task(plan-worker) for each ready task
  3. As workers return:
     - DONE -> mark complete in TodoWrite, check off plan checkbox
     - BLOCKED -> log reason, retry once, then escalate to user
  4. When wave completes, unlock next wave of dependent tasks
```

### Phase 4: Converge

1. All tasks complete -> run full test suite from orchestrator
2. Run linting (per CLAUDE.md)
3. If anything broke from parallel work (merge conflicts, integration issues):
   - Small fixes: handle in orchestrator context
   - Big fixes: spawn a fix-up planner agent.
4. Update plan file: all checkboxes marked `[x]`
5. Present summary to user:

```
plan-to-action complete

Tasks: 8/8 completed
Commits: 6 (2 tasks shared a commit)
Tests: 47 passing

Failed/retried: 1 (auth service - fixed on retry)

Next:
1. Review changes
2. Push to remote
```

Subagents/swawm agents should have reported solutions they encountered.  The orchestrator may now compound these solutions into ./docs/solutions using the `compound-docs` skill

## Worker Memory

Instruct code agents to report any solutions to unexpected problems back to the orchestrator so that it can use the compound-docs skill to compound those learnings into ./docs/solutions/

- Codebase patterns discovered
- Common gotchas and conventions
- What worked and what did not

This **compounds** -- future plan-to-action runs benefit from prior worker learnings.

## Quality Gates

### Per Worker (before reporting DONE)
- [ ] Tests pass for their scope
- [ ] Lint passes on changed files
- [ ] Follows existing patterns (grep for similar code first)
- [ ] Commit message in conventional format
- [ ] Acceptance criteria from task met

### Orchestrator (at convergence)
- [ ] Full test suite passes
- [ ] No integration conflicts between worker outputs
- [ ] All plan checkboxes marked complete
- [ ] All TodoWrite tasks marked completed

## Integration

**Input from:** A plan markdown file.

**Works with:**
- `./docs/solutions`
- `compound-docs skill`
- `create-agent-skills` -- if work produces reusable patterns

## Common Pitfalls

- **Too-large tasks** -- If a worker needs >1 file change across unrelated areas, split the task
- **Missing pattern references** -- Always include "see X file for conventions" in task descriptions
- **Skipping approval** -- Always show task list before spawning. Better to adjust now than re-run workers
- **Ignoring BLOCKED** -- Do not auto-retry more than once. Escalate to user.
