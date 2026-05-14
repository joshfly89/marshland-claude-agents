# Challenge Agent Examples

The challenge agent's job is one thing: find where the code is wrong. Not style, not architecture — correctness. These examples show it catching bugs that look fine on the first read.

---

## Example: Non-Atomic Referral Credit Writes Leave Phantom Credits

**Input — code sent for review:**

> Review the `tryRewardReferrer` logic in `libs/services/src/functions/subscriptions/handlers/processStripeWebhookEventHandler.ts` around lines 1016–1054. This function rewards a referrer with credits when a referred user's first invoice payment succeeds.
>
> Check: (1) Are the credit increment writes (`referralSubscribedCount` + `creditsRemaining`) issued as a single atomic DynamoDB transaction, or as two separate writes? (2) If the idempotency flag write fails and triggers the rollback path, are the rollback writes also atomic, or could a crash between line ~1022 and ~1044 leave the referrer with permanently inflated credits? (3) Does the outer catch block at line ~1054 swallow a failed rollback silently? (4) Is there a scenario where a concurrent execution adds credits, this execution also adds credits, and the rollback only subtracts once (net double-credit)?
>
> Report: identify the specific lines where partial failure produces phantom credits with no recovery path.

**Challenge agent output:**

**Verdict**: Has Issues

**Bugs** (will break at runtime):

**Lines 1016–1022: Non-atomic credit writes expose double-credit window**

The two credit writes are separate DynamoDB operations:

```typescript
await UserSchemaTable.patch({ userId: referrer.userId })
  .add({ referralSubscribedCount: 1 })
  .go();  // Line 1018

await ReferralCreditTable.upsert({ userId: referrer.userId })
  .add({ creditsRemaining: 1 })
  .go();  // Line 1022
```

**Concrete failure scenario:**
1. Execution A increments `referralSubscribedCount` (line 1018 succeeds)
2. Lambda crashes/times out before line 1020 executes
3. SQS redelivers the message (visibility timeout expires)
4. Execution B reads `referralRewardProcessed: undefined` (line 998 early-exit condition fails — flag not set yet)
5. Execution B increments both counters again
6. Execution B's idempotency flag write succeeds (line 1027)
7. **Result**: `referralSubscribedCount = 2`, `creditsRemaining = 1` (or vice versa depending on where the crash fell)

The idempotency guard at line 998 only protects against **completed** executions, not partial writes. Any crash between lines 1016–1029 allows the next SQS retry to re-execute the entire block.

**Fix**: Wrap lines 1016–1022 in an ElectroDB transaction (both entities share `userService` and the same physical table, so they're transaction-eligible):

```typescript
await UserService.transaction
  .write(({ user, credits }) => [
    user.patch({ userId: referrer.userId }).add({ referralSubscribedCount: 1 }).commit(),
    credits.upsert({ userId: referrer.userId }).add({ creditsRemaining: 1 }).commit(),
  ])
  .go();
```

---

**Lines 1039–1044: Rollback writes are non-atomic, crash leaves phantom credits**

The rollback path uses two separate writes:

```typescript
await UserSchemaTable.patch({ userId: referrer.userId })
  .add({ referralSubscribedCount: -1 })
  .go();  // Line 1040
await ReferralCreditTable.patch({ userId: referrer.userId })
  .add({ creditsRemaining: -1 })
  .go();  // Line 1043
```

**Concrete failure scenario:**
1. Concurrent execution B wins the idempotency flag write race (line 1027 succeeds for B, A gets `ConditionalCheckFailed`)
2. Execution A enters rollback at line 1039
3. Line 1040 succeeds (decrements `referralSubscribedCount`)
4. Lambda crashes before line 1042 executes
5. **Result**: `referralSubscribedCount` is net zero, but `creditsRemaining` is permanently +1 — only B's increment applied, A's rollback never finished

The outer catch block at line 1054 swallows the error (logs but no rethrow), so SQS won't redeliver. The phantom credit is permanent.

**Fix**: Wrap rollback writes in the same transactional pattern.

---

**Line 1047: Non-idempotency errors in flag write are silently swallowed**

If the idempotency flag write at line 1027 throws anything other than `ConditionalCheckFailed` (e.g., DynamoDB throttle, network timeout), the error is rethrown at line 1047, then caught by the outer catch at line 1054, which logs but does not rethrow.

**Concrete failure scenario:**
1. Credit writes succeed (lines 1016–1022)
2. Idempotency flag write throws `ProvisionedThroughputExceededException` (line 1027)
3. Inner catch rethrows (line 1047)
4. Outer catch logs and swallows (lines 1054–1060)
5. **Result**: Credits applied, flag not set, SQS redelivers, credits applied again

**Fix**: The outer catch should rethrow to trigger SQS retry.

---

**Edge Cases** (will break under specific conditions):

**Lines 1032–1045: Concurrent rollbacks can double-subtract**

If two executions both reach the rollback path concurrently (both lose the idempotency race to a third execution), each decrements the counters:

1. Executions A and B both increment counters
2. Execution C increments and wins the idempotency write
3. Execution A detects `ConditionalCheckFailed`, rolls back `-1`
4. Execution B detects `ConditionalCheckFailed`, rolls back `-1`
5. **Result**: Net credits = C's +1 + A's -1 + B's -1 = **-1**

**Likelihood**: Low in normal operation, but guaranteed if SQS batch size > 1 and multiple messages reference the same referee. Use a conditional decrement (`where creditsRemaining > 0`) in the rollback path as a safety floor; transactional writes eliminate the race entirely.

---

**Lines 1054–1061: Outer catch swallows all errors, blocks SQS retry**

Any failure in the initial credit writes (1016–1022) — including transient DynamoDB throttling — is swallowed by the top-level catch. SQS considers the message successfully processed and deletes it. The referrer never receives credit with no indication of failure.

**Fix**: Remove the outer catch or rethrow after logging.

**Type Safety:**

None — all DynamoDB interactions are typed via ElectroDB entities, and the error narrowing at line 1032 is adequate.

| Lines | Issue | Failure Mode | Priority |
|-------|-------|--------------|----------|
| 1016–1022 | Non-atomic credit writes | Crash between writes → partial credit on retry | High |
| 1039–1044 | Non-atomic rollback | Crash during rollback → phantom credits | High |
| 1047 + 1054 | Outer catch swallows rethrown errors | Throttle errors block retries | Medium |
| 1032–1045 | Concurrent rollback double-subtract | 3+ simultaneous executions → negative balance | Low |
| 1054–1061 | All errors swallowed | Transient failures become permanent | High |

---

**Why this is better than default Claude:** Default Claude would flag the non-atomic writes and suggest using a transaction. This agent works through all four failure paths independently — partial forward write, partial rollback, error-type leakage through nested catches, and concurrent double-subtract — giving a concrete step-by-step scenario for each and identifying that the outer catch blocking SQS retries is a separate high-priority bug from the atomicity problem.
