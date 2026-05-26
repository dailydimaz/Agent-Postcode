# PostHog MCP — Engineering & Product Tool Reference

This is the PostCode view of PostHog MCP — focused on the tools engineers, DevOps, and product managers need. For the marketing view (funnels, cohorts, conversions), see Agent AHHOG.

Tool names use bare suffixes — prefix with `PostHog:` or `mcp__posthog__` depending on your client.

---

## Start here every session

| Tool | When to call |
| --- | --- |
| `read-data-schema` | **Always first.** Discovers real event names, properties, and data structure. Never assume. |
| `project-get` | Confirm which project you're working in |
| `projects-get` | List available projects (multi-project orgs) |
| `switch-project` | Change active project |

---

## Error tracking — the core of PostCode

Use these for any "something is broken" or "bug report" request.

| Tool | Returns | Use when |
| --- | --- | --- |
| `query-error-tracking-issues-list` | All active error issues with occurrence count, user count, session count, last seen | **First call for any bug/error request.** Sort by `occurrences` or `users` to find the worst offenders |
| `query-error-tracking-issue` | Single issue: status, impact counts, top stack frame, first/last seen, assignee | Drill into a specific issue after listing |
| `query-error-tracking-issue-events` | Sampled `$exception` events: stack traces, browser, OS, URL, `$session_id` | Get actual stack traces and reproduction context |
| `error-tracking-issues-partial-update` | Update issue status (active → resolved → suppressed) | Mark fixed after a deploy |

**Workflow pattern for a bug report:**
```
query-error-tracking-issues-list (find issue)
  → query-error-tracking-issue (scope: users, sessions, first seen)
  → query-error-tracking-issue-events (stack trace, reproduction context)
  → query-session-recordings-list with $session_id (watch the user's session)
  → draft GitHub issue with all evidence
```

**Parameters that matter:**
- `status`: `active` (default), `resolved`, `suppressed`, `all`
- `orderBy`: `occurrences`, `users`, `sessions`, `last_seen`, `first_seen`
- `dateRange`: narrow to recent window for regression detection
- `url`: filter to a specific page
- `searchQuery`: free-text search across exception types and values
- `library`: filter by SDK (e.g. `posthog-js` for frontend only)

---

## Logs — for backend and infrastructure investigation

| Tool | Returns | Use when |
| --- | --- | --- |
| `query-logs` | Log entries filtered by severity, service, date, search term | "What do the logs say about X" |
| `logs-count` | Scalar count of matching logs | Quick "how many errors in last hour" |
| `logs-count-ranges` | Bucketed log counts over time | Spot a spike visually |
| `logs-sparkline-query` | Time-bucketed volume by severity or service | Log volume trend |
| `logs-services-create` | Top 25 services by log volume | Which service is noisiest |
| `logs-attributes-list` | Available log attribute keys | Discover filterable dimensions |
| `logs-attribute-values-list` | Values for a specific attribute | Find valid filter values |

**Query pattern:**
```
logs-services-create → identify noisy service
  → query-logs (filter by service + severity=error + date range)
  → logs-count-ranges (confirm spike timing)
  → correlate spike timing with deploy or flag change
```

---

## Session recordings — qualitative debugging

| Tool | Returns | Use when |
| --- | --- | --- |
| `query-session-recordings-list` | List recordings with filters (date, URL, events, session IDs) | Find sessions where error occurred |
| `session-recording-get` | Recording metadata: duration, interactions, console logs, person | Inspect a specific session |
| `session-recording-summarize` | AI summary of what the user did and where they struggled | **Slow (~5 min first run)** — warn user. Use when you need to understand UX context quickly |

**Getting session IDs:** `query-error-tracking-issue-events` returns `$session_id` on events. Pass these directly to `query-session-recordings-list` with `session_ids` parameter.

**Warning on summarize:** first-time summaries take ~5 minutes. Cached ones are instant. Always warn the user before calling.

---

## Feature flags — rollout management

| Tool | Returns | Use when |
| --- | --- | --- |
| `feature-flag-get-all` | All flags with status, rollout, variants | Flag inventory |
| `feature-flag-get-definition` | A single flag's full config | Inspect rollout conditions |
| `feature-flags-status-retrieve` | Health and evaluation status | "Is this flag working correctly" |
| `feature-flags-evaluation-reasons-retrieve` | Why a flag evaluated a certain way for a user | Debug "why isn't the flag on for this user" |
| `feature-flags-test-evaluation-create` | Test flag evaluation for a specific user/time | Reproduce a flag state at a past moment |
| `feature-flags-user-blast-radius-create` | How many users a flag change would affect | Before changing rollout — check impact |
| `update-feature-flag` | Modify a flag (rollout %, conditions, variants) | **Tier 3 — confirm if live** |
| `scheduled-changes-create` | Schedule a future flag change | Gradual rollout scheduling |

**Audit workflow:**
```
feature-flag-get-all → list all flags
  → feature-flags-status-retrieve (for flags with issues)
  → feature-flags-evaluation-reasons-retrieve (debug specific user)
  → feature-flags-user-blast-radius-create (before any change)
  → update-feature-flag (Tier 3 — confirm if live)
```

