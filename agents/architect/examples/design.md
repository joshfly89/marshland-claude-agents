# Architect Agent Examples

The architect agent's job is to push back on design decisions before you build them. It's not looking for bugs in code — it's asking "should this exist at all, and if so, is this the right shape?" These examples show it redirecting decisions that would have caused pain later.

---

## Example 1: Service Split That Creates More Problems Than It Solves

**Input — design proposal:**

> I want to speed up deploy times properly. We originally had every API route as its own Lambda to support mobile clients. We then rebuilt it using an external API dispatcher pattern with an in-Lambda router, which effectively reinvents what API Gateway does natively. It solved the slow deploy problem, reducing it from 1 hour to 20 minutes. Please reevaluate the complexity of this solution while still maintaining the deploy time improvements we achieved.

**Architect agent output:**
  What You Have Now

  You went from ~80 individual Lambdas (~480 CloudFormation resources, ~1 hr
  deploys) to 13 Lambdas (~78 resources, ~20 min deploys). The win is real. The
  cost: you wrote a 132-line in-Lambda router with 91 tests to replicate what
  API Gateway already does. Every new route requires 3 touch points (infra
  config, dispatcher route table, handler import), and all your /sites/* logs
  merge into one stream — painful when debugging at 2am.

  The Real Deploy-Time Driver

  CloudFormation resource count, not Lambda count per se. The relationship is
  sublinear — adding 10 individual Lambdas won't cost you 10x the time. The
  original pain was 80+ Lambdas each creating ~6 CFN resources (function, role,
  log group, API route, permissions, etc.).

  Best Option: Hybrid — Split High-Value Routes Out

  Keep dispatchers for low-traffic CRUD. Move ~5-10 high-value endpoints back to
   individual Lambdas:

  Split out (get isolated logs + per-route metrics):
  - /sites/{siteId}/live-dashboard — highest traffic, hardest to debug combined
  - /sites/{siteId}/energy-charts
  - /sites/{siteId}/automations (list)
  - /users/{userId}/tesla/products (OAuth-critical)

  Keep in dispatchers (low-traffic CRUD that doesn't need observability
  isolation):
  - All onboarding mutations
  - Settings/reserve/energy-mode writes
  - Notification preferences

  Result: ~25 min deploys (acceptable), per-route logs where they matter, no
  behavior change.

  What NOT to Do

  - Consolidate further to 1 Lambda — blast radius and observability get worse,
  not better
  - Lambda Function URLs — you'd own JWT auth in application code; high
  regression risk for marginal gain
  - Code-generate route manifests — premature abstraction at 58 routes

  Immediate Quick Win (0 migration)

  Before splitting anything: add structured logging to the dispatcher — {
  endpoint, method, statusCode, durationMs } on every request. Pipe to New Relic
   custom events. Gets you most of the observability benefit with no infra
  change.

---

**Why this is better than default Claude:** Default Claude would say "both approaches have tradeoffs" and sketch out both architectures. This agent has a stance. It names the specific SST v3 costs (cold starts, IAM surface, log group fragmentation) and gives a concrete threshold for when extraction is actually justified.
