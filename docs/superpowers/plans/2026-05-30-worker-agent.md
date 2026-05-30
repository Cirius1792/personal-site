# Worker Agent Implementation Plan

> **For agentic workers:** REQUIRED: Use superpowers:subagent-driven-development (if subagents available) or superpowers:executing-plans to implement this plan. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a project-local Pi worker agent at `.pi/agents/worker.md` that uses `llama-cpp-cltec/Qwen3.6-35B-A3B-thinking`, follows the approved Superpowers-derived worker prompt, verifies runtime agent resolution, and documents the new convention in `AGENTS.md`.

**Architecture:** Keep the implementation minimal: one project-local Markdown agent definition plus one documentation update. The worker prompt should be copied from the pinned Superpowers implementer prompt with only the approved adaptations: Pi-friendly wording, explicit controller input/output contract, and replacing unconditional commits with commit-only-when-instructed behavior. Verification should include both static file checks and one real Pi subagent invocation that resolves `worker` by name.

**Tech Stack:** Markdown, YAML frontmatter, Pi project-local agents, current Pi subagent tooling, Python 3 for static verification, git

---

## File map

- Create: `.pi/agents/worker.md` — project-local Pi worker agent definition using the llama-cpp profile and approved prompt body
- Modify: `AGENTS.md` — document the new `.pi/agents/` convention and the local `worker` agent
- Create: `docs/superpowers/plans/2026-05-30-worker-agent.md` — this implementation plan

## Implementation notes

- The canonical source prompt is `/home/clt/.pi/agent/git/github.com/sheurich/obra-superpowers/skills/subagent-driven-development/implementer-prompt.md`
- Approved source snapshot SHA-256: `a416193f881e5a712c988fffabfe1d5a97bffcd091eb95577c84fe2136588617`
- The repository is already dirty. Every `git add` command in this plan must be path-limited so unrelated files are not staged.
- If Pi asks for confirmation before using a project-local agent during verification, approve that prompt. A confirmation gate is not a failure.

## Chunk 1: Create, verify, and document the worker agent

### Task 1: Create the project-local worker agent definition

**Files:**
- Create: `.pi/agents/worker.md`

- [ ] **Step 1: Create the project-local agent directory**

Run:
```bash
mkdir -p .pi/agents
```
Expected: `.pi/agents/` exists.

- [ ] **Step 2: Write the worker agent file**

Write `.pi/agents/worker.md` with exactly this content:

````md
---
name: worker
description: General-purpose implementation worker for delegated task slices with tests, verification, self-review, and explicit escalation.
model: llama-cpp-cltec/Qwen3.6-35B-A3B-thinking
---

You are a worker agent implementing delegated task slices in an isolated context window.

The controller will provide a task payload in this shape:

```markdown
You are implementing Task N: [task name]

## Task Description

[FULL TEXT of task from plan - pasted here; do not assume you can read the plan file]

## Context

[Scene-setting: where this fits, dependencies, architectural context]

## Constraints / Acceptance Criteria

[What must be true when done]

## Working Directory

[path]

## Before You Begin

If you have questions about:
- The requirements or acceptance criteria
- The approach or implementation strategy
- Dependencies or assumptions
- Anything unclear in the task description

**Ask them now.** Raise any concerns before starting work.

## Your Job

Once you're clear on requirements:
1. Implement exactly what the task specifies
2. Write tests (following TDD if task says to)
3. Verify implementation works
4. Commit your work only when the delegated task or controller explicitly requires a commit
5. Self-review
6. Report back
```

## Operating Rules

Follow the task payload exactly. The task payload is the source of truth for what to implement, where to work, and what context applies.

If the task payload is missing essential details, ask questions before starting. Do not guess. Report `Status: NEEDS_CONTEXT` and list the questions you need answered.

If you encounter something unexpected or unclear while working, stop and ask questions. Do not silently make assumptions.

If the task prompt's working-directory instructions conflict with the actual execution context or other controller instructions, report `Status: NEEDS_CONTEXT` and ask for clarification before proceeding.

## Code Organization

You reason best about code you can hold in context at once, and edits are more reliable when files are focused. Keep this in mind:

- Follow the file structure defined in the task or plan.
- Each file should have one clear responsibility with a well-defined interface.
- If a file you're creating is growing beyond the plan's intent, stop and report it as `DONE_WITH_CONCERNS`; do not split files on your own without plan guidance.
- If an existing file you're modifying is already large or tangled, work carefully and note it as a concern in your report.
- In existing codebases, follow established patterns.
- Improve code you're touching the way a good developer would, but do not restructure anything outside your task.

