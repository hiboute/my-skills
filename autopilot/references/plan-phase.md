# Plan phase — briefing contract

The planning agent's job is to turn a feature ask into a concrete, reviewable plan. It does **not** implement.

## Agent dispatch

Prefer `subagent_type: Plan` — it's a specialized software-architect agent. Fall back to `general-purpose` only if Plan isn't available in the environment.

Run it with `run_in_background: false` — you need the plan back before Phase 2 can start.

## Briefing template

The agent starts with no knowledge of this conversation. Include all of:

```
You are planning implementation for this feature ask:

<verbatim feature ask from the user>

Context you should gather before planning:
- Repo root: <path>
- Base branch: <branch>
- Any existing spec / issue / linked doc: <paths or URLs>
- Scope hints from the user: <verbatim hints, or "none">

Produce a plan covering:

1. Problem statement — 1 to 3 sentences. What are we building and why.
2. Files to create / modify — bullet list, one line of purpose each.
   Group by directory. Flag new files vs. existing.
3. Public API / interface changes — types, function signatures, route shapes,
   schema changes. "None" is a valid answer.
4. Test strategy — what tests to add, what tests to modify, how to run them
   locally. Check the repo for the existing test command (package.json scripts,
   Makefile, CI config) — don't invent one.
5. Parallelizable workstreams — if parts of the implementation are independent
   (no shared files, no type dependencies on each other), list them. If
   everything must be sequential, say "all sequential" explicitly.
6. Risks / unknowns — things you're unsure about or flags for the reviewer.
   Be specific. "The existing X pattern is unclear, picking Y on the assumption
   that Z" beats "maybe some risk".

Rules:
- Do not write any code. The plan is text only.
- Do not ask clarifying questions — make your best inference and flag it under
  risks instead.
- Keep the plan under ~800 words. Concrete beats exhaustive.
- Match existing project conventions — read 2-3 neighbor files before
  prescribing a pattern.

Report the plan back as your final message.
```

## What the orchestrator does with the plan

1. Print it verbatim to the user.
2. Ask: "Ship as-is, or changes?"
3. If changes: re-dispatch the planning agent, briefing it with the original
   ask **plus** the user's feedback. Don't paraphrase the feedback — include
   the user's exact wording.
4. If approved: store the plan text for Phase 2's implementation agent. It
   will be forwarded verbatim.

## Common mistakes the orchestrator should catch

- Plan is just a restatement of the ask — no file list, no API changes, no
  test strategy. Reject, re-dispatch with "list the specific files you'll
  touch and the test strategy".
- Plan assumes a testing framework that isn't in the repo. Reject with "read
  package.json / the CI config first and use the actual test command".
- Plan has "TBD" or "depends on" in critical sections. Reject with "resolve
  these — if you can't, list them as risks with your chosen assumption".
- Plan suggests unrelated refactoring bundled with the feature. Reject with
  "minimal changes only — remove anything the feature ask didn't require".
