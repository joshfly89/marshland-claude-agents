# Challenge Agent Examples

The challenge agent's job is one thing: find where the code is wrong. Not style, not architecture — correctness. These examples show it catching bugs that look fine on the first read.

---

## Example: Apple Subscription Grace Window Applied Asymmetrically

**Input — code sent for review:**

> Review `libs/resources/src/electro/PaymentTable/services/appStoreActiveCheck.ts`. This file was recently refactored (commit 43461a33) to add a 48-hour grace window on top of `currentPeriodEnd` for `active`, `in_grace`, and `past_due` statuses. The old code for the cron path treated `active`/`in_grace`/`past_due` as always-active (no `currentPeriodEnd` check at all).
>
> Check: (1) Is the grace window applied consistently across all active-ish statuses, or are some statuses treated asymmetrically? (2) Can a user who explicitly cancels (`canceled` status) end up with LESS remaining access than a user whose subscription lapses without a cancel event (`active` with expired `currentPeriodEnd`)? (3) Is there a path where `currentPeriodEnd` is null/undefined that bypasses the grace window comparison silently?
>
> Report any correctness bugs found — specifically cases where two users with equivalent billing states receive different access durations.

**Challenge agent output:**

**Verdict**: Has Issues

**Bugs** (will break at runtime):

None — the logic correctly implements the intended grace period policy.

**Edge Cases** (will break under specific conditions):

1. **`canceled` users lose 48 hours of paid access vs. lapsed-active users** (lines 17-21 vs. 13-15)
   - **Condition**: User A has `status: 'active'`, `currentPeriodEnd: expiredTimestamp` (expired 24h ago, still within 48h grace). User B explicitly canceled (`status: 'canceled'`), same `currentPeriodEnd`.
   - **Result**: User A gets access (`expiredTimestamp + 48h > now`). User B loses access immediately (`expiredTimestamp > now` fails).
   - **Likelihood**: High for any user who cancels instead of letting the subscription lapse.
   - **Fix**: Apply the grace window to `canceled` status as well:
     ```ts
     if (status === 'canceled') {
       return (
         typeof sub.currentPeriodEnd === 'number' &&
         sub.currentPeriodEnd + APP_STORE_STALE_PERIOD_GRACE_MS > now
       );
     }
     ```
   - **Note**: The current code punishes users who explicitly cancel through Apple's UI — they lose 48h of paid time — vs. users whose subscriptions simply expire without a `canceled` event, who keep the grace period. This is the opposite of fair.

2. **Missing `currentPeriodEnd` bypasses grace window for active-ish statuses** (line 14)
   - **Condition**: Record has `status: 'active'` but `currentPeriodEnd` is `null`/`undefined`/non-numeric.
   - **Result**: Returns `true` (grants access forever with no expiration check).
   - **Likelihood**: Medium — depends on whether Apple's server-to-server notifications always include `currentPeriodEnd`. If a malformed/partial notification writes a record without this field, the subscription becomes immortal.
   - **Impact**: A user could retain premium access indefinitely if their record gets into this state via a notification parsing bug or manual DB edit.
   - **Fix**: Treat missing `currentPeriodEnd` as expired for `active`/`in_grace`/`past_due`:
     ```ts
     if (['active', 'in_grace', 'past_due'].includes(status)) {
       if (typeof sub.currentPeriodEnd !== 'number') return false; // was: true
       return sub.currentPeriodEnd + APP_STORE_STALE_PERIOD_GRACE_MS > now;
     }
     ```
   - **Counterargument**: If the intention is "Apple says active, so trust it even without an end date," this is by design — but it's a type safety hole that could be exploited.

**Type Safety** (compiler won't catch but will bite you):

None — the optional `currentPeriodEnd` check uses runtime `typeof` guards, and the status check is a loose string comparison (appropriate given the untyped input from Apple).

The asymmetry in edge case #1 is the critical correctness bug: a user who cancels loses 48 hours of paid access compared to a user who simply lets their subscription expire. If the grace window exists to accommodate Apple's delayed renewal notifications, it should apply equally to all billing end states, not just non-canceled ones.

---

**Why this is better than default Claude:** Default Claude would read the grace window logic and confirm it looks correct. This agent re-derives the invariant from first principles — "two users with equivalent billing states should receive equal access duration" — and finds the asymmetry between `canceled` and lapsed-`active` that violates it. It also identifies the immortal-subscription hole that wasn't part of the original question.
