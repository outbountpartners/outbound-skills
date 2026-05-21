---
name: query-leaderboard
description: Use when the user asks about SDR performance, rankings, leaderboard standings, or "who's doing well/badly" for any time period — today, this week, last month, this quarter, etc.
---

# Query the SDR leaderboard

## When to use

User asks anything about SDR performance rankings or aggregated activity:
- "How's the team doing this month?"
- "Who's the top SDR this week?"
- "Show me the leaderboard"
- "Are the SA SDRs hitting their numbers?"
- "Compare Harry vs Sam this quarter"
- "Who's behind target?"

## Tool

`leaderboard_get` from the `outbound-partners` MCP server.

## How to handle the request

1. **Resolve the time period** from the user's phrasing into one of these enum values:

   `today, yesterday, this_week, last_week, this_month, last_month, this_quarter, last_quarter, this_year, last_year, all_time`

   Default to `this_week` if the user doesn't specify. Ask only if genuinely ambiguous.

2. **Resolve any filters**:
   - "UK team" / "SA team" → `location: "UK"` or `location: "SA"`
   - "Include people who've left" → `include_inactive: "true"`

3. **Call the tool** with the resolved arguments.

4. **Present the result conversationally**, not as a raw table dump. Pull out the headline:
   - Top SDR by meetings seen
   - Anyone notably above or below the pack
   - Show rate, conversion rate, total dials for context

5. **Highlight context the user is likely asking about**:
   - "How many dials did Harry make?" → cite his dials count + connect rate
   - "Who booked the most meetings?" → rank by `meetings_booked`
   - "Who's at target?" → cross-reference with `commission_get` if the user wants targets

## Example queries → responses

**User:** "How's the team doing this week?"

→ Call `leaderboard_get` with `{ "period": "this_week" }`. Then summarise: who's #1 by meetings seen, the spread, anyone notable.

**User:** "Show me the UK leaderboard for last month"

→ Call `leaderboard_get` with `{ "period": "last_month", "location": "UK" }`. Summarise just UK rows.

**User:** "All-time top performer?"

→ Call `leaderboard_get` with `{ "period": "all_time" }`. Report the #1 by meetings_seen.

## What NOT to do

- Don't try to compute rankings yourself — the server has the canonical ranking algorithm baked in (meetings_seen → meetings_booked → fewest dials).
- Don't fall back to `meetings_list` and try to aggregate. The leaderboard data is materialized and far more accurate than ad-hoc aggregation.
- Don't show all 12 numeric fields by default. Pick the 3-4 relevant to the question.

## Ranking algorithm (for your reference)

1. Most meetings seen (primary)
2. Most meetings booked (tiebreak)
3. Fewest dials (tiebreak — efficiency matters)

Each row in the response includes `rank` already, so respect that order.