---

## Experiments — A/B test management

| Tool | Returns | Use when |
| --- | --- | --- |
| `experiment-list` | All experiments with status (draft/running/stopped) | Inventory |
| `experiment-get` | Full experiment config: variants, metrics, flag key | Inspect a test |
| `experiment-results-get` | Statistical results: conversion, credible intervals, winner | "Did the experiment win?" |
| `experiment-timeseries-results` | Results over time | Check for novelty effect, peeking |
| `experiment-launch` | Launch a draft experiment | **Tier 3 — confirm** |
| `experiment-pause` / `experiment-resume` | Pause/resume | **Tier 3 — confirm** |
| `experiment-end` | Stop the experiment | **Tier 3 — confirm** |
| `experiment-ship-variant` | Roll the winner to 100% | **Tier 3 — confirm; high-impact** |

**Readout workflow:**
```
experiment-get (config: variants, metrics, sample size, start date)
  → experiment-results-get (statistical summary: winner, credible intervals, exposure)
  → experiment-timeseries-results (check stability over time — is the result consistent?)
  → synthesize: ready to ship? or need more data?
  → if shipping: experiment-ship-variant (Tier 3 — confirm) + draft PR for flag cleanup
```

**Shipping criteria to check:**
- Sufficient exposure (both variants have meaningful sample)
- Credible interval doesn't cross zero for primary metric
- Result is stable over time (no novelty spike early then regression)
- No guardrail metric is degraded

---

## Behavioral queries — regression and health detection

| Tool | Returns | Use when |
| --- | --- | --- |
| `query-trends` | Event counts over time, with breakdowns | Detect volume spikes/drops post-deploy |
| `query-funnel` | Step-by-step conversion | "Did the deploy break checkout?" |
| `query-retention` | Cohort retention grid | "Is retention holding after the change?" |
| `query-paths` | Navigation flows | "Where do users go after hitting the error?" |
| `execute-sql` | Raw HogQL query | Custom correlation queries |

**Regression detection pattern:**
```
query-trends (event: key action, date: last 14 days, interval: day)
  → look for drop after deploy date
  → if drop: query-funnel to find which step broke
  → if error-related: query-error-tracking-issues-list filtered to same date
```

**HogQL for deploy correlation:**
```sql
SELECT
  toDate(timestamp) as day,
  count() as errors,
  uniq(distinct_id) as users
FROM events
WHERE event = '$exception'
  AND timestamp >= now() - interval 14 day
GROUP BY day
ORDER BY day
```

---

## Instrumentation audit

| Tool | Returns | Use when |
| --- | --- | --- |
| `read-data-schema` | All events, properties, property values | "Is X event being tracked?" |
| `event-definitions-list` | Event definitions with metadata | Check if an event exists and when last seen |
| `execute-sql` | Raw query on events table | Spot-check event shape and property values |

**Audit workflow:**
```
read-data-schema (discover what's actually being tracked)
  → compare against expected events for the feature
  → execute-sql: check a sample of recent events for correct properties
  → identify gaps → generate posthog.capture() code snippet for missing events
```

---

## Annotations — marking deploys and incidents

| Tool | Use when |
| --- | --- |
| `annotation-create` | Mark a deploy, flag change, or incident on the timeline **(Tier 2 — do then report)** |
| `annotations-list` | See all existing markers |

Always annotate a deploy or major flag change — it makes every future chart immediately readable.

---

## Insights and dashboards — engineering views

| Tool | Use when |
| --- | --- |
| `insight-create` | Build a saved error rate / uptime / event volume trend **(Tier 2)** |
| `dashboard-create` | Create an engineering health dashboard **(Tier 2)** |
| `dashboard-insights-run` | Refresh a dashboard | Get current numbers on an existing board |

---

## Key SQL patterns (HogQL)

**Error rate over time:**
```sql
SELECT toDate(timestamp) as day, count() as errors
FROM events WHERE event = '$exception'
AND timestamp > now() - interval 30 day
GROUP BY day ORDER BY day
```

**Top error types:**
```sql
SELECT properties.$exception_type, count() as n, uniq(distinct_id) as users
FROM events WHERE event = '$exception'
AND timestamp > now() - interval 7 day
GROUP BY 1 ORDER BY 2 DESC LIMIT 20
```

**Events missing expected property:**
```sql
SELECT count() as missing
FROM events WHERE event = 'your_event'
AND NOT JSONHas(properties, 'expected_property')
AND timestamp > now() - interval 7 day
```

**Funnel drop after specific date:**
```sql
SELECT
  countIf(event = 'step_1') as step1,
  countIf(event = 'step_2') as step2,
  round(countIf(event = 'step_2') / countIf(event = 'step_1') * 100, 1) as conversion_pct
FROM events
WHERE timestamp > '2026-05-01'
AND distinct_id IN (SELECT distinct_id FROM events WHERE event = 'step_1' AND timestamp > '2026-05-01')
```