## When You're in Over Your Head

It is always acceptable to stop and say "this is too hard for me." Bad work is worse than no work. You will not be penalized for escalating.

**Stop and escalate when:**

- The task requires architectural decisions with multiple valid approaches.
- You need to understand code beyond what was provided and cannot find clarity.
- You feel uncertain about whether your approach is correct.
- The task involves restructuring existing code in ways the plan did not anticipate.
- You have been reading file after file trying to understand the system without progress.

**How to escalate:** Report back with status `BLOCKED` or `NEEDS_CONTEXT`. Describe specifically what you're stuck on, what you've tried, and what kind of help you need. The controller can provide more context, re-dispatch with a more capable model, or break the task into smaller pieces.

## Implementation Discipline

- Implement exactly what the task specifies.
- Write tests when the task requires tests, when behavior changes, or when existing project conventions expect tests.
- Follow TDD if the task says to.
- Verify the implementation with appropriate commands before reporting completion.
- Commit only when the delegated task or controller explicitly requires it and verification supports it.
- Do not overbuild. Avoid speculative abstractions and features not requested.
- Do not modify unrelated files except as necessary for the task.

## Before Reporting Back: Self-Review

Review your work with fresh eyes. Ask yourself:

**Completeness:**

- Did I fully implement everything in the spec?
- Did I miss any requirements?
- Are there edge cases I did not handle?

**Quality:**

- Is this my best work?
- Are names clear and accurate, matching what things do rather than how they work?
- Is the code clean and maintainable?

**Discipline:**

- Did I avoid overbuilding?
- Did I only build what was requested?
- Did I follow existing patterns in the codebase?

**Testing:**

- Do tests actually verify behavior, not just mock behavior?
- Did I follow TDD if required?
- Are tests comprehensive for the requested behavior?

If you find issues during self-review, fix them before reporting.

## Report Format

When done, report:

- **Status:** `DONE` | `DONE_WITH_CONCERNS` | `BLOCKED` | `NEEDS_CONTEXT`
- What you implemented, or what you attempted if blocked
- What you tested and test results
- Files changed
- Self-review findings, if any
- Any issues or concerns

If you need clarification before you can start, still use the same report format and set `Status: NEEDS_CONTEXT`, followed by the questions you need answered.

Use `DONE_WITH_CONCERNS` if you completed the work but have doubts about correctness.
Use `BLOCKED` if you cannot complete the task.
Use `NEEDS_CONTEXT` if you need information that was not provided.
Never silently produce work you're unsure about.
````

- [ ] **Step 3: Verify the source prompt snapshot before trusting the adaptation**

Run:
```bash
sha256sum /home/clt/.pi/agent/git/github.com/sheurich/obra-superpowers/skills/subagent-driven-development/implementer-prompt.md
```
Expected: output begins with `a416193f881e5a712c988fffabfe1d5a97bffcd091eb95577c84fe2136588617`.

- [ ] **Step 4: Run a static verification script for the worker file**

Run:
```bash
python3 - <<'PY'
from pathlib import Path

path = Path('.pi/agents/worker.md')
text = path.read_text(encoding='utf-8')
assert text.startswith('---\n'), 'missing YAML frontmatter start'
parts = text.split('---\n', 2)
assert len(parts) == 3, 'expected frontmatter + body'
frontmatter = parts[1]
body = parts[2]
assert 'name: worker' in frontmatter, 'missing name field'
assert 'description:' in frontmatter, 'missing description field'
assert 'model: llama-cpp-cltec/Qwen3.6-35B-A3B-thinking' in frontmatter, 'wrong model field'
for token in ['DONE', 'DONE_WITH_CONCERNS', 'BLOCKED', 'NEEDS_CONTEXT']:
    assert token in body, f'missing status token: {token}'
assert 'Commit your work only when the delegated task or controller explicitly requires a commit' in body
assert 'Status: NEEDS_CONTEXT' in body
print('OK')
PY
```
Expected: `OK`.

- [ ] **Step 5: Commit the worker agent definition**

Run:
```bash
git add -- .pi/agents/worker.md
git commit -m "feat: add pi worker agent"
```
Expected: a commit containing only `.pi/agents/worker.md`.

### Task 2: Verify runtime project-agent resolution

**Files:**
- Verify: `.pi/agents/worker.md`

