---
name: client-onboarding
description: Use when onboarding a new client end-to-end. Multi-step write flow — creates the client, the initial campaign, invites the client-side user, and sets the first month's meeting target.
---

# Onboard a new client end-to-end

## When to use

Multi-step write request from the user:
- "Onboard Acme as a new client"
- "Set up a new client called Acme — campaign Q3 Outbound, contact Jane at jane@acme.com"
- "Create the new XYZ Corp account with their Q3 campaign"

## Tools used

In order:
1. `clients_create` — the client record
2. `campaigns_create` — the initial campaign
3. `users_invite` — invite the client-side user
4. `campaigns_update` — set the first monthly target if user provided one

All from the `outbound-partners` MCP server. **Requires** `clients:write`, `campaigns:write`, and `users:write` scopes on your API key. `users:write` is super-admin only.

## Required information

You'll need:

| For… | Required |
|---|---|
| Client | `name`, `industry`, `primary_contact: { name, email, phone }` |
| Campaign | `name`, `start_date`, `end_date`, `overall_target` |
| User | `name`, `email`, `role: "client_user"`, `client_id` (resolves from step 1) |
| Monthly target | YYYY-MM key + integer (e.g. `"2026-07": 20`) |

## Workflow

1. **Gather everything** before any writes. Ask only for what you don't have. Group questions:
   ```
   I'll set up Acme. To do this in one go, I need:
   - Industry (e.g. SaaS, Fintech, Healthcare)
   - Primary contact: name, email, phone
   - Campaign name + date range
   - Initial overall meeting target
   - Client-side user to invite: name + email
   ```

2. **Confirm the full plan** with the user before any writes:
   ```
   I'll create:
     Client:    Acme Ltd (SaaS, primary contact: Joe Bloggs / joe@acme.com)
     Campaign:  Q3 Outbound (2026-07-01 → 2026-09-30, target 60 meetings)
     Invite:    Joe Bloggs (joe@acme.com) as client_user
     Target:    20 meetings for 2026-07
   Proceed?
   ```

3. **Execute in order, stopping on failure**:
   ```
   a) clients_create  → capture client_id from response
   b) campaigns_create with client_id  → capture campaign_id
   c) users_invite with client_id, role: "client_user"  → returns invitation_url
   d) campaigns_update with monthly_targets: { "2026-07": 20 }
   ```

4. **If a step fails**: report which one, what error, and ask whether to retry / skip / abort. **Do NOT continue past a failure silently.**

5. **Final summary**: list the new IDs + the invitation URL.

## Validation that will trip you up

- `clients_create` requires all three contact fields. If user only gave email, ask for phone.
- `industry` is required and NOT NULL.
- `users_invite` with `role: "client_user"` REQUIRES `client_id`. Server returns 422 if missing.
- `users_invite` with `role: "sdr"` REQUIRES `location` (UK or SA).
- `monthly_targets` keys must match the YYYY-MM regex.

## Idempotency

For safety on multi-step flows, **pass an `Idempotency-Key` header** on each POST call (use a fresh UUID for the whole onboarding session, suffixed per step: `<uuid>-client`, `<uuid>-campaign`, etc.). This protects against partial-completion if the network blips.

(The MCP tools don't expose idempotency keys directly. If you need exact-once semantics, drop to direct API calls; otherwise the natural workflow is safe as long as you check the response of each step before proceeding.)

## What NOT to do

- Don't do the writes one at a time across multiple turns asking for fields — gather everything first, confirm once, execute in one go.
- Don't skip the confirmation step — you're creating 4 records and one email (the invitation) goes out.
- Don't send the invitation email with `send_email: false` unless the user wants to handle delivery themselves.
- Don't continue if `clients_create` succeeds but `campaigns_create` fails — surface the partial state to the user so they can clean up.

## Example

**User:** "Onboard Acme — SaaS company, contact is Jane Doe (jane@acme.com, +44 207 123 4567). New campaign Q3 Outbound running July through September, target 60. Also invite Jane to the portal."

→ Confirm everything in one block, then run the 4 tool calls in sequence, surfacing any failure immediately.
