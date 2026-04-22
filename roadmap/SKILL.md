---
name: roadmap
description: Generate a strategic product roadmap for a codebase and publish it to a GitHub Project (v2), OR extend an existing roadmap by adding a new sprint (and optionally a new phase) on top. Issues become project items; phase/priority/complexity/impact/sprint are custom fields. Re-runnable — each call is idempotent and adds one new sprint if the project already exists, plus a new phase when the work genuinely belongs to a new strategic theme. Use when the user asks to create, plan, publish, or extend a roadmap; wants to "add a new sprint", "open a new phase", "plan the next sprint", "start Phase 5 on X"; or says things like "build me a roadmap for this project", "plan phases on a GitHub project", "turn this repo into a roadmap of issues on a project board".
---

# Roadmap

Analyze a codebase, infer its audience and vision, then generate a MoSCoW-prioritized feature roadmap and publish it to a **GitHub Project (v2)** — with one issue per feature, added as project items, and phase/priority/complexity/impact/sprint stored as project custom fields.

The skill runs in one of two **modes**, chosen automatically:

- **Greenfield** — no existing roadmap project for this repo. Creates the project, fills it with 5–10 features across 3–4 phases, all tagged as `Sprint 1`.
- **Incremental** — a `<Repo name> Roadmap` project already exists. Reads the current board state (done/in-progress/open counts, per-phase completion), then generates **3–7 new features** as the next sprint (`Sprint N+1`). New features usually reuse the existing phases, but the skill **can open a new phase** (`Phase N+1 — <Theme>`) when the batch genuinely belongs to a new strategic theme — see the trigger rules in `~/.claude/skills/my-skills/roadmap/references/github-mapping.md`. Existing items are left alone. The skill is safe to re-run: each run adds one sprint on top of whatever's there, and optionally a new phase.

## When to use

Trigger on requests like:
- "Create a roadmap for <repo>"
- "Plan the next 3 phases of this project as a GitHub project board"
- "Turn my repo into a prioritized feature backlog on a Project"
- "What should I build next? File issues and set up the project board"

Do **not** use for:
- A single feature idea → use the user's existing workflow to file one issue.
- Refactoring / bug hunting → use the `ideation` skill instead.
- A roadmap where the user has already written it → just file the issues they gave you.

## Inputs

Before starting, make sure you have:

1. **Target repository** in `owner/repo` form. If the user doesn't specify one, ask.
2. **Project owner** — usually the same as the repo owner (user or org). GitHub Projects v2 live at the user/org level, not inside the repo. If the repo is under a personal account, the project owner defaults to that user; if under an org, default to the org. Only ask if it's ambiguous.
3. **Codebase access** — either the local working directory is the project, or you can read it remotely with `gh api repos/<owner>/<repo>/contents/<path>` and `gh search code`.
4. **Auth scope** — creating/editing Projects v2 requires the `project` OAuth scope (and `read:project` for reads). The user may need `gh auth refresh -s project` first. If the first project call fails with a scope error, surface this clearly and stop.
5. **Optional**: competitor analysis notes the user provides. If present, weave them in (see "Competitor insights" below).

Confirm the repo + project owner once, then proceed autonomously. Do not ask clarifying questions about audience, vision, or priorities — infer them from the code, README, and package manifest. Making confident inferences is the whole point of this skill.

## Pipeline

The skill runs two phases. Phase 2 depends on Phase 1.

### Phase 1 — Discovery