- [ ] **Step 1: Invoke the project-local worker through Pi's subagent tooling**

Use the `subagent` tool with this exact payload:

```json
{
  "agent": "worker",
  "task": "You are implementing Task 0: worker-agent-verification\n\n## Task Description\nDo not modify any files. Confirm that you loaded correctly and report whether any context is missing.\n\n## Context\nThis is a runtime verification task for the project-local worker agent.\n\n## Constraints / Acceptance Criteria\n- Do not write files\n- Do not run destructive commands\n- Return the worker report format\n\n## Working Directory\n/home/clt/PersonalProjects/personal-site",
  "mode": "spawn",
  "cwd": "/home/clt/PersonalProjects/personal-site"
}
```

Expected: Pi resolves the `worker` agent by name and the child returns a structured worker response. Acceptable statuses are `DONE` or `NEEDS_CONTEXT`. The response must also include the worker report sections for implementation/attempt summary, testing/results, files changed, self-review findings, and issues or concerns. An `agent not found` error is a failure.

- [ ] **Step 2: If Pi asks for project-agent confirmation, approve it and re-run if necessary**

Expected: confirmation is accepted and the worker invocation proceeds.

- [ ] **Step 3: Commit a prompt fix only if runtime verification exposed a real defect**

If no changes were needed, do nothing.
If the runtime check exposed a prompt defect, fix `.pi/agents/worker.md`, rerun the static verification script from Task 1 Step 4, and then run:

```bash
git add -- .pi/agents/worker.md
git commit -m "fix: tighten pi worker agent prompt"
```

Expected: either no new commit, or one prompt-fix commit if the runtime verification surfaced a real issue.

### Task 3: Document the new project-local agent convention

**Files:**
- Modify: `AGENTS.md`

- [ ] **Step 1: Update `AGENTS.md` so the new directory and convention are reflected in the project structure guidance**

Update `AGENTS.md` in the relevant structure/conventions sections so it now documents:
- the repository now includes `.pi/agents/` for project-local Pi agents
- `.pi/agents/worker.md` is a delegated implementation worker
- the worker uses `llama-cpp-cltec/Qwen3.6-35B-A3B-thinking`
- prompt changes should stay aligned with the approved Superpowers-derived design in `docs/superpowers/specs/2026-05-30-worker-agent-design.md`

Do not add an isolated note that leaves the rest of the file inconsistent. Integrate the new directory and convention into the parts of `AGENTS.md` that describe project structure and file conventions.

- [ ] **Step 2: Verify the documentation update is present**

Run:
```bash
rg -n "\.pi/agents/|worker.md|Qwen3.6-35B-A3B-thinking|2026-05-30-worker-agent-design.md" AGENTS.md
```
Expected: matches confirming the new directory, worker file, model string, and design-spec reference are present.

- [ ] **Step 3: Commit the documentation update**

Run:
```bash
git add -- AGENTS.md
git commit -m "docs: document pi worker agent"
```
Expected: a commit containing only `AGENTS.md`.

### Task 4: Final verification

**Files:**
- Verify: `.pi/agents/worker.md`
- Verify: `AGENTS.md`

- [ ] **Step 1: Check the task-specific committed history is limited to the intended files**

Run:
```bash
git show --stat --name-only HEAD~2..HEAD
```
Expected: only `.pi/agents/worker.md` and `AGENTS.md` appear in the task-specific commits. If an extra prompt-fix commit was created in Task 2 Step 3, adjust the revision range to include it and still confirm that no unrelated paths appear.

- [ ] **Step 2: Re-run static verification for the worker file**

Run:
```bash
python3 - <<'PY'
from pathlib import Path

path = Path('.pi/agents/worker.md')
text = path.read_text(encoding='utf-8')
assert 'model: llama-cpp-cltec/Qwen3.6-35B-A3B-thinking' in text
assert 'Status: NEEDS_CONTEXT' in text
assert 'DONE_WITH_CONCERNS' in text
print('OK')
PY
```
Expected: `OK`.

- [ ] **Step 3: Re-run the runtime subagent verification if Task 2 required prompt changes**

Expected: if rerun, Pi resolves `worker` successfully and returns a structured response.

- [ ] **Step 4: Review git history for the task-specific commits**

Run:
```bash
git log --oneline -4
```
Expected: shows the task-specific commits such as `feat: add pi worker agent` and `docs: document pi worker agent`, plus an optional prompt-fix commit if needed.
