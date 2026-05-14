# Security Agent Examples

The security agent reviews code the way an attacker reads it — looking for the path through, not the happy path. These examples show it catching vulnerabilities that pass code review and TypeScript and only surface in production.

---

## Example: JWT Fallback Decoder Accepts Forged Tokens, BOLA in Every Resolver

**Input — code sent for review:**

> Review the JWT handling in `libs/trpc/src/lib/trpc.ts` (lines 21-29 and 60-69), specifically the `decodeJwtPayload()` function and its use in `createTRPCContext`.
>
> Check: (1) Does `decodeJwtPayload()` verify the JWT signature, or does it only base64-decode the payload without cryptographic validation? (2) When is this fallback function used — only as a last resort if API Gateway authorizer fails, or does it run on every request? (3) Can an attacker craft a JWT with an arbitrary `sub` claim and pass it to this function? (4) Is there any upstream validation (API Gateway, middleware) that prevents unsigned JWTs from reaching the tRPC context? (5) What would happen if the API Gateway authorizer is misconfigured or temporarily disabled?
>
> Identify scenarios where an attacker could forge a JWT to impersonate any user, including whether the fallback creates a privilege escalation if the authorizer is bypassed.

**Security agent output:**

**Risk Level**: Critical

**Vulnerability 1 — JWT fallback decoder accepts any base64 string as authentication** (lines 21-29, 62-69)

The `decodeJwtPayload()` function base64-decodes the JWT payload without verifying the cryptographic signature. It is invoked any time API Gateway authorizer claims are absent:

```typescript
function decodeJwtPayload(token: string) {
  const parts = token.split('.');
  const payload = Buffer.from(parts[1], 'base64').toString('utf-8');
  return JSON.parse(payload);  // NO SIGNATURE VERIFICATION
}

if (!userId) {
  const authHeader = event.headers?.authorization ?? event.headers?.Authorization;
  if (authHeader) {
    const token = authHeader.startsWith('Bearer ') ? authHeader.slice(7) : authHeader;
    const payload = decodeJwtPayload(token);  // accepts any base64 string
    userId = typeof payload?.sub === 'string' ? payload.sub : undefined;
  }
}
```

**Attack vector:**
1. Attacker crafts a JWT with an arbitrary `sub` targeting a victim's Cognito ID:
   ```
   anystring.eyJzdWIiOiJ2aWN0aW0tdXNlci1pZCIsImV4cCI6OTk5OTk5OTk5OX0=.anysignature
   ```
2. Attacker sends a request to any tRPC endpoint with `Authorization: Bearer <fake-jwt>`
3. On routes with `jwtAuth: false` or when the API Gateway authorizer is misconfigured/disabled, the fallback populates `ctx.userId` with the forged `sub` value
4. The resolver processes the request as if the victim is authenticated

**Impact**: Complete account takeover. Attacker can read victim's sites, automations, billing, settings, Tesla credentials; modify or delete automations; trigger subscription manipulation; delete the victim's account via `deleteUserAccount`.

**Fix**: Remove the fallback decoder entirely. If the API Gateway authorizer fails to populate claims, the request must be rejected:

```typescript
// DELETE lines 62-69 (the entire fallback block)
// Instead, enforce at the procedure level:
const requireAuth = t.middleware(async ({ ctx, next }) => {
  if (!ctx.userId) throw new TRPCError({ code: 'UNAUTHORIZED' });
  return next({ ctx: { ...ctx, userId: ctx.userId } });
});

export const authedProcedure = t.procedure.use(maintenanceGuard).use(requireAuth);
```

Replace all `publicProcedure` with `authedProcedure` in user-facing routers (users, automations, sites, tesla, subscriptions, notifications). Only health checks and truly public endpoints should use `publicProcedure`.

Verification test: `Authorization: Bearer fake.eyJzdWIiOiJ0ZXN0In0=.fake` sent to a protected endpoint must return `401`, not a successful response.

---