Produce a structured understanding of the project. Read:
- `README.md`, `CONTRIBUTING.md`, any `docs/` folder
- `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod`
- A sample of source files to gauge patterns and maturity
- Existing issues / milestones (so the roadmap layers onto, not duplicates, what's there)
- Recent git history (activity, contributor count)

Infer, in order:
1. **Project type** (web-app, cli, library, api, desktop-app, mobile-app, other)
2. **Tech stack** (primary language, frameworks, key dependencies)
3. **Target audience** — primary persona, secondary personas, pain points, goals, usage context
4. **Product vision** — one-liner, problem statement, value proposition, success metrics
5. **Current state** — maturity (idea/prototype/mvp/growth/mature), existing features, known gaps, technical debt
6. **Competitive context** — alternatives, differentiators, market position
7. **Constraints** — technical, resources, dependencies

For the detailed methodology and what to look for at each step, load `~/.claude/skills/my-skills/roadmap/references/discovery.md`.

**Hold the discovery object in memory** — you don't need to write it to disk. It's the input to Phase 2.

### Phase 2 — Features + Publishing

First, **detect the mode**. Try to find an existing `<Repo name> Roadmap` project for the owner. If one exists, you're in incremental mode — see `~/.claude/skills/my-skills/roadmap/references/github-mapping.md` → "State analysis (incremental mode)" to read the current board before brainstorming. That analysis becomes extra input alongside the discovery object.

Then generate features. The target count depends on mode:

- **Greenfield** — generate **5–10 features** organized into **3–4 phases**, all tagged for `Sprint 1`.
- **Incremental** — generate **3–7 new features** as the next sprint (`Sprint N+1`). Default to reusing existing phases; opening a new phase (`Phase N+1`) is allowed only when the trigger rules in `~/.claude/skills/my-skills/roadmap/references/github-mapping.md` §2.5.4 are met (explicit user ask, or a coherent new theme with ≥2 features when earlier phases are mostly Done). Do **not** re-file features that already exist as open items on the board.

For each feature, produce:
- `title` — short, action-oriented
- `description` — 2–3 sentences
- `rationale` — why this matters for the primary persona (cite pain points / goals)
- `priority` — `must` / `should` / `could` / `wont` (MoSCoW)
- `complexity` — `low` / `medium` / `high`
- `impact` — `low` / `medium` / `high`
- `acceptance_criteria` — testable bullets
- `user_stories` — "As a <persona>, I want to <action> so that <benefit>"
- `dependencies` — feature titles this one needs first (can reference existing open issues on the board)

Organize features into phases in this shape (greenfield creates the full structure; incremental mode reuses existing phases and may add a new one — see `~/.claude/skills/my-skills/roadmap/references/github-mapping.md` for the trigger rules):
- **Phase 1 — Foundation / MVP** — must-haves, quick wins
- **Phase 2 — Enhancement** — should-haves, UX improvements
- **Phase 3 — Scale / Growth** — could-haves, advanced features
- **Phase 4 — Future / Vision** — long-term, experimental (optional)

Prioritization rules:
- Core user need + low complexity → `must` / Phase 1
- Core user need + high complexity → `must` or `should` / Phase 1–2 (big bets)
- Nice-to-have + low complexity → `could` / Phase 2–3 (fill-ins)
- Nice-to-have + high complexity → `wont` or Phase 4 (avoid)

For the prioritization framework and feature-brainstorming prompts, load `~/.claude/skills/my-skills/roadmap/references/features.md`.

**Then publish to a GitHub Project.** The mapping between roadmap structures and GitHub artifacts (project creation, custom fields to create, labels, issue body template, dependency links) is in `~/.claude/skills/my-skills/roadmap/references/github-mapping.md` — load it before you start creating anything.

## Operating rules

1. **Non-interactive by default.** Don't ask the user to fill gaps in discovery — infer them. The only things you may confirm are the target repo and the project owner. The user may also steer an incremental run ("focus this sprint on performance") — honor that as a bias during feature generation.
2. **Mode is decided automatically.** Existing `<Repo name> Roadmap` project → incremental. No project → greenfield. Don't ask which mode to run in.
3. **Idempotent project setup.** If a project with the intended title already exists for the owner, reuse it rather than creating a duplicate. Same for custom fields and single-select options — list before creating, treat "already exists" as success.
4. **Existing items are read-only.** In incremental mode, never edit fields on existing items, never re-file their issues, never change their sprint or phase tag. Adding new **options** to the Phase or Sprint single-select fields is allowed (and is how sprints/phases extend) — that's a field-level change, not an item-level one. The one item-level exception: the first time the Sprint field is created on a pre-existing project, backfill all existing items as `Sprint 1` (see `~/.claude/skills/my-skills/roadmap/references/github-mapping.md`).
5. **Don't duplicate existing issues.** Before filing, list open issues in the repo and skip or merge with anything that already describes the same feature. For duplicates, still add the existing issue to the project and set its fields (greenfield only) — that way the board reflects reality.
6. **Create labels idempotently.** If a label already exists, reuse it. Never error on a duplicate-label response.
7. **Draft-friendly.** Create issues as normal (not draft) but set the project Status field to `Todo` and label them `status:idea` so the user can triage before committing to them.
8. **Ordering.** Project → fields → labels → (if incremental: analyze state) → issues → add items to project → set field values → second pass for dependencies. File issues in dependency order so the `depends on #N` references resolve.
9. **Report back.** After publishing, print a summary: project URL, mode, sprint just added, new items' breakdown by priority and phase, carryover items from prior sprints, any features skipped as duplicates, and any fallbacks taken.

## Competitor insights

If the user supplies competitor analysis (either inline or by pointing to a file), use it to:
- Populate the `competitive_context` part of the discovery
- Boost priority on features that directly address competitor pain points
- Tag those issues with the `competitor-insight` label and mention the specific competitor + pain point in the issue body's rationale. Also set the `Competitor` text field on the project item if it was created (see `~/.claude/skills/my-skills/roadmap/references/github-mapping.md`).

No competitor data → skip this entirely. Don't invent competitors.

## Output contract

At the end of a successful run, the following will exist on GitHub:

- **One Project (v2)** named `<Repo name> Roadmap` under the project owner, with custom fields: `Phase`, `Priority`, `Complexity`, `Impact`, `Sprint`, and (if competitor data was provided) `Competitor`. The default `Status` field is kept and all new items start at `Todo`.
- **Issues** in `owner/repo` (one per new feature), each added as an item on the project with the fields above populated. In greenfield mode, all are tagged `Sprint 1`. In incremental mode, all new items are tagged `Sprint N+1` — existing items retain their prior sprint. If a new phase was opened, it's added as a new option on the Phase field and only new items get tagged with it.
- **Dependency cross-references** via `depends on #N` in issue bodies, including back-references to existing open items where relevant.

Plus a short Markdown summary posted back to the user (not to GitHub) with:
- Mode (greenfield or incremental), the sprint that was just added, and any new phase that was opened
- Link to the Project board
- Counts by phase and priority for the new sprint
- Carryover: open items from prior sprints, flagged if they look stalled
- Any features skipped as duplicates of existing issues
- Any fallbacks taken (e.g., a field couldn't be created, so a label carried the signal instead)
