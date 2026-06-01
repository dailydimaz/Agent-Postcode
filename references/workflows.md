# Agent PostCode — Workflow Recipes

Each recipe: triggers, inputs needed, steps, synthesis, output.

---

## signals-inbox-triage

**Triggers:** "what should we work on", "signals inbox", "turn signals into tasks", "prioritize product issues", "find engineering work from PostHog"

**Inputs needed:** time window (default: last 7 days), product area if known, connected context sources (GitHub/Linear/support/Slack/transcripts/billing) if relevant.

**Steps:**
1. `read-data-schema` + `project-get` → confirm the project and real event names
2. In parallel where possible: `query-error-tracking-issues-list`, `query-logs`, `query-funnel`/`query-trends`, `experiment-list`, `feature-flag-get-all`, surveys
3. Pull connected context only when useful: GitHub issues/PRs, Linear backlog, support tickets, Slack incidents, call transcripts, billing/CRM records, Sentry
4. Cluster related signals into tasks: same page, feature, flag, error type, funnel step, or customer segment
5. For each task, score impact, urgency, confidence, effort, and suggested owner surface

**Synthesis:**
- Prefer one task per root cause, not one task per symptom
- Keep support/backlog context secondary unless PostHog shows product impact
- Separate "fix now" from "instrument/learn more" when the evidence is weak

**Output:** Ranked signal inbox report. For the top 1-3 tasks: title, evidence, impact, likely root cause, recommended action, GitHub/Linear task draft, and PR path if autonomy was granted.

---

## error-triage

**Triggers:** "what's broken", "top errors", "bug report", "what errors do we have", "users are complaining"

**Inputs needed:** time window (default: last 7 days), environment filter if relevant.

**Steps:**
1. `query-error-tracking-issues-list` (status: active, orderBy: users, dateRange: last 7 days) → get top 10
2. For top 3 by user count: `query-error-tracking-issue` → scope (users, sessions, first seen)
3. For the #1 issue: `query-error-tracking-issue-events` → stack trace, browser, OS, URL
4. If `$session_id` present: `query-session-recordings-list` with those IDs → optional `session-recording-summarize`

**Synthesis:**
- Rank by user impact (users affected × recency)
- Identify if any are regressions (first seen = recent deploy date)
- Note any clustering (same page, same browser, same release)

**Output:** Error triage report. TL;DR = top 3 issues with user count and severity. For #1: full reproduction context. Prioritized fix list. GitHub issue draft for #1.

---

## log-investigation

**Triggers:** "check the logs", "what do logs say about X", "backend errors", "server logs"

**Inputs needed:** service name (if known), search term or error keyword, time window.

**Steps:**
1. `logs-services-create` → identify top services by error volume
2. `query-logs` filtered by service + severity=error (or warn) + search term
3. `logs-count-ranges` → confirm timing of spike
4. `logs-sparkline-query` broken by severity → visualize trend

**Synthesis:**
- Is this a new pattern or existing noise?
- Is it isolated to one service or spreading?
- Does timing correlate with a deploy, flag change, or traffic spike?

**Output:** Log investigation report. Timeline of the spike, top log patterns, root service, correlation hypothesis.

---

## regression-detection

**Triggers:** "did the deploy break anything", "something changed after the deploy", "regression check", "release impact"

**Inputs needed:** deploy date/time, optionally the deploy annotation or commit.

**Steps:**
1. `annotations-list` → find any deploy annotation near the date
2. `query-error-tracking-issues-list` filtered by `first_seen` after deploy date → new errors introduced
3. `query-trends` for key events (page views, signups, purchases, API calls) — compare 7 days before vs after deploy
4. `query-funnel` for primary conversion funnel — before vs after
5. If errors found: `query-error-tracking-issue-events` for stack trace

**Synthesis:**
- New errors: likely regressions (correlation with deploy)
- Volume drops in key events: possible functional regression
- Funnel drop at specific step: points to which feature broke

**Output:** Regression report. Green/amber/red verdict. New errors list. Metric delta table. Recommended action (ship/roll back/investigate further).

