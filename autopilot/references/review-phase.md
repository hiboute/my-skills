# Review phase — briefing contract

The review agent evaluates the diff against the ask + plan, with fresh context, and produces categorized findings. The fix agent then resolves blocking/should-fix findings. Cap iteration at 2 rounds.

## Agent dispatch — review

Use `subagent_type: general-purpose`. Always available, and the briefing
template below does the heavy lifting — specialized reviewer plugins aren't
required.

If you know a more specialized reviewer plugin is installed in this
environment (e.g. one that focuses on PR-scale diffs or security), you may
substitute it, but don't assume it exists. The skill is self-sufficient with
`general-purpose`.

## Briefing template — review

```
You are reviewing a feature implementation. You have not seen the plan or the
implementation session — review with fresh eyes.

Feature ask (from the user):
<ask>

Plan that was approved and followed:
<plan text>

Branch under review: <branch>
Base branch: <base>

Inspection approach:
- Read the full diff: `git diff <base>...<branch>`
- Read each changed file in full (not just the diff hunks). Context matters.
- Check that tests were added per the plan's test strategy and that they run
  (re-run the test command yourself).

Find issues in these categories, ranked by severity:

- BLOCKING — bugs, logic errors, security issues, data-loss risks, broken
  tests, or changes that violate the plan's intent. Anything that should
  stop this from merging.
- SHOULD-FIX — missing edge-case handling, test gaps, unclear code,
  unnecessary scope creep beyond the plan, naming issues that will confuse
  readers.
- NIT — stylistic nits, minor doc gaps, things a maintainer might care
  about but that don't affect correctness.

For each finding, include:
- Severity (BLOCKING / SHOULD-FIX / NIT)
- File path and line number(s)
- What's wrong in one sentence
- Concrete remediation (not "consider refactoring" — say what to change)

If you find zero BLOCKING and zero SHOULD-FIX issues, say so explicitly.

Do not make changes. Do not run the implementation agent's decisions through
a second-guessing pass — evaluate them against the plan and the ask, not
against what you would have done.

Report back as a structured list, grouped by severity.
```

## Decision tree after review

The orchestrator reads the findings and picks one of:

- **Zero BLOCKING + zero SHOULD-FIX findings** — pipeline done. Return the
  review text in the final handoff. NIT findings are included but not
  actioned.
- **Any BLOCKING or SHOULD-FIX, iteration count < 2** — dispatch a fix agent
  (below), then re-run the review agent on the updated diff. Increment the
  counter.
- **Iteration count = 2 and findings remain** — stop. Hand off to the user
  with the remaining findings and a note that the pipeline hit the 2-round
  cap. Do not start a third round without explicit user approval.

## Agent dispatch — fix

Use `general-purpose` for the fix agent. It needs to read the review findings,
the existing code, and make targeted changes.

## Briefing template — fix

```
A code review surfaced issues on branch <branch> (base: <base>). Fix the
BLOCKING and SHOULD-FIX findings below. Leave NITs alone unless they're
trivial to fix in passing.

Review findings:
<verbatim review output>

Rules:
- Make the minimum change that resolves each finding. Don't refactor nearby
  code that wasn't flagged.
- Preserve the commit history style from the implementation phase — new
  commits in logical units, one per finding or per cluster of related fixes.
- Re-run the project's test command before declaring done. If the fixes
  broke other tests, fix those too.
- If a finding is wrong (the reviewer misunderstood), push back: explain in
  your summary why you didn't change anything, rather than making a
  pointless edit.

Report back with:
- One-line summary per finding: what you did (or why you didn't)
- New commit hashes
- Test command output after fixes

Keep the report under 250 words.
```

## Post-fix flow

After the fix agent returns:
1. Read its summary.
2. Re-dispatch the review agent on the updated diff — same briefing, explicitly
   noting "this is round N of review after fixes".
3. Apply the decision tree again.

## Counter hygiene

Track iteration count in the orchestrator's working memory. Reset per
autopilot run; don't carry counts across runs.
