---
name: status-report
description: Use when the user asks for a status report, daily summary, weekly recap, or "how are we doing" overview. Pulls from multiple endpoints (leaderboard, commission, recent meetings) and produces a Slack-ready narrative.
---

# Generate a portal status report

## When to use

User asks for an aggregated rollup:
- "Give me a status report"
- "How are we doing this week?"
- "Daily summary please"
- "Weekly recap for the team"
- "Brief me on yesterday"

## Tools used (typically in parallel)

- `leaderboard_get` — period rankings
- `commission_get` — target progress per SDR
- `meetings_list` — recent meeting activity (filter by date)
- `clients_list` (optional) — churn-risk clients if relevant

All from the `outbound-partners` MCP server.

## Workflow

1. **Resolve the period**:
   - "daily" / "today" / "yesterday" → 1-day window
   - "weekly" / "this week" → `this_week`
   - "monthly" → `this_month`
   - If unclear, default to `this_week`

2. **Run the queries in parallel** (single message, multiple tool calls):
   ```
   leaderboard_get { period: "this_week" }
   commission_get  { month: "<current>" }
   meetings_list   { date_from: <start>, date_to: <end>, limit: 100 }
   ```

3. **Synthesize into 4 sections**:

   ### Headline
   One sentence. Top performer, total meetings booked this period, any team-level outlier.

   ### Top performers
   Top 3 SDRs by `meetings_seen`, with their dials + show rate.

   ### Behind target
   Any SDR below 75% of their commission target (use `commission_get` data). Name + percentage + gap.

   ### Activity by day
   Mini chart-in-text: meeting count per day for the period.

   ```
   Mon ████████ 8
   Tue ██████ 6
   Wed █████████████ 13
   Thu ███████ 7
   Fri ████████████ 12
   ```

4. **Optionally include**:
   - **Pipeline value** — sum `pipeline.value` across `qualified_*` meetings in the period
   - **Churn risk** — clients with `renewal_status: "Pending"` and `renewal_date` within 60 days (separate `clients_list` call)

5. **Format for the channel the user named**:
   - "for Slack" → use Slack mrkdwn formatting (\* for bold, no markdown headers)
   - "for email" → use HTML or proper markdown
   - Default to markdown

## Example output (Slack format)

```
*Weekly recap — week of May 19, 2026*

🥇 *Harry Brown* leads with 8 meetings seen (16 booked). Team total: 28 meetings this week.

*Top 3:*
1. Harry Brown — 8 seen / 16 booked / 73% show rate
2. Sam SDR — 5 seen / 9 booked / 56% show rate
3. Asha (SA) — 4 seen / 7 booked / 57% show rate

*Behind target (May 2026):*
- Pat — 33% of 12 target (4/12)
- Riley — 50% of 6 target (3/6, SA)

*Daily breakdown:*
```
Mon ████████ 8
Tue ██████ 6
Wed ████████████ 12
Thu ██████ 6
Fri ████████ 8
```
```

## What NOT to do

- Don't write 8 paragraphs. Status reports are scanned, not read. Keep each section ≤ 5 lines.
- Don't fabricate trends ("up 23% from last week") unless you actually pulled last week's data too.
- Don't include raw UUIDs in a status report — use names.
- Don't include `not_qualified_*` meetings in the "top performer" math — those don't count for commission. The leaderboard already excludes them; trust its `meetings_seen` count.
