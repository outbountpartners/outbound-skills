---
name: commission-check
description: Use when the user asks about commission, monthly meeting targets, or how much they've earned — for themselves or the whole team. Works for any month, past or current.
---

# Check commission progress

## When to use

User asks about commission, targets, or earnings:
- "What's my commission this month?"
- "Show me everyone's commission for May"
- "How much did Harry earn last month?"
- "Who's hit their target?"
- "What's the target for SA SDRs?"

## Tool

`commission_get` from the `outbound-partners` MCP server.

## Two modes

### 1. Just targets (cheap, fast)

```json
{ "targets_only": "true" }
```

Returns one row per active SDR with `target_meetings` and `target_source` ("custom" or "default"). Use this when the user just wants to know what the targets are, without commission math.

### 2. Full commission progress (default)

```json
{ "month": "2026-05" }
```

Returns the same plus: `meetings_completed`, `percentage_achieved`, `commission_system` (`2026+` or `legacy`), `commission_rate_per_meeting`, `commission_tier_label`, `commission_amount`.

## How to handle the request

1. **Resolve the month**:
   - "this month" / "current month" → omit `month` (server defaults to current)
   - "May" / "May 2026" → `month: "2026-05"`
   - "last month" → calculate the previous month in YYYY-MM

2. **Resolve the scope**:
   - "Harry's commission" → resolve user name to id (call `users_list` with `search: "Harry"`), then pass `sdr_id`
   - "My commission" — if the session has an acting user via x-acting-user header, use it; otherwise ask
   - "Everyone's commission" → omit `sdr_id`

3. **Call the tool** with resolved args.

4. **Present results**:
   - Single SDR: state target, completed, percentage, tier label, total commission
   - All SDRs: rank by `commission_amount` or `percentage_achieved`, highlight outliers
   - Always include the **currency** (GBP)

## Defaults to know

- UK SDRs default to **12 meetings/month**
- SA SDRs default to **6 meetings/month**
- A custom row in `sdr_targets` overrides the default → `target_source: "custom"`

## Commission tiers (for your reference)

The server computes everything, but knowing the tiers helps you explain results:

**UK_TIERS (Jan 2026+):**
- 0–74.99% → £0
- 75–99.99% → £25/meeting
- 100–119.99% → £50/meeting
- 120–149.99% → £60/meeting
- 150%+ → £75/meeting

**SA_TIERS (Jan 2026+):**
- 0–99.99% → £0
- 100–119.99% → £20/meeting
- 120–149.99% → £30/meeting
- 150%+ → £50/meeting

**LEGACY_TIERS (pre-2026):**
- 0–49.99% → £0
- 50–99.99% → £15/meeting
- 100–119.99% → £25/meeting
- 120–149.99% → £35/meeting
- 150%+ → £50/meeting

## What NOT to do

- Don't compute commission yourself — the server applies the correct tier and excludes "not qualified" sub_statuses. Just call the tool.
- Don't fetch `meetings_list` and try to count completed meetings — the commission endpoint already filters out `not_qualified_*`, `unqualified`, `not_relevant`, `working_with_competitor` for you.
