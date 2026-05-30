# Worker Agent Design

## Goal

Add a project-local Pi worker agent that uses the llama-cpp model profile already configured in Pi and behaves like the Superpowers implementer/worker prompt.

## Background

In the current Pi environment, project-local agent files are loaded from `.pi/agents/*.md` by the available subagent-capable harness/tooling. This design assumes that capability is already present in the user's Pi setup; enabling or installing that capability is out of scope.

Agent files are Markdown documents with YAML frontmatter plus a prompt body.

The existing Pi model configuration already defines this provider/model profile:

- `llama-cpp-cltec/Qwen3.6-35B-A3B-thinking`

The requested worker should be local to this repository, not a global user agent.

## Scope

This design covers only the worker agent definition file.

In scope:
- Create `.pi/agents/worker.md`
- Set the worker to use `llama-cpp-cltec/Qwen3.6-35B-A3B-thinking`
- Use a near-copy of the Superpowers implementer/worker prompt
- Keep the delegated-task, escalation, verification, and self-review protocol

Out of scope:
- New Pi extensions or tools
- Changes to `~/.pi/agent/models.json`
- A controller/orchestrator agent
- Automatic task decomposition or plan execution infrastructure

## Requirements

The worker agent must:

1. Live at `.pi/agents/worker.md`
2. Be discoverable by Pi as a project-local agent
3. Use YAML frontmatter with at least:
   - `name: worker`
   - a clear `description`
   - `model: llama-cpp-cltec/Qwen3.6-35B-A3B-thinking`
4. Use `/home/clt/.pi/agent/git/github.com/sheurich/obra-superpowers/skills/subagent-driven-development/implementer-prompt.md` as the canonical source prompt snapshot, pinned to SHA-256 `a416193f881e5a712c988fffabfe1d5a97bffcd091eb95577c84fe2136588617`
5. Preserve the following from that source prompt:
   - delegated-task framing
   - implementation discipline
   - stop-and-escalate rules
   - self-review checklist
   - status vocabulary: `DONE`, `DONE_WITH_CONCERNS`, `BLOCKED`, `NEEDS_CONTEXT`
   - final report structure
6. Allow only minimal edits needed to make the prompt read naturally as a Pi project agent
7. Apply one explicit behavioral adaptation from the source prompt: replace the unconditional `Commit your work` instruction with `Commit your work only when the delegated task or controller explicitly requires a commit`
8. Expect the invoking controller to paste the full delegated task into the task prompt
9. When essential context is missing, respond with `Status: NEEDS_CONTEXT` and ask the clarifying questions needed to proceed
10. Return one of these statuses in every final or blocking report:
   - `DONE`
   - `DONE_WITH_CONCERNS`
   - `BLOCKED`
   - `NEEDS_CONTEXT`

## Proposed Approach

Create a single Markdown agent definition at `.pi/agents/worker.md`.

The frontmatter will declare the agent identity and the llama-cpp model profile. The body will be a near-copy of `/home/clt/.pi/agent/git/github.com/sheurich/obra-superpowers/skills/subagent-driven-development/implementer-prompt.md` as pinned by SHA-256 `a416193f881e5a712c988fffabfe1d5a97bffcd091eb95577c84fe2136588617`, adapted only where necessary to make the instructions read naturally in Pi.

Allowed edits are intentionally narrow:
- remove wording that assumes a non-Pi harness or a specific external task-dispatch tool
- rephrase references to the incoming task so they work as a Pi agent prompt
- replace the unconditional commit instruction with an explicit "commit only when instructed" rule
- preserve all other behavioral sections, status values, escalation rules, and the self-review/reporting structure

The prompt will remain self-contained so any controller can dispatch work by including the full task payload directly in the task prompt. The worker will not depend on external plan files being re-read unless the controller explicitly asks it to do so.

## Agent Structure

### Frontmatter

The file will define:

```yaml
---
name: worker
description: General-purpose implementation worker for delegated task slices with tests, verification, self-review, and explicit escalation.
model: llama-cpp-cltec/Qwen3.6-35B-A3B-thinking
---
```

No explicit `tools` field will be added in this first version. That avoids accidentally restricting the worker more than intended before real usage reveals a need for tighter controls.

### Prompt Body

The prompt body will preserve these Superpowers behaviors:

- delegated-task framing
- "ask questions before starting" behavior
- implementation discipline
- stop-and-escalate rules
- self-review checklist
- structured report format

