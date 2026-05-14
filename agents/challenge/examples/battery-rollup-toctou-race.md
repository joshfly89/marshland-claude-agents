# Challenge Agent Examples

The challenge agent's job is one thing: find where the code is wrong. Not style, not architecture — correctness. These examples show it catching bugs that look fine on the first read.

---

## Example: DynamoDB Stream Race Corrupts Battery Rollup Stats

**Input — code sent for review:**

> Review the `mergeRollup` function in `libs/services/src/functions/automations/streams/powerHistoryRollups.rollups.ts` (approximately lines 36–77). This function is called from a DynamoDB stream handler and may execute concurrently across multiple stream shards.
>
> Check: (1) Is there a read-then-write gap where `batteryStats` is computed from a snapshot read outside the transaction, then committed via `.set()` inside the transaction? (2) Are the `.add()` and `.set()` operations in the transaction subject to the same atomicity guarantees, or is `.set()` vulnerable to last-write-wins overwrite? (3) Under concurrent execution, can two workers both read the same `batteryPercentStart`/`batteryPercentEnd` snapshot and produce a corrupted final rollup where the later writer silently overwrites the earlier one's correctly-merged battery values?
>
> Report: is this a real race condition, or is there a condition expression or lock that prevents it?

**Challenge agent output:**

**Verdict**: Has Issues

**Bugs** (will break at runtime):

**Lines 36-41: Read-then-write race condition in concurrent stream processing**

When two DynamoDB stream workers process different telemetry records for the same rollup bucket (same `siteId`, `userId`, and `bucketHour`/`bucketDay`/`bucketMonth`) within ~milliseconds of each other:

1. **Worker A** reads `currentRollup` at line 36 (e.g., `batteryPercentStart: 60`, `batteryPercentStartAt: 1000`)
2. **Worker B** reads the same `currentRollup` at line 36 (same snapshot: `batteryPercentStart: 60`, `batteryPercentStartAt: 1000`)
3. **Worker A** calls `buildBatteryPercentStats()` with a reading at `createdDate: 900` → computes `batteryPercentStart: 55`, `batteryPercentStartAt: 900` (earlier timestamp wins)
4. **Worker B** calls `buildBatteryPercentStats()` with a reading at `createdDate: 1100` → computes `batteryPercentStart: 60`, `batteryPercentStartAt: 1000` (existing timestamp wins since `1100 > 1000`)
5. **Worker A** commits its transaction with `.set({ batteryPercentStart: 55, batteryPercentStartAt: 900, ... })`
6. **Worker B** commits its transaction with `.set({ batteryPercentStart: 60, batteryPercentStartAt: 1000, ... })`
7. **Worker B's write overwrites Worker A's values** — the rollup now shows `batteryPercentStart: 60, batteryPercentStartAt: 1000`, losing the earlier battery reading (55 at 900).

The `.add()` calls at lines 62-72 are **atomic and safe** — DynamoDB's `ADD` operation is commutative under concurrent writes. But `.set()` at line 73 performs **last-write-wins overwrite** with no condition expression.

**Why this happens:**
- ElectroDB `.update().set()` translates to DynamoDB `UpdateItem` with a `SET` clause — it overwrites the attribute unconditionally
- The transaction provides **no isolation** between the read at line 36 and the write at line 73 — no optimistic locking, no condition expression checking `updatedAt` or a version number
- DynamoDB stream shards invoke Lambda concurrently, and telemetry records for the same rollup key can arrive in different batches milliseconds apart (especially during DemandGuard when `telemetryIntervalMinutes: 1`)

**Concrete scenario where it breaks:**
- Site has DemandGuard active → 1-minute telemetry
- Two records arrive in separate stream batches within <500ms:
  - Record 1: `createdDate: 2026-04-29T14:00:00.000Z` (earlier), `batteryPercent: 75`
  - Record 2: `createdDate: 2026-04-29T14:01:00.000Z` (later), `batteryPercent: 72`
- Both target the same `bucketHour: "2026-04-29T14"`
- If Record 2's transaction commits after Record 1's, the rollup shows `batteryPercentStart: 72` instead of the correct `75`
- `batteryPercentMin`/`batteryPercentMax` will also be corrupted if Record 2's snapshot was stale

**Fix:** Use a conditional write to prevent overwrites when the stored timestamp is better than the computed one:

```ts
const updateStep = target
  .update(key)
  .add({ /* ... */ })
  .set(setAttributes);

if (batteryStats.batteryPercentStartAt < (currentRollup?.batteryPercentStartAt ?? Infinity)) {
  updateStep.where(({ batteryPercentStartAt }, { eq, attribute_not_exists }) =>
    attribute_not_exists(batteryPercentStartAt) ||
    eq(batteryPercentStartAt, currentRollup?.batteryPercentStartAt)
  );
}

return [updateStep.commit()];
```

**Edge Cases** (will break under specific conditions):

**Lines 92-141: `buildBatteryPercentStats` logic assumes single-writer semantics**

The timestamp comparison logic at lines 99-108 (start) and 110-120 (end) is correct only if the read snapshot reflects all prior writes. Under concurrent execution, two workers may both see `batteryPercentStartAt: undefined` and both write their own `batteryPercentStart`, causing one to be lost. Similarly for `batteryPercentMin`/`batteryPercentMax` at lines 122-131 — if Worker A computes `min(70, 65) = 65` and Worker B computes `min(undefined, 68) = 68`, and B's write lands last, the rollup shows 68 instead of the true minimum 65.

**Likelihood:** Medium-high during DemandGuard (1-minute telemetry creates 5x more rollup updates per hour than off-peak 5-minute telemetry). Low otherwise.

**Type Safety:**

None — the types accurately reflect the lack of transactional safety.

---

**Why this is better than default Claude:** Default Claude would note a potential race condition and suggest "using transactions or locks." This agent precisely distinguishes between the `.add()` calls (commutative, safe) and the `.set()` call (last-write-wins, unsafe), traces the exact sequence of worker interleaving that produces silent data corruption, ties the likelihood to a specific feature flag (`DemandGuard`/1-minute telemetry), and provides a working condition expression fix rather than a generic recommendation.
