# Challenge Agent Examples

The challenge agent's job is one thing: find where the code is wrong. Not style, not architecture — correctness. These examples show it catching bugs that look fine on the first read.

---

## Example 1: TOCTOU Race in a Stripe + DynamoDB Cancellation Flow

**Input — code sent for review:**

```typescript
// packages/functions/src/subscriptions/cancel.ts
export const cancelSubscription = protectedProcedure
  .input(z.object({ subscriptionId: z.string() }))
  .mutation(async ({ ctx, input }) => {
    const sub = await db.subscriptions
      .get({ userId: ctx.userId, subscriptionId: input.subscriptionId })
      .go();

    if (sub.data?.status !== 'active') {
      throw new TRPCError({ code: 'BAD_REQUEST', message: 'Not active' });
    }

    await stripe.subscriptions.cancel(input.subscriptionId);

    await db.subscriptions
      .patch({ userId: ctx.userId, subscriptionId: input.subscriptionId })
      .set({ status: 'cancelled' })
      .go();
  });
```

**Challenge agent output:**

Two bugs, one severe.

**Bug 1 — TOCTOU race:** You read `status === 'active'`, then cancel in Stripe, then write `cancelled` to DynamoDB. If two requests arrive at the same time — say, the user double-clicks — both will pass the status check before either writes back. Both will call `stripe.subscriptions.cancel()`. The second Stripe call will throw a `resource_missing` error, your Lambda will return a 500, and the user sees an error on a cancellation that actually succeeded.

Fix: Use a DynamoDB conditional write to own the status transition atomically before touching Stripe:

```typescript
// Attempt to own the transition first
await db.subscriptions
  .patch({ userId: ctx.userId, subscriptionId: input.subscriptionId })
  .set({ status: 'cancelling' })
  .where(({ status }, { eq }) => eq(status, 'active'))
  .go(); // throws ConditionalCheckFailedException if already cancelling/cancelled

await stripe.subscriptions.cancel(input.subscriptionId);

await db.subscriptions
  .patch({ userId: ctx.userId, subscriptionId: input.subscriptionId })
  .set({ status: 'cancelled' })
  .go();
```

**Bug 2 — inconsistent state on Lambda crash:** If the Lambda times out or throws after the Stripe cancel but before the DB write, Stripe has the subscription as cancelled but your DB still shows `active`. The user lost access but your app thinks they're paid. Add a Stripe webhook handler for `customer.subscription.deleted` as the source of truth write — don't rely solely on the inline flow.

---

**Why this is better than default Claude:** Default Claude would note the race condition exists and suggest a mutex or queue. It wouldn't connect it to ElectroDB's conditional write API, wouldn't mention the Stripe error surface, and wouldn't flag the inconsistent-state failure mode as a separate bug. This agent knows the stack and sees both problems.

---

## Example 2: Silent Undefined on an ElectroDB Query Result

**Input — code sent for review:**

```typescript
// packages/functions/src/users/getProfile.ts
export const getProfile = protectedProcedure.query(async ({ ctx }) => {
  const result = await db.users.query
    .byEmail({ email: ctx.userEmail })
    .go();

  const user = result.data[0];

  return {
    id: user.userId,
    name: user.name,
    plan: user.plan ?? 'free',
    createdAt: user.createdAt,
  };
});
```

**Challenge agent output:**

`result.data[0]` is typed as `User` by ElectroDB but is `undefined` at runtime when no record matches. TypeScript won't catch this — `data` is typed `User[]`, and array index access on an empty array silently produces `undefined` unless you have `noUncheckedIndexedAccess` in your tsconfig (you almost certainly don't).

When a new user signs in before their record is fully provisioned, this throws `Cannot read properties of undefined (reading 'userId')` in a context where `ctx.userEmail` is valid. The error message gives no indication the user record is missing.

```typescript
const result = await db.users.query.byEmail({ email: ctx.userEmail }).go();
const user = result.data[0];

if (!user) {
  throw new TRPCError({
    code: 'NOT_FOUND',
    message: 'User record not found — may still be provisioning',
  });
}

return {
  id: user.userId,
  name: user.name,
  plan: user.plan ?? 'free',
  createdAt: user.createdAt,
};
```

Also: if this query can return multiple rows (e.g., duplicate emails from a data migration), you're silently taking the first one without logging the anomaly. Worth asserting `result.data.length === 1` if email should be unique.

---

**Why this is better than default Claude:** Default Claude would mention "you should check if the array is empty." This agent explains *why* TypeScript misses it (ElectroDB's return type), names the specific real-world scenario that triggers it (provisioning lag), and flags the second latent bug about duplicate rows. It also knows not to add a try/catch wrapper around something that isn't an exception.

---

## Example 3: Missing Idempotency in a Stripe Webhook Handler

**Input — code sent for review:**

```typescript
// packages/functions/src/webhooks/stripe.ts
export const handler = async (event: APIGatewayProxyEvent) => {
  const sig = event.headers['stripe-signature']!;
  const stripeEvent = stripe.webhooks.constructEvent(
    event.body!,
    sig,
    process.env.STRIPE_WEBHOOK_SECRET!
  );

  if (stripeEvent.type === 'checkout.session.completed') {
    const session = stripeEvent.data.object as Stripe.Checkout.Session;

    await db.orders.create({
      orderId: ulid(),
      userId: session.metadata!.userId,
      amount: session.amount_total!,
      status: 'paid',
      stripeSessionId: session.id,
    }).go();

    await ses.sendEmail({
      to: session.customer_email!,
      template: 'order-confirmation',
      data: { orderId: /* can't access — not returned */ '?' },
    });
  }

  return { statusCode: 200 };
};
```

**Challenge agent output:**

Three correctness issues.

**Bug 1 — no idempotency guard:** Stripe delivers webhooks at-least-once. Network hiccups, Lambda timeouts, and Stripe's own retry logic mean this handler will run multiple times for the same `checkout.session.completed` event. Each run creates a new order with a new `ulid()`. A user can end up with duplicate orders and duplicate confirmation emails.

Fix: check and record the Stripe event ID before processing:

```typescript
const existing = await db.stripeEvents
  .get({ eventId: stripeEvent.id })
  .go();

if (existing.data) {
  return { statusCode: 200 }; // already processed
}

// Write the event record first (conditional on not existing)
await db.stripeEvents.create({ eventId: stripeEvent.id, processedAt: new Date().toISOString() }).go();
```

**Bug 2 — `orderId` not available for the email:** You call `ulid()` inside the create call but can't reference it afterward to pass to the email. Either generate it before the create or retrieve it from the DynamoDB response.

**Bug 3 — non-null assertions on user-controlled fields:** `session.metadata!.userId` and `session.customer_email!` will throw if those fields are absent. Validate them before proceeding — a malformed session shouldn't crash the webhook handler and cause Stripe to consider delivery failed.

---

**Why this is better than default Claude:** Default Claude catches the idempotency issue. This agent catches all three bugs, explains the specific Stripe delivery behavior that causes it, knows `ulid()` is being shadowed by scope, and flags the non-null assertion pattern as a correctness risk rather than a style preference.
