# Security Agent Examples

The security agent reviews code the way an attacker reads it — looking for the path through, not the happy path. These examples show it catching vulnerabilities that pass code review and TypeScript and only surface in production.

---

## Example: Tesla OAuth Flow Missing State Validation and Redirect Allowlist

**Input — code sent for review:**

> Review the OAuth flow in `libs/services/src/functions/tesla/getAuthUrl.ts` (lines 7-32) and the callback handler in `libs/services/src/functions/tesla/exchangeCodeForTokenResolver.ts`.
>
> Check: (1) Is the OAuth state parameter cryptographically random per-session, or does it use a default/hardcoded value? (2) On the callback side, is the returned state parameter validated to match the session's original state? (3) Are there CSRF protections (nonce, SameSite cookies, origin checks)? (4) Is the `redirectUri` validated against an allowlist, or can an attacker supply arbitrary callback URLs? (5) What happens if state validation fails — does the handler reject the request or silently proceed?
>
> Identify any auth bypass or CSRF attack vectors, including concrete scenarios where an attacker could hijack the OAuth token flow or trick a user into authorizing to a malicious callback.

**Security agent output:**

**Risk Level**: Critical

**Vulnerability 1 — Hardcoded OAuth state parameter enables CSRF** (lines 10, 22)

The state parameter uses a static default value instead of a cryptographically random per-session nonce.

**Attack vector:**
1. Attacker initiates their own Tesla OAuth flow from a legitimate Grid Getter account using the predictable state value `'tesla-oauth'`
2. Attacker tricks victim into clicking the Tesla authorization URL via phishing ("Your Grid Getter connection expired, click here to reconnect")
3. Victim authorizes Tesla access; Tesla redirects to callback with `code=VICTIM_CODE&state=tesla-oauth`
4. Attacker calls `exchangeCodeForToken` with `{ userId: ATTACKER_USER_ID, code: VICTIM_CODE }` — the `state` parameter is never passed to `exchangeCodeForToken`, never stored, never compared
5. Victim's Tesla token is now stored under the attacker's userId in `UserAuthTokenTable`

**Impact**: Full account takeover of victim's Tesla Powerwall. Attacker can read live battery data, control charge/discharge modes, modify energy plans, and enable/disable automations. Victim sees "Tesla Connected" but their tokens are in the attacker's account.

**Fix**: Generate a cryptographically random state per session, store it in DynamoDB with a 10-minute TTL, and validate it on callback before any token exchange:

```typescript
// getAuthUrl.ts
const state = randomBytes(32).toString('base64url');
await OAuthStateTable.create({ state, userId: input.userId, ttl: Math.floor(Date.now() / 1000) + 600 }).go();

// exchangeCodeForToken.ts
const stateRecord = await OAuthStateTable.get({ state }).go();
if (!stateRecord.data || stateRecord.data.userId !== userId) {
  throw new TRPCError({ code: 'FORBIDDEN', message: 'Invalid or expired OAuth state' });
}
await OAuthStateTable.delete({ state }).go(); // prevent replay
```

---

**Vulnerability 2 — Missing redirect URI allowlist enables token theft** (lines 7-32)

`redirectUri` is validated only as a URL format (`z.string().url()`), not against an allowlist of allowed Grid Getter domains.

**Attack vector:**
1. Attacker calls `getAuthUrl` with `redirectUri: 'https://attacker.com/steal'`
2. Zod validates it's a valid URL but not that it's an allowed domain
3. Tesla redirects to `https://attacker.com/steal?code=VICTIM_CODE`
4. Attacker exchanges the stolen code under their own userId

**Impact**: Same as Vulnerability 1 — full Powerwall control — but easier to exploit since the attacker controls the redirect destination directly.

**Fix**: Enforce an allowlist before generating the auth URL:

```typescript
const ALLOWED_REDIRECT_URIS = [
  'https://gridgetter.com/tesla/callback',
  'https://staging.gridgetter.com/tesla/callback',
  process.env.IS_LOCAL ? 'http://localhost:4200/tesla/callback' : null,
].filter(Boolean);

if (!ALLOWED_REDIRECT_URIS.includes(input.redirectUri)) {
  throw new TRPCError({ code: 'BAD_REQUEST', message: 'Invalid redirect URI' });
}
```

Apply the same check in `exchangeCodeForToken` before calling Tesla's token endpoint.

---

**Vulnerability 3 — OAuth endpoints use `publicProcedure`** (lines 45-50, `api/src/trpc/tesla/index.ts`)

Both `getAuthUrl` and `exchangeCodeForToken` require no authentication. The `userId` is taken from the request body without verifying it matches the authenticated session.

**Attack vector:**
```bash
curl -X POST https://api.gridgetter.com/v1/oauth/tesla/authorize \
  -d '{"userId":"VICTIM_USER_ID","code":"STOLEN_AUTH_CODE","redirectUri":"https://gridgetter.com/tesla/callback"}'
# No authentication required. Victim's Tesla tokens overwritten.
```

**Impact**: Account takeover without needing to authenticate as the victim. Bypasses any frontend session/cookie protections.

**Fix**: Require authentication and validate ownership in the resolver:

```typescript
exchangeCodeForToken: protectedProcedure
  .mutation(async ({ input, ctx }) => {
    if (input.userId !== ctx.userId) {
      throw new TRPCError({ code: 'FORBIDDEN', message: 'Cannot connect OAuth for another user' });
    }
    return exchangeCodeForToken(input);
  }),
```

---

**Vulnerability 4 — No authorization code replay protection** (lines 10, 98-110)

There is no server-side tracking of used authorization codes. An attacker with a valid code can attempt to exchange it multiple times within the 10-minute expiration window.

**Attack vector:** Attacker intercepts the callback URL (network MitM, server logs) and calls `exchangeCodeForToken` with the same `code` before Tesla invalidates it.

**Impact**: Token theft via replay. Less reliable since Tesla usually invalidates codes after first use, but exploitable in race conditions.

**Fix**: Track used authorization codes in DynamoDB with a TTL matching Tesla's expiration window.

| Lines | Issue | Attack Vector | Priority |
|-------|-------|---------------|----------|
| 10, 22 | Hardcoded state parameter | CSRF hijacks victim's Tesla token | Critical |
| 7-32 | Missing redirectUri allowlist | Token theft via attacker-controlled callback | Critical |
| 45-50 | No auth on OAuth endpoints | Unauthenticated token injection to arbitrary users | Critical |
| 10, 98-110 | No code replay protection | Attacker exchanges code twice within expiration | High |

---

**Why this is better than default Claude:** Default Claude would flag the missing state validation and suggest adding a nonce. This agent works through all four attack paths independently — CSRF via predictable state, token theft via open redirect, unauthenticated endpoint allowing arbitrary userId injection, and code replay — and gives a concrete `curl` exploit for the third vulnerability that makes the severity undeniable.
