# Agent PostCode — Workflow Recipes

Each recipe: triggers, inputs needed, steps, synthesis, output.

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
