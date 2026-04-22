# Implement phase — briefing contract

The implementation agent(s) turn the approved plan into a committed diff on an isolated branch. They run tests. They do **not** open PRs.

## Isolation

Pick one of two paths:

- **Agent-level isolation** — dispatch with `isolation: "worktree"`. The tool creates a temporary worktree for the agent and cleans it up if no changes were made, otherwise returns the worktree path and branch in the result. Preferred when the implementation fits in one agent call.
- **Skill-level isolation** — invoke the `superpowers:using-git-worktrees` skill first to create a worktree the orchestrator controls. Preferred when dispatching multiple parallel implementation agents that must land on the same branch.

Never dispatch an implementation agent against the main working tree.

## Branch naming

`autopilot/<short-slug>` where the slug is a 3–6-word summary of the feature
(kebab-case, no trailing slashes). Examples: `autopilot/add-csv-export`,
`autopilot/fix-oauth-refresh-race`.

## Single-stream dispatch

When the plan says "all sequential", dispatch one agent with the full plan.

```
You are implementing this feature on branch <branch> in worktree <path>.

Feature ask (verbatim from the user):
<ask>

Approved plan:
<plan text, verbatim>

Scope hints from the user (honor verbatim, don't relitigate):
<hints or "none">

Rules:
- Implement exactly what the plan says. No bonus refactors, no unsolicited
  dependency bumps, no "while I'm here" changes.
- Follow the test strategy in the plan. Add tests as the plan specifies.
- Run the project's test command before declaring done. If tests fail,
  investigate and fix — don't paper over failures with skips.
- Commit in logical chunks with descriptive messages. One commit per
  conceptual unit beats a single "implement feature" commit.
- Match existing code conventions. Read 2-3 neighbor files before
  introducing patterns.
- If you hit a blocker the plan didn't anticipate, stop, commit what works,
  and return a summary explaining the blocker and what you need.

Report back with:
- List of commits you made (one line each, with the hash)
- Files changed (grouped: new / modified / deleted)
- Tests added or modified, and the test command output (pass/fail)
- Any deviation from the plan, with reason
- Any blocker that prevented you from finishing

Keep the final report under 300 words. Detail belongs in commit messages.
```

## Parallel dispatch

When the plan lists independent workstreams:

- **Independence check** — two workstreams are independent only if they don't
  share files AND don't depend on each other's types / APIs. If one defines
  a type the other consumes, they're sequential.
- **Single message, multiple `Agent` calls** — parallel dispatch means one
  orchestrator turn with N `Agent` tool uses. Anything else is sequential.
- **Shared context per agent** — each parallel agent gets: the verbatim ask,
  the full plan (not just its slice — it needs to understand the whole
  feature), its assigned slice, and a list of the other workstreams running
  in parallel so it knows what **not** to touch.
- **Same branch, same worktree** — all parallel agents write to the same
  worktree. Git handles interleaved commits fine; just give each agent a
  distinct commit message prefix (`[stream-A]`, `[stream-B]`) so the
  reviewer can trace work later.

If two supposedly-independent agents both edit the same file, that's a plan
bug — surface it to the user after the phase and consider whether to re-plan.

## What the orchestrator does after the phase

1. Read each implementation agent's summary.
2. If any reported a blocker: decide — dispatch follow-up with resolved
   scope, ask the user, or stop.
3. If tests failed in any summary: count that toward the review phase's
   fix-loop budget before proceeding, or dispatch a targeted fix agent now.
4. If all agents reported success: advance to Phase 3 (review). Don't verify
   by reading the diff yourself — that's the reviewer's job with fresh
   context.