The prompt will explicitly tell the worker that:

- the task payload is the source of truth
- missing essential details should trigger `NEEDS_CONTEXT`, not guessing
- unexpected ambiguity should trigger escalation
- verification should happen before claiming completion

### Controller Input Contract

Controllers should invoke the worker with a prompt that includes, at minimum:

- task name
- task description
- context
- constraints or acceptance criteria
- working directory

When the controller also sets Pi subagent `cwd`, that `cwd` is the source of truth. The prompt's `## Working Directory` section must match it and exists mainly to keep the worker's natural-language instructions self-contained. If the two disagree, the worker should treat that mismatch as `NEEDS_CONTEXT`.

Recommended shape:

```markdown
You are implementing Task N: [task name]

## Task Description
[full delegated task]

## Context
[architectural and dependency context]

## Constraints / Acceptance Criteria
[what must be true when done]

## Working Directory
[path]
```

If any of those essentials are missing and they prevent safe execution, the worker should stop and report `NEEDS_CONTEXT`.

### Worker Output Contract

The worker should preserve the Superpowers report structure and emit these sections:

- `Status:` `DONE` | `DONE_WITH_CONCERNS` | `BLOCKED` | `NEEDS_CONTEXT`
- what it implemented or attempted
- what it tested and the results
- files changed
- self-review findings, if any
- issues or concerns

When the worker needs clarification before it can proceed, it should still emit `Status: NEEDS_CONTEXT` and then list the specific clarifying questions.

## Invocation Model

The worker is a project-local Pi agent, not a new Pi tool.

Expected usage:

1. A controller invokes agent `worker`
2. The controller includes the delegated work item in the task prompt using the documented input contract
3. The worker executes according to `.pi/agents/worker.md`
4. The worker returns a structured status report using the documented output contract

This keeps the design minimal and compatible with the current Pi environment's project-agent discovery behavior.

## Error Handling

The worker must handle ambiguous or incomplete tasks conservatively.

### Missing information

If the delegated prompt omits acceptance criteria, constraints, working directory, or other essential context, the worker should respond with `Status: NEEDS_CONTEXT` and ask the specific questions required before starting.

### Unclear implementation path

If the worker can proceed only by making unapproved architectural choices, it should stop and return `BLOCKED` or `NEEDS_CONTEXT`.

### Low-confidence completion

If the worker finishes but still has correctness concerns, it must report `DONE_WITH_CONCERNS` instead of silently presenting the work as complete.

## Testing and Verification

Because this change is a single agent-definition file, verification is lightweight but must include one runtime discovery check:

1. Confirm `.pi/agents/worker.md` exists
2. Confirm the frontmatter parses as valid agent metadata
3. Confirm the model string exactly matches `llama-cpp-cltec/Qwen3.6-35B-A3B-thinking`
4. Confirm the prompt body preserves the canonical Superpowers status protocol and self-review structure, with the approved commit-behavior adaptation
5. Run one runtime discovery check using the current Pi subagent path, for example by invoking agent `worker` with a trivial self-contained task in `spawn` mode and repository `cwd`, and confirm Pi resolves the agent by name before execution
6. If project-agent confirmation is required by the runtime, approve that prompt during verification; failure to confirm is not an agent-definition failure

## Alternatives Considered

### 1. Literal copy of the Superpowers implementer prompt

Pros:
- Maximum fidelity to the source prompt

Cons:
- More likely to retain wording that assumes a different harness or task-dispatch style

### 2. Near-copy with Pi-native framing

Pros:
- Preserves intended behavior
- Fits Pi's project-local agent format cleanly
- Avoids unnecessary harness-specific wording

Cons:
- Slightly less literal than a raw copy

### 3. Short prompt that merely references Superpowers ideas

Pros:
- Lower maintenance

Cons:
- Less reliable, especially for a local model, because too much behavior would be implied instead of specified

## Decision

Use option 2: a near-copy of the canonical Superpowers implementer prompt with Pi-native frontmatter and only minimal wording changes.

## Implementation Notes

Implementation should copy the established worker wording closely enough that status handling and escalation remain recognizable to any controller already using the Superpowers style.

### Minimal Output Example

```markdown
Status: NEEDS_CONTEXT

Questions:
- What files should I modify?
- What verification command should I run?

Files changed:
- None
```

If future usage shows the worker needs tighter tool constraints or stronger formatting instructions, those can be added in a later iteration without changing the core design.