**Vulnerability 2 — Broken object-level authorization in every resolver** (all resolver files)

Resolvers accept `userId` as a client-controlled input field and pass it directly to the service layer without checking that `input.userId === ctx.userId`:

```typescript
// getUserSettingsResolver
return await getUserSettings(input);  // input.userId is attacker-controlled

// getSiteResolver
const site = await getSite(input);  // input.userId from request body, not ctx.userId

// deleteUserAccountResolver
return await service.deleteUserAccount(input);  // passes input directly
```

**Attack vector:**
1. Attacker authenticates with their own valid JWT (`ctx.userId = "attacker-id"`)
2. Attacker crafts a tRPC request with `{ userId: "victim-user-id" }` in the input body
3. Resolver passes `input.userId` to the service layer and returns victim's data to the attacker

**Impact**: Horizontal privilege escalation. Attacker can enumerate users, read other users' automations/sites/settings/entitlements, modify other users' data via mutation resolvers, and harvest PII and Tesla credentials. This attack works even when JWT validation is functioning correctly — the problem is trusting the client's userId over the server-verified claim.

**Fix**: Never trust client-provided `userId`. Extract it from `ctx`:

```typescript
// BEFORE:
export const getUserSettingsResolver = async ({ input }) => getUserSettings(input);

// AFTER:
export const getUserSettingsResolver = async ({ input, ctx }) => {
  if (input.userId !== ctx.userId) throw new TRPCError({ code: 'FORBIDDEN' });
  return getUserSettings({ userId: ctx.userId });
};
```

Remove `userId` from input schemas for user-scoped operations. Create a shared ownership helper:

```typescript
function requireOwnership(ctx: TRPCContext, resourceUserId: string) {
  if (!ctx.userId) throw new TRPCError({ code: 'UNAUTHORIZED' });
  if (ctx.userId !== resourceUserId && !ctx.groups.includes('admin')) {
    throw new TRPCError({ code: 'FORBIDDEN' });
  }
}
```

---

**Vulnerability 3 — `jwtAuth` defaults to `false`, unsafe for new routes** (`infra/utils/trpcRoute.ts:27`)

The `jwtAuth` flag is optional and defaults to `false`. Any developer who creates a new tRPC router and forgets to set `jwtAuth: true` ships an unauthenticated API namespace.

**Attack vector:** Developer creates a new `payments` router, forgets `jwtAuth: true`. All procedures in that router are accessible without authentication. Combined with Vulnerability 1, attacker can forge a JWT with any `sub` and call `payments.getInvoices({ userId: 'victim' })`.

**Impact**: Entire API namespace becomes unauthenticated and susceptible to the forged-JWT attack, leading to potential mass data breach.

**Fix**: Default to `jwtAuth: true` and require explicit opt-out:

```typescript
export const trpcRoute = <TService extends RouterNamespaces>({
  jwtAuth = true,  // was: optional, defaulted to false
  ...
}) => { /* ... */ };
```

Audit all existing routes: `grep -r "jwtAuth:" infra/routes` — verify only `weather` and `referrals` have `jwtAuth: false`. Add an infrastructure test to prevent regression.

| Lines | Issue | Attack Vector | Priority |
|-------|-------|---------------|----------|
| 21-29, 62-69 | JWT fallback accepts forged tokens | Forge any sub claim → complete account takeover | Critical |
| All resolvers | BOLA — client-controlled userId | Authenticated attacker reads/modifies any user's data | Critical |
| trpcRoute.ts:27 | `jwtAuth` defaults to false | Missing flag ships unauthenticated namespace | High |

---

**Why this is better than default Claude:** Default Claude would catch the `decode` vs `verify` distinction and suggest using `aws-jwt-verify`. This agent identifies that the fallback is triggered by any missing authorizer claims (not just outages), shows the exact forged token payload, independently finds the BOLA vulnerability affecting every resolver as a second critical-severity issue that wasn't in the original question, and fixes the unsafe default in `trpcRoute` to close the third failure mode.
