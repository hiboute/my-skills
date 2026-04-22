# Implement phase — briefing contract

The implementation agent(s) turn the approved plan into a committed diff on an isolated branch. They run tests. They do **not** open PRs.

## Isolation

Never dispatch an implementation agent against the main working tree. Pick
one of two paths:

### Path A — Agent-level isolation (single-agent implementation)

Dispatch with `isolation: "worktree"`. The `Agent` tool creates a temporary
worktree for the agent and cleans it up if no changes were made, otherwise
returns the worktree path and branch in the result. Preferred when the plan
says "all sequential" and the implementation fits in one agent call.

### Path B — Orchestrator-managed worktree (parallel agents on shared branch)

Preferred when the plan lists independent workstreams and you'll dispatch
multiple implementation agents that must land on the same branch.

**Directory selection — priority order:**

1. `.worktrees/` if it already exists at the repo root (preferred — hidden)
2. `worktrees/` if it exists (fallback — visible)
3. A `worktree.*director` preference in the project's `CLAUDE.md` (grep for
   it before creating anything)
4. Default to creating `.worktrees/` yourself

**Safety — verify the directory is git-ignored before creating the worktree:**

```bash
git check-ignore -q .worktrees 2>/dev/null || echo "NOT IGNORED"
```

If `.worktrees/` isn't ignored, fix it before proceeding — otherwise the
worktree contents pollute `git status` and can get accidentally committed:

```bash
echo '.worktrees/' >> .gitignore
git add .gitignore && git commit -m "chore: ignore .worktrees/"
```

**Create the worktree on a new branch:**

```bash
SLUG="<short-slug>"            # e.g. add-csv-export
BRANCH="autopilot/$SLUG"
WORKTREE_PATH=".worktrees/$SLUG"
git worktree add "$WORKTREE_PATH" -b "$BRANCH"
```

**Install project dependencies inside the worktree** (auto-detect):

```bash
cd "$WORKTREE_PATH"
[ -f package.json ]    && (command -v pnpm >/dev/null && pnpm install) \
                            || (command -v npm  >/dev/null && npm install)
[ -f pyproject.toml ]  && (command -v poetry >/dev/null && poetry install) \
                            || (command -v uv >/dev/null && uv sync) \
                            || (command -v pip >/dev/null && pip install -e .)
[ -f requirements.txt ] && command -v pip >/dev/null && pip install -r requirements.txt
[ -f Cargo.toml ]      && command -v cargo >/dev/null && cargo build
[ -f go.mod ]          && command -v go    >/dev/null && go mod download
```

**Baseline test run** — optional but recommended. If the project has a test
command (check `package.json` scripts, `Makefile`, `pyproject.toml`), run it
once inside the worktree and record the result. This lets the review phase
distinguish new test failures from pre-existing ones.

**Hand the path to each dispatched agent** — every parallel agent gets
`cwd: $WORKTREE_PATH` in its briefing so they all work against the same
worktree on the same branch.

**Cleanup after the full pipeline** — after the user accepts the handoff
(or rejects and wants to discard), run:

```bash
git worktree remove "$WORKTREE_PATH"   # or --force if uncommitted state
```

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
- **One worktree per agent, one branch per agent** — never share a worktree
  across concurrent agents. Two agents running `git add`/`git commit` in the
  same worktree contend on `.git/index.lock`, and operations fail
  non-deterministically even when the agents touch disjoint files. Give each
  agent its own worktree on its own stream branch, branched from the target
  branch.

### Setup — one worktree per stream

Create the target branch first (it's the integration branch the orchestrator
merges streams onto), then fan out one worktree + branch per stream:

```bash
SLUG="<short-slug>"
TARGET="autopilot/$SLUG"
# Integration branch the streams merge into (created from the base branch
# the orchestrator was given, e.g. main or develop).
git worktree add ".worktrees/$SLUG" -b "$TARGET"

# One worktree + branch per independent stream, each branched from $TARGET
for STREAM in stream-a stream-b stream-c; do
  git worktree add ".worktrees/$SLUG-$STREAM" -b "$TARGET-$STREAM" "$TARGET"
done
```

Hand each dispatched agent its own `cwd: .worktrees/<slug>-<stream>` in the
briefing. Agents commit freely on their own branches — no lock contention.

### Integration — after all streams finish

Once every parallel agent has returned successfully, the orchestrator (or a
dedicated integration agent) merges each stream branch back into `$TARGET`
in a controlled sequence:

```bash
cd ".worktrees/$SLUG"
for STREAM in stream-a stream-b stream-c; do
  git merge --no-ff "$TARGET-$STREAM"   # or: cherry-pick its range
done
```

Prefer `--no-ff` merges so each stream's commits stay recognizable in the
log (the commit-message prefix `[stream-A]` etc. is still useful even with
merge commits). If a merge conflicts, the orchestrator investigates — that
usually means the independence check in the plan was wrong, and is a signal
to stop and re-plan rather than paper over it.

After successful integration, clean up the per-stream worktrees (the
integration worktree stays until the final handoff):

```bash
for STREAM in stream-a stream-b stream-c; do
  git worktree remove ".worktrees/$SLUG-$STREAM"
done
```

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
