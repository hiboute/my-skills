---
name: ideation
description: Analyze a codebase for concrete improvements — security vulnerabilities, performance wins, documentation gaps, code-quality issues, code-revealed feature opportunities, or UI/UX friction — and publish findings to GitHub as issues with proper labels. Use when the user asks to audit, review, or find issues in a repo; wants a triaged list of improvements; or says things like "find security issues and file them", "audit performance", "discover improvement opportunities", "what can we clean up in this codebase?".
---

# Ideation

Run targeted codebase analysis in one of six modes and publish each finding to GitHub as a labeled issue.

## Modes

Pick the mode that matches the user's intent. If they don't specify, ask which one.

| Mode | Focuses on | Load |
|---|---|---|
| `security` | OWASP-style vulnerabilities, secrets, unsafe inputs | `categories/security.md` |
| `performance` | Bundle size, runtime, memory, DB, network, rendering, caching | `categories/performance.md` |
| `documentation` | README, API docs, inline comments, examples, architecture, troubleshooting | `categories/documentation.md` |
| `code-quality` | Large files, code smells, complexity, duplication, naming, structure, linting, tests, types, dead code | `categories/code-quality.md` |
| `code-improvements` | Code-revealed feature opportunities extending existing patterns | `categories/code-improvements.md` |
| `ui-ux` | Usability, accessibility, perceived performance, visual polish, interactions | `categories/ui-ux.md` |

Each mode has its own analysis methodology, severity/impact scale, and issue template. Load only the mode the user picked — don't preload all six.

## When to use

Trigger on requests like:
- "Audit this repo for security issues"
- "Find performance wins and file them as issues"
- "What documentation is missing? Open issues for each gap"
- "Review code quality in <dir> and file cleanup tasks"
- "What could we build here based on existing patterns?"
- "Find UI/UX friction and open issues"

Do **not** use for:
- Strategic product planning → use the `roadmap` skill.
- Fixing one specific known issue → just do it, don't run the whole pipeline.
- Running all six modes at once → pick one per invocation. If the user wants multiple, run them in sequence with separate summaries.

## Inputs

Before starting:

1. **Target repository** in `owner/repo` form. If not provided, ask.
2. **Mode** (one of the six above). If the user was vague, ask which mode; don't assume.
3. **Scope** (optional) — a subdirectory, file pattern, or "whole repo". Default to whole repo.
4. **Max findings** (optional) — defaults to **5 per invocation**. Quality over volume.

Confirm the target + mode once, then proceed autonomously.

## Pipeline

1. **Load the mode file** (`categories/<mode>.md`). It contains the specific analysis steps, severity scale, and issue-body template for that mode.
2. **Analyze the codebase.** Read files, run searches, look at dependencies. Follow the methodology in the mode file.
3. **Shortlist findings.** Aim for the 3–7 most impactful, non-duplicate, actionable issues. If you find 20 candidates, pick the 5 best — don't dump everything.
4. **Check existing issues.** For each finding, scan open issues in the repo. If something similar is already tracked, skip the finding or note it as a duplicate in the summary.
5. **Publish.** Load `references/github-mapping.md` and create one GitHub issue per finding, with the labels and body format from the mode file.
6. **Summarize.** Print a Markdown summary back to the user with a link to each created issue.

## Operating rules

1. **One mode per run.** Keeps findings coherent and labels consistent.
2. **Cap at 5 findings by default.** Users can ask for more explicitly. A focused list gets triaged; a dump gets ignored.
3. **Rank by impact, not by what's easy to find.** Critical security > minor code smell every time.
4. **Avoid false positives.** If you're not confident the finding is real, either verify deeper or drop it. Do not ship speculative issues.
5. **Cite file paths and line numbers** in every issue body. Without specific locations, findings are useless.
6. **Concrete remediation required.** Every issue needs "how to fix it" — not just "this is bad". If you can't describe the fix, the finding isn't ready.
7. **Idempotent labels.** Create labels if missing; never error on duplicates.
8. **Draft-friendly.** File issues with `status:triage` label so the user can review before committing to action.

## Output contract

At the end of a successful run, the target repository will contain:
- N new issues (one per finding), each with:
  - Title summarizing the finding
  - Body including: description, rationale, affected files (with line numbers where applicable), proposed remediation, category/severity, references
  - Labels: `ideation`, `ideation:<mode>`, `severity:<value>` (or `impact:<value>` for performance/code-improvements), category label, `status:triage`

Plus a Markdown summary back to the user listing each created issue with its severity.

## Mode quick reference

Before reading the full mode file, remember:

- **security** uses `severity: critical | high | medium | low` and CWE references
- **performance** uses `impact: high | medium | low` and effort estimates
- **documentation** uses `priority: high | medium | low` and `targetAudience`
- **code-quality** uses `severity: critical | major | minor | suggestion`
- **code-improvements** uses `effort: trivial | small | medium | large | complex` — this is the one mode that finds *features* the code reveals, not defects
- **ui-ux** uses five categories (usability, accessibility, performance, visual, interaction) and is the only mode that benefits from running the app in a browser (optional)

For full rules, templates, and GitHub label schemas, load `references/github-mapping.md` alongside the mode file.
