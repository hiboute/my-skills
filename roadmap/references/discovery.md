# Discovery methodology

Goal: produce a complete, opinionated understanding of the project that can feed Phase 2 feature generation. You run this **non-interactively** — infer every field from what you can read, don't ask the user.

## What to read

Scan these sources, in roughly this order of signal strength:

1. **README.md** — project purpose, target audience, features, tagline
2. **Package manifest** — `package.json` / `pyproject.toml` / `Cargo.toml` / `go.mod` — description, keywords, dependencies
3. **Docs folder** — `docs/`, `ARCHITECTURE.md`, `CONTRIBUTING.md`, any existing `ROADMAP.md`
4. **A sample of source files** — pick 5–10 representative files across the main directories to confirm tech choices and patterns
5. **Existing issues, milestones, labels** on GitHub — what's already planned or in flight
6. **Recent git history** — `git log` for activity level, contributor count for resource inference
7. **TODO/FIXME/HACK comments** — seed for `technical_debt` and `known_gaps`

If a source is missing, skip it — don't invent content. Move on.

## Fields to produce

Hold this object in memory (no need to write it to disk). The Phase 2 work reads directly from it.

```
project_name: string
project_type: web-app | mobile-app | cli | library | api | desktop-app | other
tech_stack:
  primary_language: string
  frameworks: [string]
  key_dependencies: [string]
target_audience:
  primary_persona: string
  secondary_personas: [string]
  pain_points: [string]
  goals: [string]
  usage_context: string
product_vision:
  one_liner: string
  problem_statement: string
  value_proposition: string
  success_metrics: [string]
current_state:
  maturity: idea | prototype | mvp | growth | mature
  existing_features: [string]
  known_gaps: [string]
  technical_debt: [string]
competitive_context:
  alternatives: [string]
  differentiators: [string]
  market_position: string
  competitor_pain_points: [string]
  competitor_analysis_available: boolean
constraints:
  technical: [string]
  resources: [string]
  dependencies: [string]
```

## Inference heuristics

**project_type**
- Has a binary entry point + CLI flags → `cli`
- Has a routing library + frontend framework → `web-app`
- Exposes HTTP routes only, no UI → `api`
- Published to npm/PyPI/crates.io with only programmatic entry points → `library`
- Electron / Tauri / native desktop dependency → `desktop-app`
- React Native / Swift / Kotlin project → `mobile-app`
- Default → `other`

**target_audience.primary_persona**
- CLI / developer tool → "developers" or a more specific engineering role
- SaaS web app with auth → "end users" or a specific business role based on copy
- Library → "developers building X"
- Data / analytics tool → "data analysts" or "data engineers"
- If the README explicitly names the audience, use that verbatim

**pain_points** — read README "Why X?" / "Problems solved" sections. If absent, infer from the domain: e.g., an ORM implies SQL boilerplate pain, a state manager implies prop-drilling pain.

**maturity**
- No tests, thin README, few commits → `prototype`
- Tests exist, featureset described, few releases → `mvp`
- Multiple releases, active issues, real users cited → `growth`
- Stable API, security policy, large contributor count, regular releases → `mature`

**known_gaps** — comes from three sources: TODO/FIXME comments, missing sections in docs, and features the README says are "planned" or "coming soon".

**technical_debt** — HACK / XXX / FIXME comments, suspiciously long files, files named `*.old.*` or `*.deprecated.*`, commented-out code blocks.

**constraints.resources** — git contributor count as a proxy. 1 contributor → solo project, 2–5 → small team, 6+ → team with review process. If the README names sponsors/employer, note it.

## What "good enough" looks like

- Every field populated with a non-empty value (use a reasonable inference, not "unknown")
- `target_audience.pain_points` and `target_audience.goals` each have at least 2 entries — these directly drive Phase 2 feature ideas
- `current_state.existing_features` lists at least 3 real features you can cite in the code
- `product_vision.one_liner` fits in 15 words

When those are true, stop researching and move to Phase 2.

## Competitor data (optional)

If the user provided competitor analysis:
- Fill `competitive_context.alternatives` and `competitor_pain_points` from their input
- Set `competitor_analysis_available: true`

Otherwise:
- Fill `alternatives` with 2–3 well-known tools in the same category based on domain knowledge
- Leave `competitor_pain_points` empty
- Set `competitor_analysis_available: false`

Do **not** fabricate competitor pain points. Empty is fine.