---

## session-replay-debug

**Triggers:** "watch the session", "show me what the user saw", "replay for this error", "session recording"

**Inputs needed:** session ID (from error event), or enough context to filter (user email, page URL, date).

**Steps:**
1. If session ID known: `session-recording-get` → metadata
2. If not known: `query-session-recordings-list` filtered by URL or person or date → find candidate sessions
3. For sessions linked to an error: `query-error-tracking-issue-events` → extract `$session_id` first
4. `session-recording-summarize` → AI summary of what happened (warn user: ~5 min first run)

**Synthesis:**
- What did the user do before the error?
- Did they encounter friction (rage clicks, confusion, dead ends)?
- Is this a code bug or a UX/content problem?

**Output:** Session analysis report. User journey narrative. Root cause assessment (code vs UX). Recommended fix type (engineering vs design vs copy).

---

## feature-flag-audit

**Triggers:** "check our flags", "flag rollout status", "is this flag safe to ship", "which flags are active", "stale flags"

**Inputs needed:** optionally a specific flag key, or "all flags".

**Steps:**
1. `feature-flag-get-all` → inventory: active flags, rollout %, variants
2. For flagged concerns: `feature-flags-status-retrieve` → health check
3. `feature-flags-user-blast-radius-create` for any flag the user wants to change → scope impact before touching
4. `experiment-list` → cross-reference flags with running experiments (experiment flags shouldn't be touched manually)

**Synthesis:**
- Stale flags: active flags older than 90 days that are at 100% rollout — candidates for cleanup
- Risk flags: flags with complex conditions or large blast radius
- Experiment flags: do not modify independently of the experiment

**Output:** Flag audit report. Stale flag list (safe to remove). Risk flags (review before changing). Experiment-linked flags (hands off). Cleanup PR description for stale flags if GitHub is connected.

---

## ast-flag-detection

**Triggers:** "find flags in the codebase", "audit flags in code", "which flags are still referenced", "remove stale flags"

**Inputs needed:** codebase path, optional flag key or product area.

**Steps:**
1. Discover SDK usage with structural tooling when available. If `@posthog/enricher` or a local equivalent exists, prefer it; it detects SDK calls, `capture()` events, `init()` calls, feature flag checks, assignments, and variant branches across JS/TS/Python/Go/Ruby.
2. If no parser exists, use `rg` to find PostHog SDK imports, `capture`, `getFeatureFlag`, `isFeatureEnabled`, `feature_enabled`, `posthog`, and local wrapper names, then inspect matches manually.
3. `feature-flag-get-all` + `experiment-list` → map code references to live flags and experiments.
4. Classify each flag: active rollout, experiment-owned, fully rolled out/stale, unknown in PostHog, dead code candidate.
5. For stale flags, inspect variant branches and tests before proposing removal.

**Synthesis:**
- Never delete a flag because text search missed it; validate references and rollout state
- Do not manually change experiment-owned flags outside the experiment workflow
- For unknown flags, distinguish "code references missing PostHog flag" from "dynamic flag key can't be resolved"

**Output:** Code flag map with file/line references, PostHog status, staleness classification, cleanup risk, and PR plan for safe removals.

---

## auto-instrumentation

**Triggers:** "build this with tracking", "add PostHog instrumentation", "instrument this feature", "track this event", "make sure analytics is included"

**Inputs needed:** feature or user flow, expected success/failure events, rollout needs, privacy constraints.

**Steps:**
1. `read-data-schema` → learn existing event naming, property conventions, and identity/group usage
2. Inspect current SDK initialization and wrapper utilities in the codebase
3. Check feature flags/experiments if the change is gated or A/B tested
4. Implement the product change and instrumentation together: capture success/failure events, relevant properties, exception context, and flag/variant exposure where appropriate
5. Add or update tests for the behavior and tracking calls
6. Define the PostHog verification query: event count, property shape, error rate, or funnel step after deploy

**Synthesis:**
- Reuse local analytics wrappers and event naming conventions
- Avoid collecting secrets, PII, or high-cardinality blobs
- Prefer stable property names over dumping component state

**Output:** Implementation/PR summary with event names, properties, flag/experiment keys, tests run, and the PostHog query to verify after ship.

---

## code-scaffolding

**Triggers:** "scaffold the experiment", "generate feature flag code", "add rollout code", "set up the A/B test"

**Inputs needed:** feature name, flag key or experiment name, variants, target audience, success metric.

**Steps:**
1. Inspect existing feature flag and experiment patterns in the repo
2. `feature-flag-get-definition` or `feature-flag-get-all` → confirm flag shape, rollout, variants
3. `experiment-get` if tied to an existing experiment; otherwise draft the experiment/flag config before code
4. Add the flag/variant branch in code with a safe fallback
5. Add exposure and success/failure instrumentation using existing analytics wrappers
6. Update tests and PR description with rollout and cleanup plan

**Synthesis:**
- Feature flag code must fail closed or preserve the current experience
- Multivariate experiments need explicit variant handling and a default branch
- Every scaffolded experiment should include a future cleanup note

**Output:** Code change or PR draft with flag key, variant behavior, instrumentation, tests, rollout plan, and cleanup criteria.

---

## experiment-readout

**Triggers:** "did the experiment win", "is the test ready to ship", "A/B results", "experiment summary", "should we ship the variant"

**Inputs needed:** experiment name or ID (use `experiment-list` to find if not given).

**Steps:**
1. `experiment-list` → find the experiment
2. `experiment-get` → config: variants, metrics, start date, sample size target
3. `experiment-results-get` → statistical summary: winner, credible intervals, exposure per variant
4. `experiment-timeseries-results` → stability check: is the result consistent over the last week or volatile?

**Synthesis — ship checklist:**
- [ ] Both variants have sufficient exposure (≥ 100 users each, ideally more)
- [ ] Primary metric credible interval doesn't cross zero for test variant
- [ ] Result is stable (not a novelty spike)
- [ ] No guardrail metric (error rate, bounce, retention) is materially degraded
- [ ] Experiment has been running long enough (typically ≥ 2 weeks)

**Verdict:** SHIP / WAIT (needs more data) / DON'T SHIP (variant lost or guardrail hit) / INVESTIGATE (anomalous result).

**Output:** Experiment readout. Statistical summary table. Ship checklist with pass/fail. Recommended action. If SHIP: `experiment-ship-variant` (Tier 3 — confirm) + PR description for removing the flag and dead code.

---

## instrumentation-audit

**Triggers:** "is X tracked", "check our analytics setup", "missing events", "are we capturing Y", "instrumentation review"

**Inputs needed:** feature name or list of expected events.

**Steps:**
1. `read-data-schema` → current event inventory
2. Compare against expected events for the feature/area
3. For events that exist: `execute-sql` spot-check — recent sample, check properties are correct
4. For events that should exist but don't: identify the gap and generate the fix

**Synthesis:**
- Missing events → code snippet with `posthog.capture('event_name', { properties })`
- Events exist but wrong properties → code snippet with correct property schema
- Events exist and correct → confirm and document

**Output:** Instrumentation audit. Green/red per expected event. For each gap: the `posthog.capture()` call needed and where in the codebase to add it.

---

## product-health-check

**Triggers:** "how is the product doing", "error rate", "are users happy", "product status", "weekly health check"

**Inputs needed:** time window (default: last 7 days vs prior 7 days).

**Steps (parallel where possible):**
1. `query-error-tracking-issues-list` → active error count, top 3 by users
2. `query-trends` for key events: DAU, core action, conversion — current vs prior period
3. `query-retention` (last 4 weeks) → is retention stable?
4. `logs-count-ranges` → error log volume trend
5. `experiment-list` (status: running) → any experiments currently live that could be skewing metrics

**Synthesis:**
- Error health: errors up/down? Any critical new issues?
- Engagement health: DAU and core actions up/down?
- Retention health: stable, improving, degrading?
- Confounders: are running experiments making the metrics harder to read?

**Output:** Health dashboard report. Color-coded status per dimension (🟢🟡🔴). Key changes vs prior period. Top concern + recommended action.

---

## incident-response

**Triggers:** "something is on fire", "site is down", "critical error", "P0", "production incident"

**Inputs needed:** what's broken (error message, URL, feature name), when it started.

**Priority:** speed over completeness. Get to a hypothesis fast.

**Steps (fast path):**
1. `query-error-tracking-issues-list` (orderBy: last_seen, dateRange: last 2 hours) → what just appeared?
2. `logs-count-ranges` (last 2 hours, severity: error) → volume spike?
3. `query-trends` (DAU or core action, last 24h, hourly) → when did it start dropping?
4. `annotations-list` → any recent deploy or flag change?
5. `feature-flag-get-all` → any flag recently changed?

**Synthesis (in < 2 minutes of analysis):**
- What broke: error type and page
- When: first seen timestamp
- Likely cause: correlate with deploy/flag change time
- Blast radius: users affected

**Output:** Incident brief (< 5 bullets). Likely cause. Immediate recommendation (roll back flag? deploy hotfix? disable feature?). Slack alert draft if Slack MCP connected.

---

## deploy-impact

**Triggers:** "did the release go well", "impact of the deploy", "post-deploy check", "release report"

**Inputs needed:** deploy date/time, version or commit (optional).

**Steps:**
1. `annotation-create` for the deploy date if not already marked (Tier 2)
2. `query-error-tracking-issues-list` (first_seen: after deploy) → new errors
3. `query-trends` (key events, 7 days before vs 7 days after, or hourly around deploy)
4. `query-funnel` (primary funnel, before vs after)
5. `logs-count-ranges` (around deploy window)

**Synthesis:**
- New errors introduced? How many users?
- Key metric movement: positive, neutral, or regression?
- Funnel impact: better, same, or broke a step?

**Verdict:** GREEN (clean deploy) / AMBER (minor issues, monitor) / RED (regression, consider rollback).

**Output:** Deploy impact report. Verdict with evidence. New error list if any. Metric delta table. Recommended next action.

---

## signal-to-pr

**Triggers:** "fix it", "create a PR from this signal", "open the PR", "handle this bug", "implement the fix"

**Inputs needed:** a scoped signal or task, target repo/branch, autonomy level, expected verification.

**Steps:**
1. Confirm the signal evidence: error issue, log spike, funnel drop, experiment readout, or instrumentation gap
2. Inspect the relevant code path and recent changes
3. Create a branch using the output-format branch naming convention if writing is authorized
4. Implement the smallest fix that addresses the diagnosed root cause; include instrumentation when behavior changes
5. Run focused tests/checks and, where possible, add or update tests
6. Commit with the PostHog signal reference and open/draft a PR with evidence, scope, verification, and follow-up monitoring

**Synthesis:**
- Do not turn a vague error list into a broad PR; split into one PR per root cause
- If the cause remains unclear, open an investigation task instead of guessing in code
- Protected-branch merges always require explicit per-merge confirmation

**Output:** PR or PR draft with PostHog evidence, files changed, tests run, rollout/instrumentation notes, and post-ship signal to monitor.

---

## stacked-prs

**Triggers:** "create a stacked PR", "split this into stacked PRs", "manage branches for this fix", "ship this in layers"

**Inputs needed:** target base branch, desired split, dependency order, verification for each layer.

**Steps:**
1. Split the work by independently reviewable behavior: instrumentation/schema first, feature flag scaffolding second, behavior change third, cleanup last
2. Use the repo's established Git workflow (`git`, `jj`, or local scripts) without rewriting unrelated user changes
3. Each stack layer gets its own PostHog signal reference, tests, and PR description
4. Link dependent PRs and state the merge order
5. Re-run checks after rebases/restacks

**Synthesis:**
- Stacks are for reducing review risk, not hiding uncertainty
- Keep each PR deployable or clearly gated behind a flag
- Cleanup PRs should only follow after the experiment/rollout signal is conclusive

**Output:** Stack plan or opened PR stack: branch names, dependency order, evidence per PR, tests, and merge confirmation boundary.
