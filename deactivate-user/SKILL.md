---
name: deactivate-user
description: Use when an admin wants to deactivate a portal user (revoke their access). Walks through confirmation, captures the reason for the audit log, and uses the safe deactivation flow (preserves audit trail, prevents login).
---

# Deactivate a user safely

## When to use

User wants to revoke another user's portal access:
- "Deactivate Sam from the portal"
- "Remove access for sam@..."
- "Ban this user, they left the company"

## Tool

`users_deactivate` (a convenience tool — internally PATCHes the user with `is_active: false`) from the `outbound-partners` MCP server.

**Requires** `users:write` scope on your API key. **Super-admin only** — regular admins cannot mint such keys.

## What deactivation actually does

This isn't a soft delete or a flag flip. It:

1. Sets `users.is_active = false` in the public users table
2. Sets `banned_until = 100 years from now` on the Supabase auth user
3. **Deletes all active sessions** for that user (they're logged out immediately, every device)
4. Future login attempts get rejected with `"This account is deactivated."`

The user record itself is preserved — meeting history, audit trail, leaderboard aggregates all keep their references. This is intentional.

## Safety checklist

Before invoking, walk through:

1. **Identify the right user**. If the request was by name, resolve to an id:
   ```
   users_list { search: "<name>", is_active: "true" }
   ```
   If multiple matches, show them all (id, email, role, last_sign_in_at) and ask which.

2. **Check what depends on this user**. Useful context to surface:
   - Their role (deactivating a super_admin is high-impact)
   - Whether they own any active campaigns (`campaigns_list` with `campaign_manager_id`)
   - Recent meeting activity (`meetings_list` with `sdr_id`, `date_from: now-7d`)

3. **Capture the reason** from the user — for the audit log:
   ```
   Why are you deactivating Sam SDR (sam@…)?
     a) Left the company
     b) Role change — no longer needs access
     c) Account compromised
     d) Other (specify)
   ```

4. **Confirm explicitly** before invoking:
   ```
   ⚠️  Confirm: deactivate Sam SDR (sam@…, sdr role)?
   They will be logged out immediately and unable to log in again.
   Reason logged: "Left the company"
   ```

5. **Invoke**:
   ```
   users_deactivate { id: "<uuid>" }
   ```

   This carries `annotations.destructiveHint: true` so most MCP clients will prompt the user one more time.

6. **Report the result**:
   - Confirm the deactivation succeeded
   - Note that the audit_log row now records who did it and when
   - If sessions were deleted, mention that the user has been signed out from all devices

## Reactivation

If the user needs access back, use `users_reactivate { id }`. This:
- Sets `is_active = true`
- Sets `banned_until = none` in Supabase auth
- Does NOT restore previous sessions — they need to log in fresh

## What NOT to do

- Don't deactivate by name without resolving to an id first — name collisions exist.
- Don't deactivate without capturing a reason. The audit log will show who did it, but not why, and "why" is what matters in 3 months.
- Don't tell the user the deactivation is reversible casually — sessions are gone, devices need re-login, and if the user was a super_admin you may have just locked yourself out.
- Don't suggest deletion as an alternative — hard-deletes aren't exposed in v1 (and shouldn't be without explicit super-admin review). Deactivation is the right tool.

## Example

**User:** "Sam left last week. Deactivate his account."

1. `users_list { search: "Sam", role: "sdr", is_active: "true" }` → pick the right Sam
2. Show what depends on him (recent meetings, campaign ownership)
3. Ask reason → "Left the company"
4. Confirm → user OKs
5. `users_deactivate { id: "..." }`
6. "Done. Sam (sam@…) is deactivated as of <timestamp>. Reason recorded: Left the company. He has been signed out from all sessions."
