# Publishing ideation findings to GitHub

Load this alongside the mode-specific category file. The category file tells you **what** to look for and what goes in the issue body; this file tells you **how** to publish to GitHub consistently across modes.

Use the GitHub MCP tools (`mcp__github__*`). If only `gh` CLI is available, the equivalent commands are noted.

## Shared labels (create if missing, idempotent)

Every ideation issue gets these regardless of mode:

| Label | Color | Purpose |
|---|---|---|
| `ideation` | `#0052CC` | Marks all issues created by this skill |
| `ideation:<mode>` | see category file | Identifies which mode ran (e.g., `ideation:security`) |
| `status:triage` | `#EDEDED` | Newly filed, awaiting human triage |

Mode-specific labels (severity/impact/priority/effort, category, etc.) are listed in each category file. Create any missing ones before filing issues.

Use `mcp__github__get_label` to check existence. On creation-denied responses, log a warning and proceed without that label — the issue body is still useful.

## Step 1 — Pre-flight

1. Confirm the target repository (`owner/repo`).
2. List open issues (`mcp__github__list_issues`, state: `open`, per_page: 100). Keep a mental map of titles so you can skip duplicates in Step 3.
3. Ensure labels exist. Create any missing ones idempotently.

## Step 2 — Shortlist

The category file describes how to find candidate findings. Before publishing, apply these filters:

- **Cap at 5 findings** unless the user asked for more. Rank by impact/severity.
- **Drop duplicates** of existing open issues.
- **Drop low-confidence findings** — if you're not sure it's real, don't file it.
- **Drop findings without a remediation** — if you can't describe the fix, it's not ready.

## Step 3 — File the issues

For each finding:

1. Use the **issue body template from the category file** verbatim for its structure.
2. Set the title to a short summary of the finding — imperative mood, ideally under 80 chars. Examples:
   - "Fix SQL injection in `searchUsers()` handler"
   - "Replace moment.js with date-fns to shrink bundle ~270KB"
   - "Add JSDoc to `src/auth/token.ts` public functions"
3. Apply all the labels specified in the category file (plus the three shared labels above).
4. Do not set milestones or assignees — leave triage to the user.

Example (`mcp__github__issue_write` equivalent):

```
action: create
owner: <owner>
repo: <repo>
title: "<issue title>"
body: <<EOF
<body from category template>
EOF
labels:
  - ideation
  - ideation:<mode>
  - <severity/impact/priority/effort labels>
  - <category label>
  - status:triage
```

Record the issue number and URL returned by the API — you'll need them for the summary.

## Step 4 — Summary back to the user

After filing everything, print (to the user, not to GitHub) a Markdown summary. The exact structure is specified in each category file. The summary should:

- State the repo and mode
- Give counts by the mode's ranking dimension (severity / impact / priority / effort)
- List each filed issue with `#<N> [<severity>] <title>` and its URL
- List skipped duplicates, if any
- List any labels or operations that failed so the user can address them manually

## Error handling

- **Rate limits** — pause, report the count filed so far, ask the user whether to resume
- **Permission denied** — stop, surface the missing scope (`issues:write`)
- **Label creation denied** — continue without the missing labels, note in summary
- **Body too long** — trim the examples section; never skip required fields (summary, affected files, remediation)

Never retry silently more than once. Surface problems.

## Cross-mode consistency notes

- The `ideation` label is always present.
- The `ideation:<mode>` label mirrors the skill invocation (`ideation:security`, `ideation:performance`, etc.).
- Severity/impact/priority/effort labels follow the palette in each category file. Colors are suggestions — if labels already exist with different colors, leave them alone.
- Never set `assignees` — users triage their own ideation issues.
- Never close issues automatically, even if you suspect they're already fixed. Let the user decide.

## Idempotency

Running the same mode twice against the same repo should **not** create duplicate issues. The Step 2 duplicate check is your guardrail. If you're unsure whether a finding is a duplicate:
- If titles clearly refer to the same problem → skip, note as duplicate
- If titles are merely related → file the new issue, and mention the related issue in the body ("Related to #N")
