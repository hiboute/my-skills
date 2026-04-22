---
name: autopilot
description: Autonomously build a feature end-to-end — plan, implement in an isolated git worktree, then run an independent AI review with a bounded fix loop. Orchestrates three phases as fresh-context subagents so the main session stays focused on coordination. Use when the user asks to "ship feature X autonomously", "build this end-to-end", "plan and implement then review", "take this issue from spec to PR", or gives a feature description they want delivered without further step-by-step guidance. Pattern inspired by Aperant (AndyMik90/Aperant).
---

# Autopilot

Take a feature request from description to reviewed, diff-ready implementation. The skill runs a three-phase pipeline — **plan → implement → AI review** — with each phase dispatched as a subagent so the orchestrator session stays lean. If the review finds issues, a bounded fix loop runs before surfacing anything to the user.

This is the small-team-in-a-skill version of Aperant's autonomous build pipeline.

## When to use

Trigger on requests like:
- "Build <feature> autonomously"
- "Take this issue and ship it end-to-end"
- "Plan, implement, then review <thing>"
- "Do <feature> for me, I'll review at the end"
- "Run the full pipeline on <ask>"

Do **not** use for:
- Single-file edits or obvious one-line fixes → just do it, skip the pipeline.
- Debugging an existing bug → investigate and patch it directly; the autopilot
  phases are designed for net-new feature work, not root-cause chases.
- Strategic planning or roadmap work → use the `roadmap` skill.
- Finding issues to file, not fixing them → use the `ideation` skill.
- Features that need live product input from the user partway through → do it interactively instead.

## Inputs

Before starting, gather:

1. **Feature ask** — a description, a GitHub issue URL/number, or a spec file path. If the ask is one vague sentence, ask one clarifying question, then proceed.
2. **Target branch** — the base branch to diff against and eventually merge into. Default to `main`; honor the project's convention if `develop` or similar is used (check recent PR titles or `CONTRIBUTING.md`).
3. **Working directory** — the repo root. Must be a git repo. If the tree is dirty, surface that and ask before proceeding — the pipeline assumes a clean base.
4. **Scope hints** (optional) — "keep it prompt-only", "don't touch tests", "reuse existing X". Pass these verbatim into each phase agent.

Confirm the feature ask + base branch once, then proceed autonomously through the three phases.

## Pipeline

Each phase dispatches a fresh-context subagent via the `Agent` tool. The orchestrator (you) stays at the coordination layer: briefing agents, reading their summaries, deciding whether to advance or iterate.

### Phase 1 — Plan

