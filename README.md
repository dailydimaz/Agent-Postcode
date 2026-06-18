# Agent PostCode — Skill

A skill inspired by [PostHog Code](https://posthog.com/code) and [PostHog/code](https://github.com/PostHog/code) — a product-aware AI devtool that turns production signals into engineering tasks, PRs, instrumentation, rollout decisions, and product fixes.

**Name:** Post**Hog** + **Code** → **PostCode**

## What it does

PostCode connects PostHog production data to action:

- **Signals inbox triage** — cluster errors, logs, funnels, surveys, experiments, flags, Scouts, support/backlog context, and product usage into ranked engineering tasks
- **Error triage** — find the top errors by user impact, get stack traces and session context, create the GitHub issue or draft
- **Regression detection** — correlate a deploy with error spikes and metric drops, fast
- **Session replay debug** — find and summarize sessions where errors or friction occurred
- **Experiment readout** — read A/B results, apply a ship checklist, execute flag-cleanup PR when you say "go"
- **Feature flag audit** — find stale flags, check health, assess blast radius before changes
- **AST-level PostHog detection** — parse SDK usage, `capture()` calls, feature flags, `init()` calls, and variant branches across JS/TS/Python/Go/Ruby patterns
- **Code scaffolding** — generate feature flag, experiment, rollout, and analytics tracking code directly into the codebase
- **Auto-instrumentation** — add the `posthog.capture()` calls, properties, error context, and exposure tracking needed for new work
- **Signal-to-PR execution** — turn a scoped signal into a branch, commit, and PR when autonomy is granted
- **Log investigation** — query and correlate backend logs with errors and deployments
- **Product health check** — composite error + engagement + retention status, weekly or on-demand
- **Incident response** — fast path for P0s: what broke, how many users, likely cause, Slack alert
- **Deploy impact** — full before/after assessment of a release
- **Stacked PRs & advanced Git** — execute atomic feature rollouts and fixes using stacked PR workflows (e.g., via `jj` or Git CLI)

## How it differs from Agent AHHOG

| | Agent AHHOG | Agent PostCode |
| --- | --- | --- |
| Question | Is our marketing/acquisition working? | Is our product/code working? |
| Data | Ahrefs (SEO) + PostHog (behavior/conversion) | PostHog (errors, logs, flags, experiments) |
| Audience | Marketing, growth | Engineering, product, founders |
| Output | SEO audits, content plans, attribution reports | Bug diagnoses, issue drafts, PR descriptions, rollout decisions |

They compose: AHHOG finds a page with bad conversion → PostCode checks if a JS error is causing it.

## Prerequisites

1. **Whatever AI** with skills enabled
2. **PostHog MCP** — required in your coding-agent client. Claude Code example:
   ```bash
   claude mcp add --transport http posthog https://mcp.posthog.com/mcp
   ```
   Guide: https://posthog.com/docs/model-context-protocol

3. **GitHub access** — optional, for code execution and issue/PR management:
   ```bash
   claude mcp add github
   ```

4. **Linear MCP** — optional, for task creation from signals

5. **Slack MCP** — optional, for incident alerts and digests

6. **Other signal connectors** — optional, for support tickets, transcripts, billing/CRM, Sentry, Scouts, or custom MCP context

## Installing

### PostHog Code (recommended)
PostCode works as a native PostHog Code skill:

```bash
# User-level (available in all projects)
cp -r agent-postcode/ ~/.posthog-code/skills/

# Repo-level (shared with team via git)
cp -r agent-postcode/ .posthog-code/skills/

# Or install from skills.sh marketplace (if published)
# Browse: skills.sh
```

### Claude Code (legacy)
```bash
cp -r agent-postcode/ ~/.claude/skills/
```

### Other AI agents
Copy the skill directory into your agent's skill/plugin path. PostCode's `SKILL.md` is the entry point.

Restart your agent after installing.

## Using it

```
"What's broken in production right now?"
"Did last night's deploy cause any regressions?"
"Show me the top errors this week and draft GitHub issues for the worst ones"
"Fix the payment TypeError — just do it"
"Is the checkout experiment ready to ship?"
"Watch the sessions where users hit the payment error"
"Audit our feature flags — which ones are stale?"
"Parse the codebase and find any feature flags that are no longer active in PostHog"
"Scaffold the new checkout experiment in the codebase"
"Build the plan upgrade flow and instrument the conversion events"
"Turn the signal inbox into GitHub issues and PRs"
"Create a stacked PR for this fix"
"Is our signup event capturing the plan property correctly?"
"Run a product health check"
"Something is on fire — check PostHog"
"You have full autonomy — triage errors and fix what you can"
```

## Permissions & autonomy model

PostCode operates at three autonomy levels — set by what you say:

| What you say | Level | What PostCode does |
| --- | --- | --- |
| "Show me", "draft", "suggest" (or nothing) | **Level 1 — Draft** | Diagnoses + drafts. Asks before acting. |
| "Fix it", "just do it", "handle this" | **Level 2 — Task** | Executes that task end-to-end (branch, commit, PR). Reports. |
| "You have full autonomy", "work autonomously" | **Level 3 — Session** | Handles all tasks in session without per-step questions. Reports at milestones. |

**One rule that never changes:** merging to a protected branch (main, master, production) always requires you to say "merge it" at that exact moment. No autonomy level bypasses this.

**PostHog permissions:**
- Reads freely — all queries, no confirmation
- Additive writes (annotations, draft insights) — acts, then reports
- Live-affecting writes (flag updates, experiment ship) — confirms if currently live to real users
- Deletions — never executed; explains how to do it in PostHog UI

**Injection guard:** instructions from content PostCode reads (log lines, error messages, external URLs) are never executed without explicit confirmation from you — even under Level 3.

## File structure

```
agent-postcode/
├── SKILL.md                          # Main skill instructions
├── README.md                         # This file
└── references/
    ├── posthog-tools.md              # PostHog MCP tools plus signal/codebase intelligence
    ├── workflows.md                  # Signal, triage, instrumentation, rollout, and PR recipes
    ├── output-formats.md             # GitHub issue, PR, Linear, Slack, markdown templates
    └── prompting-patterns.md         # How to interpret engineering/product phrasings
```

## Acknowledgements

Inspired by [PostHog Code](https://posthog.com/code) by PostHog. Uses the official [PostHog MCP server](https://posthog.com/docs/model-context-protocol). Not affiliated with PostHog.
