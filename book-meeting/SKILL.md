---
name: book-meeting
description: Use when the user wants to create a new meeting record. Walks through the required fields, validates UUIDs, and handles common edge cases (campaign for client, SDR who booked it, contact details).
---

# Book a meeting

## When to use

User wants to log a meeting:
- "Add a meeting with Acme on Friday"
- "Book a meeting for the CGI campaign"
- "Log that I have a call with Jane Doe at AstraZeneca next Tuesday"

## Tool

`meetings_create` from the `outbound-partners` MCP server.

## Required fields (server enforces NOT NULL)

The server rejects with `422 validation_failed` if any of these are missing:

| Field | Type | How to get it |
|---|---|---|
| `date` | ISO-8601 datetime | Parse from user's phrasing. Include timezone or default to UTC. |
| `client_id` | uuid | Look up via `clients_list` if user gives the client name |
| `campaign_id` | uuid | Look up via `campaigns_list` with `client_id` filter; pick the active campaign |
| `company_name` | string | The prospect company (NOT our client — the company the meeting is WITH) |
| `contact.name` | string | The prospect's full name |
| `booked_by` OR `x-acting-user` header | uuid | The SDR or admin who booked it |

## Resolving names → IDs

This is the most common stumbling block.

1. **Client name → client_id**: call `clients_list` with `search: "<name>"`. If multiple matches, ask the user.
2. **SDR name → sdr_id**: call `users_list` with `role: "sdr"` and `search: "<name>"`.
3. **Campaign for a client** → call `campaigns_list` with the `client_id`. If exactly one active campaign exists, use it. If multiple, ask which.

## Workflow

1. Parse the user's request for:
   - The prospect company name
   - The contact person
   - The date/time
   - Which client + campaign this meeting belongs to
   - The SDR who booked it (if not the acting user)

2. **Resolve every missing piece**:
   - Use `clients_list` + `campaigns_list` + `users_list` searches as needed
   - If a piece is genuinely ambiguous (multiple matches), ask one focused question

3. **Confirm with the user** before calling `meetings_create`:
   ```
   I'm about to book this meeting:
     Client / Campaign:  CGI Leeds BU / Q2 Outbound
     Company:           AstraZeneca
     Contact:           Jane Doe (VP Ops, jane@astrazeneca.com)
     Date:              2026-06-12 14:00 UTC
     SDR:               Harry Brown
   Shall I create it?
   ```

4. **Call `meetings_create`** with the resolved arguments. Set:
   - `outcome: "scheduled"` (default for future meetings)
   - `timezone` to the user's local TZ if you can infer it, otherwise `UTC+00:00`
   - `pipeline.currency` to GBP for UK clients, USD for US (ask if unclear)

5. **Surface the result**:
   - On success: confirm with the new meeting id, restate the key fields
   - On 422 validation_failed: walk through which fields failed and re-ask
   - On 409: explain the conflict and ask how to proceed

## Example

**User:** "Book a meeting with Jane Doe at AstraZeneca for the CGI Leeds campaign next Tuesday at 2pm. Harry booked it."

1. `clients_list { search: "CGI Leeds" }` → resolves to client_id
2. `campaigns_list { client_id, status: "active" }` → picks the campaign (or asks if multiple)
3. `users_list { role: "sdr", search: "Harry" }` → resolves Harry's user_id
4. Date: next Tuesday at 14:00 (assume the user's timezone — confirm if ambiguous)
5. Confirm with the user, then `meetings_create { ... }`

## What NOT to do

- Don't fabricate UUIDs. Always resolve real ones via list endpoints.
- Don't guess the campaign. If a client has 3 active campaigns, ask.
- Don't skip the confirmation step — meetings affect SDR commission calculations.
- Don't pass `client_feedback` on create (that's for after the meeting, and triggers admin emails).
