# Agent PostCode — Claude Code Skill

A Claude Code skill inspired by [PostHog Code](https://posthog.com/code) — an AI agent that reads production signals from PostHog and turns them into engineering and product decisions: error diagnoses, experiment readouts, feature flag audits, session replay analysis, instrumentation checks, and GitHub issue / PR drafts.

**Name:** Post**Hog** + Claude **Code** → **PostCode**

## What it does

PostCode connects PostHog production data to action:

- **Error triage** — find the top errors by user impact, get stack traces and session context, create the GitHub issue (or draft to copy-paste)
- **Regression detection** — correlate a deploy with error spikes and metric drops, fast
- **Session replay debug** — find and summarize sessions where errors or friction occurred
- **Experiment readout** — read A/B results, apply a ship checklist, execute flag-cleanup PR when you say "go"
- **Feature flag audit** — find stale flags, check health, assess blast radius before changes
- **Instrumentation audit** — discover missing events, generate the `posthog.capture()` calls, commit them
- **Log investigation** — query and correlate backend logs with errors and deployments
- **Product health check** — composite error + engagement + retention status, weekly or on-demand
- **Incident response** — fast path for P0s: what broke, how many users, likely cause, Slack alert
- **Deploy impact** — full before/after assessment of a release

## How it differs from Agent AHHOG

| | Agent AHHOG | Agent PostCode |
| --- | --- | --- |
| Question | Is our marketing/acquisition working? | Is our product/code working? |
| Data | Ahrefs (SEO) + PostHog (behavior/conversion) | PostHog (errors, logs, flags, experiments) |
| Audience | Marketing, growth | Engineering, product, founders |
| Output | SEO audits, content plans, attribution reports | Bug diagnoses, issue drafts, PR descriptions, rollout decisions |

They compose: AHHOG finds a page with bad conversion → PostCode checks if a JS error is causing it.

## Prerequisites

1. **Claude Code** installed
2. **PostHog MCP** — required:
   ```bash
   claude mcp add --transport http posthog https://mcp.posthog.com/mcp
   ```
   Guide: https://posthog.com/docs/model-context-protocol

3. **GitHub MCP** — optional, untuk code execution (branch, commit, PR, merge):
   ```bash
   claude mcp add github
   ```

4. **Linear MCP** — optional, for task creation from signals

5. **Slack MCP** — optional, for incident alerts and digests

## Installing

```bash
unzip agent-postcode.zip
cp -r agent-postcode/ ~/.claude/skills/        # user-level
# or
cp -r agent-postcode/ .claude/skills/          # project-level
```

Restart Claude Code.

## Using it

```
"What's broken in production right now?"
"Did last night's deploy cause any regressions?"
"Show me the top errors this week and draft GitHub issues for the worst ones"
"Fix the payment TypeError — just do it"
"Is the checkout experiment ready to ship?"
"Watch the sessions where users hit the payment error"
"Audit our feature flags — which ones are stale?"
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
    ├── posthog-tools.md              # PostHog MCP tools for engineering/product
    ├── workflows.md                  # 10 workflow recipes
    ├── output-formats.md             # GitHub issue, PR, Linear, Slack, markdown templates
    └── prompting-patterns.md         # How to interpret engineering/product phrasings
```

## Acknowledgements

Inspired by [PostHog Code](https://posthog.com/code) by PostHog. Uses the official [PostHog MCP server](https://posthog.com/docs/model-context-protocol). Not affiliated with PostHog or Anthropic.
