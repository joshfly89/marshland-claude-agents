# Architect Agent Examples

The architect agent's job is to push back on design decisions before you build them. It's not looking for bugs in code — it's asking "should this exist at all, and if so, is this the right shape?" These examples show it redirecting decisions that would have caused pain later.

---

## Example: Lambda Routing Strategy — Hybrid vs. Consolidation

**Input — design proposal:**

> Review the Lambda routing strategy. Two patterns are used side by side: (1) individual `routeBuilder()` Lambdas per endpoint (e.g., `infra/routes/notifications/apple.ts`, `infra/routes/subscriptions/webhooks.ts`), and (2) bundled `externalRoute()` dispatcher catch-alls (e.g., `api/src/external/dispatcher.ts`, `api/src/external/dispatch/sites.ts`).
>
> Evaluate: (1) Is the hybrid principled or accidental? (2) What are the blast-radius, IAM, cold-start, and deployment tradeoffs? (3) Should there be an explicit rule? Return a verdict: keep hybrid with clear rules, consolidate, or something else.

**Architect agent output:**

Verdict: Keep the hybrid — the rule is already mostly implemented, it just isn't written down anywhere. The principle that has evolved:

- `routeBuilder()` → unauthenticated endpoints that receive external push (Stripe/Apple/Google webhooks, device token registration pre-auth). These need explicit API Gateway routes to beat the `{proxy+}` catch-alls.
- `externalRoute()` → all authenticated mobile-app API surface organized by URL namespace. One dispatcher per domain.

The single most important tradeoff: `externalRoute()` creates separate Lambdas per URL prefix in the `routes` array (`infra/utils/externalRoute.ts:45-68`). The notifications dispatcher is actually three Lambdas (`externalNotifications-0`, `-1`, `-2`) covering `/v1/notifications/`, `/v1/user/`, and `/v1/settings/`. This is undocumented and will produce confusing CloudWatch log groups at 2am — a request to `/v1/user/external/{id}` logs in `externalNotifications-1`, not `externalNotifications`. That naming formula needs a warning-level comment on the `routes` property.

The dead stub files (`infra/routes/automations/external.ts`, `infra/routes/demand/external.ts`, `infra/routes/subscriptions/external.ts`) create false surface area in the route map and should either be deleted or given a prominent "intentionally empty" comment — they're easy to mistake for active routes when grep-ing.

---

**Why this is better than default Claude:** Default Claude would list the tradeoffs of each pattern and leave the decision to the developer. This agent determines the hybrid is principled (not accidental), articulates the implicit rule so it can be documented, and identifies the specific undocumented behavior that will cause 2am confusion — the multi-Lambda naming formula hidden inside `externalRoute()`.
