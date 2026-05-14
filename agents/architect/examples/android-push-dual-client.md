# Architect Agent Examples

The architect agent's job is to push back on design decisions before you build them. It's not looking for bugs in code — it's asking "should this exist at all, and if so, is this the right shape?" These examples show it redirecting decisions that would have caused pain later.

---

## Example: Android Push Notification Dual-Client Pattern

**Input — design proposal:**

> Review the Android push notification implementation. There are two fully-implemented clients: `libs/services/src/functions/notifications/android/androidPushClient.ts` (direct Firebase Admin SDK) and `libs/services/src/functions/notifications/android/androidPushClientSNS.ts` (FCM via AWS SNS). Selection between them happens at runtime in `libs/services/src/functions/notifications/buildSendingClients.ts` via env-var. Also review `libs/services/src/functions/notifications/notificationClient.ts`.
>
> Evaluate: (1) Is the dual-client pattern a migration in progress or permanent dual support — and which design is cleaner long-term? (2) The SNS client calls GetEndpointAttributes before every send to check if the endpoint is enabled — is this the right pattern vs. reactive pruning on failure? (3) Does the runtime env-var selection introduce risk? Return a verdict with the single most important architectural concern.

**Architect agent output:**

Verdict: The `GetEndpointAttributes` pre-check at `androidPushClientSNS.ts:348-363` is the one change that matters. It doubles per-notification AWS API calls (2x SNS calls per device) with no correctness benefit — `PublishCommand` already throws `EndpointDisabledException` synchronously on a disabled endpoint, so the pre-check can't prevent a failure, it just makes it less visible. There's also a race condition: SNS can disable the endpoint between the attribute check and the publish anyway. Switch to catching `EndpointDisabledException` on Publish directly (reactive pruning). That's ~20 lines in `androidPushClientSNS.ts`.

The dual-client pattern is a completed migration with dead code that was never deleted. `androidPushClient.ts` and `fcmProvider.ts` are unreachable in prod (since `FCM_TOPIC_ARN` is always set). Delete them after confirming the ARN is present in all three stages.

Secondary finding: Tokens registered before the SNS migration exist in `AndroidPushTokenTable` without an `endpointArn`. The SNS client silently filters these out (`filter(Boolean)`) — those users receive no Android push notifications and no error is logged at the send callsite.

---

**Why this is better than default Claude:** Default Claude would outline the tradeoffs of each client pattern and suggest testing both. This agent identifies the single highest-priority change (the pre-check doubling API calls with a race condition), calls the dead code what it is, and surfaces a silent data loss bug — users silently dropped from push — that wasn't part of the original question.