Dispatch a **planning agent** with `subagent_type: Plan` (or `general-purpose` if Plan isn't available). The agent produces a written plan covering:

- Problem statement (1–3 sentences)
- Files to create / modify, with a one-line purpose each
- Public API or interface changes, if any
- Test strategy (what to add, what to modify)
- Independent workstreams that can be parallelized in Phase 2, if any
- Risks / unknowns

Full briefing rules and the plan contract: `~/.claude/skills/hiboute-skills/autopilot/references/plan-phase.md`.

Print the plan back to the user and **ask for confirmation before Phase 2**. The user may edit the plan, drop scope, or request changes. If they do, re-dispatch the planning agent with their feedback.

Skip the confirmation prompt only if the user explicitly said "don't stop to confirm" or similar in their original ask.

### Phase 2 — Implement

Set up isolation: either use the `Agent` tool with `isolation: "worktree"` (the tool creates and cleans up a temporary worktree automatically), or create one manually via `git worktree add` when you need multiple parallel agents to share the same branch. Full manual-setup procedure: `~/.claude/skills/hiboute-skills/autopilot/references/implement-phase.md`. Main branch never gets touched directly.

Dispatch an **implementation agent** (`subagent_type: general-purpose`) with:
- The approved plan (verbatim)
- The feature ask (verbatim)
- Explicit instruction to follow the plan, write tests per the plan's test strategy, and run the project's test command before declaring done
- A request for a concise final summary (under 300 words): what files changed, what tests were added/modified, any plan deviations with reasons

If the plan flagged independent workstreams, dispatch them in parallel — one subagent per workstream, single message with multiple `Agent` tool calls. Each subagent gets its slice of the plan plus shared context (types, conventions).

Full briefing rules and parallelization heuristics: `~/.claude/skills/hiboute-skills/autopilot/references/implement-phase.md`.

### Phase 3 — AI review (with bounded fix loop)

Dispatch a **review agent** (`subagent_type: general-purpose`). Brief it with:
- The original feature ask
- The plan
- The diff against the base branch
- Instruction to find bugs, logic errors, test gaps, security issues, and plan deviations
- A request for findings categorized by severity: `blocking` / `should-fix` / `nit`, with file:line citations

Collect the findings. **Decision tree**:

- **No `blocking` or `should-fix` findings** — pipeline done. Summarize and hand off.
- **Some `blocking` or `should-fix` findings, iteration count < 2** — dispatch a **fix agent** (`general-purpose`) with the findings and the existing diff context. After the fix agent returns, re-run the review agent on the updated diff. Cap at 2 fix iterations.
- **Iteration cap hit with findings remaining** — stop. Surface remaining findings to the user with a clear "pipeline hit the 2-round fix cap" note. Don't iterate further without explicit go-ahead.

Full briefing rules and the severity scale: `~/.claude/skills/hiboute-skills/autopilot/references/review-phase.md`.

## Operating rules

1. **Orchestrator-first.** You coordinate; agents execute. Don't read files or write code yourself during phases 2–3 unless an agent explicitly failed — delegation keeps the main context clean.
2. **Fresh context per phase.** Each phase agent gets only the inputs it needs. Don't forward the previous agent's full transcript — forward its summary plus the plan/ask.
3. **Worktree isolation on implementation.** Never implement against the main working tree. Use `isolation: "worktree"` on the implementation agent, or create one manually with `git worktree add <path> -b autopilot/<slug>` before dispatching (see `~/.claude/skills/hiboute-skills/autopilot/references/implement-phase.md` for the full safety procedure).
4. **One confirmation gate.** After Phase 1. Not before or after other phases. The whole point of the skill is autonomy — gating at every step defeats it.
5. **Bounded fix loop.** At most 2 fix rounds before surfacing to the user. Infinite iteration hides real problems.
6. **Parallel only when independent.** If workstreams share files or depend on each other's types/APIs, run them sequentially. Spurious parallelism produces merge conflicts and context drift.
7. **Don't invent plan deviations.** If the implementation agent couldn't follow the plan, its summary should say why — don't paper over that in your report to the user.
8. **Preserve the user's scope hints.** If they said "prompt-only" or "don't touch tests", forward that verbatim to every phase agent. Agents lose that nuance without explicit reminders.
9. **One feature per run.** If the ask bundles multiple independent features, ask the user to split them or pick one.

## Output contract

At the end of a successful run:
- **Plan** — printed inline to the user after Phase 1 (for approval) and included in the final summary.
- **Branch / worktree** — named something recognizable (`autopilot/<short-slug>`), ahead of the base branch with the implementation commits.
- **Diff** — committed on the branch. Tests pass per the plan's test strategy.
- **Review report** — the final review pass's findings, with severities. Included in the final summary.
- **Fix log** (if any) — one-line summary per fix round, e.g. `round 1: addressed 2 blocking + 1 should-fix, re-review clean`.
- **Final handoff** — a markdown summary back to the user with: feature ask, branch/worktree path, commits, diff stats, test results, review status, and any open findings that hit the iteration cap.

No GitHub PR is opened by this skill — handoff is the branch/worktree. Use `/ship` (gstack) or manual `gh pr create` to ship it.

## Failure handling

- **Planning agent returns something vague** — re-dispatch once with explicit feedback. If it's still vague, stop and tell the user the ask is underspecified.
- **Implementation agent reports it couldn't finish** — read its summary, decide whether to dispatch a follow-up with clarified scope, ask the user, or surface the blocker.
- **Test command fails and the implementation agent didn't notice** — dispatch a fix agent with the test output. Count this toward the 2-round fix cap.
- **Review agent itself looks wrong** (produces obviously bad findings) — dispatch it once more with a sharper brief. If still bad, summarize what you have and flag the review quality in the handoff.

## Attribution

Pipeline pattern ported from [Aperant](https://github.com/AndyMik90/Aperant) (AndyMik90) — an Electron-based autonomous coding framework. This skill is the in-session, single-Claude-Code-conversation adaptation of their orchestrator → planner → coder → QA workflow.
