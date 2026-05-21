---
name: update-meeting-outcome
description: Use after a meeting has happened to record its outcome, sub_status (qualification result), and any follow-up notes. Also handles client feedback submission which triggers admin email alerts.
---

# Update a meeting outcome

## When to use

User wants to record what happened after a meeting:
- "Mark Friday's AstraZeneca meeting as completed"
- "The Acme meeting didn't qualify — set sub_status to not_qualified_no_budget"
- "Add client feedback on meeting abc-123"
- "That meeting got rescheduled to next week"

## Tool

`meetings_update` (or `meetings_submit_feedback` for the feedback-only convenience case) from the `outbound-partners` MCP server.

## Outcome enum (server-enforced)

```
scheduled, pending, completed, cancelled, no_show, rescheduled, to_be_rescheduled
```

If the user says something else ("happened" → `completed`, "didn't show" → `no_show`, "moved" → `rescheduled`), map to the closest enum and confirm.

## Sub-statuses

`sub_status` is a `string[]`. Common values seen in production:

| Code | Meaning |
|---|---|
| `qualified_pipeline` | Qualified, in pipeline |
| `qualified_long_term` | Qualified but long sales cycle |
| `not_qualified_no_budget` | Not qualified — no budget |
| `not_qualified_wrong_role` | Wrong person |
| `not_qualified_no_need` | No fit |
| `not_relevant` | Not relevant at all |
| `working_with_competitor` | Already using a competitor |
| `unqualified` | Generic disqualification |

**Important:** sub_status values starting with `not_qualified_*` (or the legacy `unqualified` / `not_relevant` / `working_with_competitor`) are **excluded from commission calculations**. Confirm with the user that this is what they want before applying.

## Workflow

1. **Resolve the meeting**:
   - User gives an id directly → use it
   - User describes it ("the AstraZeneca one") → call `meetings_list` with filters (company_name, date_from/date_to, sdr_id) and confirm the match

2. **Map the user's intent**:
   - "Completed and qualified" → `outcome: "completed"`, `sub_status: ["qualified_pipeline"]`
   - "Didn't qualify" → ask which sub-reason; never silently pick one
   - "Rescheduled" → `outcome: "rescheduled"`; optionally update `date` if user gave a new one
   - "Client gave feedback: …" → use `meetings_submit_feedback` (shorter form)

3. **Confirm before applying** if the change affects commission:
   > "Setting sub_status to `not_qualified_no_budget` will exclude this meeting from Harry's commission calculation. Confirm?"

4. **Call the tool**:
   - For outcome/sub_status/notes updates: `meetings_update { id, outcome, sub_status, outcome_notes }`
   - For feedback only: `meetings_submit_feedback { id, message }`

5. **Surface the result**:
   - On success: restate the new state
   - If client_feedback was set: mention that the admin email notification was triggered

## Client feedback gotcha

Setting `client_feedback` from empty → non-empty fires `notify-feedback-admins` automatically (emails all active admins). This is intentional and matches the UI. Tell the user.

Setting `client_feedback` when it already has content does NOT fire a notification.

## Example

**User:** "Mark the AstraZeneca meeting on June 12 as completed. Harry qualified them, decent pipeline."

1. `meetings_list { company_name search via list with date filter }` → pick the meeting id
2. Confirm: "Meeting with AstraZeneca on 2026-06-12, contact Jane Doe — set to completed + qualified_pipeline?"
3. `meetings_update { id, outcome: "completed", sub_status: ["qualified_pipeline"], outcome_notes: "Decent pipeline" }`

## What NOT to do

- Don't silently apply `not_qualified_*` without user confirmation — commission impact.
- Don't submit feedback as the SDR if it's coming from the client. Feedback should be entered as the client (via `x-acting-user` header) to maintain the audit trail.
- Don't update outcome on a future-dated meeting — that's a workflow smell. Confirm the date is in the past first.
