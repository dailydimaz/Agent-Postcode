# Agent PostCode — Output Formats

Default output is always a **local markdown report**. Other formats are produced when the relevant MCP is connected and the user requests it (or when the workflow implies it — e.g. error-triage always offers a GitHub issue draft).

---

## Default: markdown report

Filename: `postcode-<workflow>-<YYYY-MM-DD>.md`

```markdown
# [Workflow name] — [target / site / project]
*Agent PostCode · [YYYY-MM-DD HH:MM UTC] · Data: PostHog MCP*

## TL;DR
- [Finding #1 with numbers]
- [Finding #2 with numbers]
- [Finding #3 with numbers]
- **Verdict / recommended action:** [one sentence]

## Scope
- Affected users: [N]
- Time window: [from] → [to]
- Environment: [production / staging / all]

## Findings

### [Category]
[Substantive analysis. Every claim cites a PostHog data point.]

| Metric | Value | Period | Source |
| --- | --- | --- | --- |
| ... | ... | ... | PostHog error tracking |

## Root cause assessment
[What caused this, or "cause unclear — here's the evidence."]

## Recommended actions

1. **[Action]** — [why, what to expect, estimated effort]
2. ...

## Appendix
[Error IDs, session IDs, SQL queries used, raw numbers]
```

---

## GitHub issue draft

Produced by: `error-triage`, `regression-detection`, `feature-flag-audit` (stale flags), `experiment-readout` (flag cleanup).

Requires GitHub MCP or delivered as a markdown block to copy-paste.

```markdown
## Title
[Error type]: [Short description of what's broken] — [N] users affected

## Description
**Reported by:** Agent PostCode · [date]
**PostHog issue ID:** [ID from error tracking]
**Severity:** [Critical / High / Medium / Low] — [N] users, [N] sessions affected
**First seen:** [timestamp]
**Last seen:** [timestamp]

## Reproduction context
- **URL / page:** [where it happens]
- **Browser / OS:** [if consistent]
- **User flow:** [what the user was doing — from session replay if available]

## Error details
**Exception type:** [e.g. TypeError]
**Message:** [error message]

**Stack trace (top frames):**
```
[stack trace excerpt — in-app frames only]
```

## Impact
[N] users affected across [N] sessions in the last [period].
[Link to PostHog error issue: https://app.posthog.com/...]

## Suggested fix
[Proposed code change or investigation direction]

## Acceptance criteria
- [ ] Error no longer appears in PostHog error tracking
- [ ] Affected user flow works end-to-end
- [ ] No regression in [related metric]
```

---

## PR description draft

Produced by: `experiment-readout` (ship winner + flag cleanup), `regression-detection` (hotfix), `instrumentation-audit` (add tracking).

```markdown
## Summary
[1-2 sentences: what this PR does and why.]

## Motivation
**Signal:** [PostHog data that motivated this change]
- Experiment [name] ran [N] days, [variant] won with [X]% improvement in [metric]
- Error [ID] affected [N] users since [date]
- Instrumentation gap: [event] was missing [property]

## Changes
- [File/component]: [what changed]
- [File/component]: [what changed]

## PostHog impact
- [ ] Removed feature flag `[flag-key]` (experiment concluded)
- [ ] Added `posthog.capture()` for [event]
- [ ] No PostHog changes needed

## Testing
- [ ] Tested locally
- [ ] PostHog error tracking shows no new exceptions on affected flow
- [ ] [Other test criteria]

## PostHog links
- Experiment: [link]
- Error issue: [link]
- Session replay: [link]
```

---

## Linear task draft

Produced when Linear MCP is connected and the workflow surfaces a clear engineering task.

Always confirm before creating: state title, project, and priority.

```
Title: [Error type / Feature]: [Short description]
Description:
  PostHog signal: [error ID / experiment ID / log query]
  Impact: [N] users affected
  Root cause: [diagnosis]
  Suggested fix: [direction]
  Evidence: [PostHog link]
Priority: [Urgent / High / Medium / Low]
Labels: [bug / experiment / instrumentation / tech-debt]
```

---

## Slack alert format

Produced when Slack MCP is connected and workflow is `incident-response` or `product-health-check`.

Always confirm channel before posting.

**Incident alert:**
```
🚨 *[INCIDENT / REGRESSION DETECTED]* · Agent PostCode · [time]

*What:* [Error type] — [short description]
*Impact:* [N] users, [N] sessions
*First seen:* [timestamp]
*Likely cause:* [hypothesis]

*Immediate action:* [roll back flag X / deploy hotfix / disable feature Y]

PostHog: [link] | Full report: [filename]
```

**Health check digest:**
```
📊 *Product Health · [date range]* · Agent PostCode

🟢 Errors: [N] active issues (↓[N] vs last week)
🟡 Engagement: DAU [N] (↔ flat vs last week)
🔴 Retention: D7 [X]% (↓[N]pp vs last week)

Top concern: [1 sentence]
Full report: [filename]
```

---

## GitHub execution formats (Level 2/3 autonomy)

When the user grants task or session autonomy, PostCode executes — not just drafts. These are the conventions it follows.

### Branch naming

```
postcode/<type>/<short-description>
```

Types: `fix` (bug fix), `ship` (experiment winner cleanup), `instrument` (add tracking), `cleanup` (stale flags/dead code).

Examples:
- `postcode/fix/payment-typeerror-847-users`
- `postcode/ship/checkout-cta-experiment`
- `postcode/instrument/add-plan-property-to-signup`
- `postcode/cleanup/remove-stale-dark-mode-flag`

### Commit message

```
<type>(<scope>): <short description>

<body — what and why, not how>

PostHog signal: <error ID / experiment ID / instrumentation gap>
Affected: <N users / N sessions>
```

Example:
```
fix(payment): catch TypeError when card token is undefined

Payment form crashed when Stripe returned an undefined token
on network timeout. Added null check before token processing.

PostHog signal: error/019612ab (TypeError: Cannot read properties of undefined)
Affected: 847 users, 1,203 sessions (last 7 days)
```

### PR execution checklist

Before opening a PR, PostCode verifies:
- [ ] Branch created from the correct base (main or as user specifies)
- [ ] Changes are minimal and scoped to the diagnosed issue
- [ ] Commit message includes PostHog signal reference
- [ ] PR description includes: signal, scope, fix summary, how to verify

After opening, PostCode reports:
```
PR opened: [title] (#number)
Branch: postcode/fix/...
Files changed: [N]
PostHog signal: [link]

Ready for review. To merge: tell me "merge it" or merge via GitHub.
```

### Merge confirmation prompt

PostCode always asks before merging, in this exact format:

```
Ready to merge PR #[N] "[title]" into [base branch].

Summary:
- [N] files changed
- Fixes: [PostHog error ID] — [N] users affected
- Branch: [branch name]

Type "merge it" to proceed, or review the PR first: [link]
```

PostCode merges only after receiving an explicit confirmation — "merge it", "yes merge", "go ahead and merge", or equivalent. It does not infer merge intent from general session autonomy.

---

- Default always → markdown report (saved locally)
- Error/regression with GitHub MCP → also offer GitHub issue draft
- Experiment ship with GitHub MCP → also offer PR description for flag cleanup
- Incident with Slack MCP → also post Slack alert (confirm channel first)
- Engineering task with Linear MCP → also offer Linear task (confirm before creating)
- Multiple findings → markdown report is the source of truth; other formats are excerpts from it
