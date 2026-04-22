# Feature generation methodology

Input: the discovery object produced in Phase 1.
Output: 5–10 features, organized into 3–4 phases, each with full MoSCoW + impact/complexity scoring, ready to publish as GitHub issues.

## Brainstorming sources

Generate feature candidates from these buckets, in order:

### 1. User pain points
For each `target_audience.pain_points[i]`:
- What minimum feature directly relieves this pain?
- Is there a lighter MVP version that tests the hypothesis first?

### 2. User goals
For each `target_audience.goals[i]`:
- What feature makes this goal faster, cheaper, or possible?
- What workflow improvement helps them repeat it?

### 3. Known gaps
For each `current_state.known_gaps[i]`:
- File a feature to close the gap. Priority depends on how central the gap is.

### 4. Competitive differentiation
For each `competitive_context.differentiators[i]`:
- What feature deepens this differentiator?
- What feature neutralizes a competitor's advantage?

### 5. Technical improvements
For each `current_state.technical_debt[i]`:
- Is this debt blocking user-facing features? If yes, surface it as a roadmap feature. If no, skip — it belongs in the ideation skill, not the roadmap.

### 6. Competitor pain points (if present)
For each entry in `competitive_context.competitor_pain_points`:
- What feature would explicitly solve what frustrates competitors' users?
- Boost priority for these — they're differentiation opportunities.

## MoSCoW prioritization

Apply these rules per feature:

**`must`** — required for the MVP or the next meaningful release
- Users fundamentally cannot function without it
- Regulatory / compliance requirements
- Blocks critical competitor pain points
- Required by a Phase 1 milestone

**`should`** — important but not critical
- Significant value, clear user demand
- Can slip one phase without blowing up the plan
- Addresses common competitor weaknesses

**`could`** — nice to have
- Enhances experience, doesn't unblock anything
- Safe to cut if time runs out

**`wont`** — acknowledged but out of scope
- Document for completeness, don't plan work

Aim for roughly a 30/40/25/5 split across must/should/could/wont. If everything is "must", you haven't prioritized.

## Complexity + impact assessment

**Complexity**
- `low` — 1–2 files, single component, under a day
- `medium` — 3–10 files, multiple components, 1–3 days
- `high` — 10+ files or architectural change, multiple days

**Impact**
- `high` — core user need, revenue driver, addresses a competitor pain point
- `medium` — meaningful UX improvement, secondary need
- `low` — edge case, polish

**Priority matrix** (use this to cross-check your MoSCoW):
- High impact + low complexity → do first (Phase 1, `must`)
- High impact + high complexity → big bet, plan carefully (Phase 1–2, `must` or `should`)
- Low impact + low complexity → fill-in (Phase 2–3, `could`)
- Low impact + high complexity → reject or push to Phase 4 (`wont`)

## Phase organization

Group features into 3–4 phases:

**Phase 1 — Foundation / MVP**
- All `must` features
- High-impact + low-complexity quick wins
- Features that unblock later work

**Phase 2 — Enhancement**
- `should` features
- UX improvements
- Medium-complexity work

**Phase 3 — Scale / Growth**
- `could` features
- Advanced functionality
- Performance / polish

**Phase 4 — Future / Vision** (optional, only if you have compelling long-term ideas)
- Experimental or market-expansion features
- Things the discovery suggests are possible but not ready

Dependencies must respect phase ordering — if Feature B depends on Feature A, A must be in an earlier or equal phase.

## Dependency mapping

Feature B depends on Feature A if:
- B requires A's functionality to work end-to-end
- B modifies code A introduces
- B uses APIs / schemas / models A creates

Mark these in each feature's `dependencies: []` field using the depended-on feature's title. When publishing to GitHub, they'll be converted to issue-number references.

## Milestone-within-phase (optional)

If a phase has more than 5 features, break it into milestones that each deliver a demonstrable outcome. Good milestones are:
- **Demonstrable** — you can show progress
- **Testable** — you can verify completion
- **Valuable** — users get something, not just code

Examples: "Users can create and save documents", "Payment processing goes live", "Mobile app ships to App Store".

If a phase has ≤5 features, skip milestones and treat the whole phase as one milestone.

## Acceptance criteria + user stories

Each feature needs:

**Acceptance criteria** — 2–5 testable bullets. Prefer the Given/When/Then shape or concrete outcomes. Examples:
- "Given a logged-in user, when they click Export, then a CSV downloads within 2 seconds"
- "Filter UI shows active filters as removable chips"
- "API rate-limit errors surface a user-visible retry prompt"

**User stories** — 1–2 stories in the form `As a <persona>, I want to <action>, so that <benefit>`. Use the personas from `target_audience.primary_persona` / `secondary_personas`.

## Feature template

```
id: feature-N (internal — not shown to user)
title: "Action-oriented title (imperative verb preferred)"
description: "2–3 sentences: what it does and roughly how it behaves from the user's perspective."
rationale: "Why this matters for <primary persona>. Cite the specific pain point or goal it addresses. If it addresses a competitor pain point, name the competitor."
priority: must | should | could | wont
complexity: low | medium | high
impact: low | medium | high
phase: 1 | 2 | 3 | 4
dependencies: ["Title of feature this depends on", ...]
acceptance_criteria: ["...", "..."]
user_stories: ["As a X, I want to Y, so that Z"]
competitor_insights: [] (optional — list competitor name + which pain point)
```

## Validation before publishing

Before moving on to GitHub publishing, sanity-check:
- At least 5 features, at most 10 (focus beats volume)
- At least one feature per phase
- No feature has an unknown dependency title
- Every feature has a non-empty rationale and at least 2 acceptance criteria
- The priority breakdown isn't all "must"
