# outbound-skills

**Claude Skills for the Outbound Partners Portal.** Higher-level than the raw MCP tools — these bundle workflow + validation + conversational patterns for common tasks.

## Skills in this repo

| Skill | What it does |
|---|---|
| [`query-leaderboard`](./query-leaderboard/) | Conversational queries about SDR performance, rankings, period comparisons |
| [`commission-check`](./commission-check/) | "What's my commission this month?" / "Show me everyone's commission" |
| [`book-meeting`](./book-meeting/) | Walk through creating a meeting with proper name → UUID resolution |
| [`update-meeting-outcome`](./update-meeting-outcome/) | Record outcome + sub_status + feedback after a meeting; handles the admin-email-on-feedback hook |
| [`client-onboarding`](./client-onboarding/) | End-to-end multi-step: create client → campaign → invite user → set target |
| [`deactivate-user`](./deactivate-user/) | Safe user deactivation with confirmation + reason capture |
| [`status-report`](./status-report/) | Slack/email-ready weekly or daily team summary |

## Prerequisites

Each skill assumes the [`outbound-partners` MCP server](https://github.com/outbountpartners/outbound-mcp) is connected to your Claude session and your API key has the required scopes:

| Scope | Needed by |
|---|---|
| `meetings:read` | query-leaderboard, status-report, book-meeting (lookup) |
| `meetings:write` | book-meeting, update-meeting-outcome |
| `clients:read` | book-meeting, client-onboarding, status-report |
| `clients:write` | client-onboarding |
| `campaigns:read` | book-meeting, status-report |
| `campaigns:write` | client-onboarding |
| `users:read` | book-meeting, status-report, deactivate-user (lookup) |
| `users:write` *(super-admin only)* | client-onboarding, deactivate-user |
| `leaderboard:read` | query-leaderboard, status-report |
| `commission:read` | commission-check, status-report |

Issue keys with the appropriate scopes from the `/api` page inside the portal (admin-only).

## Install

### Install all skills at once

```bash
claude plugin install outbountpartners/outbound-skills
```

### Install a single skill

```bash
claude plugin install outbountpartners/outbound-skills/query-leaderboard
```

### Or clone manually

```bash
git clone https://github.com/outbountpartners/outbound-skills.git
# Then point Claude at the folder(s) you want active.
```

## Authored using

[`skill-creator@claude-plugins-official`](https://github.com/anthropics/skills) — the official Anthropic skill-creator plugin. Each `SKILL.md` follows the standard frontmatter (`name`, `description`) so the skill is discoverable across Claude surfaces (Code, Desktop, API).

## How skills interact with the MCP server

```
User: "How's the team doing this week?"
       │
       ▼
Claude loads SKILL.md (query-leaderboard) into context
       │
       ▼
Skill instructs Claude:
  - "Default to this_week if user doesn't specify"
  - "Call leaderboard_get tool"
  - "Format conversationally, highlight top 3"
       │
       ▼
Claude invokes outbound-mcp tool: leaderboard_get { period: "this_week" }
       │
       ▼
MCP server proxies to Portal API
       │
       ▼
Portal API returns ranked SDRs
       │
       ▼
Skill guides Claude's response shaping
```

## Contributing a new skill

1. Pick a recurring workflow that isn't covered.
2. Create a folder `<skill-name>/` with a `SKILL.md` inside.
3. Frontmatter must include `name` and `description`. The description is what Claude uses to decide when to activate the skill — make it specific (e.g., "Use when the user asks about...").
4. Document the tool(s) the skill uses + the workflow Claude should follow.
5. Open a PR.

## Versioning

These are living documents. We don't tag versions yet — the latest `main` is what gets used. If a skill changes incompatibly with how a particular tool behaves, we'll update both atomically.

## Maintainer

Rai — rai@outboundpartners.com
