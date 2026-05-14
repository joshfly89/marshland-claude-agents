# Security Agent Examples

The security agent reviews code the way an attacker reads it — looking for the path through, not the happy path. These examples show it catching vulnerabilities that pass code review and TypeScript and only surface in production.

---

## Example: App Store Webhook Accepts Any Signature When X509Certificate Unavailable

**Input — code sent for review:**

> Review the App Store webhook handler in `api/src/external/subscriptions/appStoreHandlers.ts` (lines 85-100) and trace how `signedPayload` and `signedRenewalInfo` are processed.
>
> Check: (1) Is the incoming webhook's cryptographic signature verified before processing the payload? (2) What signature scheme does Apple use (JWS, HMAC, certificate pinning)? (3) Are the signed fields (`signedPayload`, `signedRenewalInfo`) validated before being passed to business logic? (4) Can an attacker send a webhook with an arbitrary payload that bypasses signature checks? (5) What happens if signature verification fails — does the handler reject or continue?
>
> Identify any scenario where an unauthenticated attacker could inject false subscription events to modify a user's entitlements or trigger refund/credit processing.

**Security agent output:**

**Risk Level**: Critical

**Vulnerability 1 — Certificate validation fallback accepts any signature** (`appleCertificateValidation.ts:117-124`)

The certificate chain validation function returns `true` — accepts any signature — when the runtime lacks support for `crypto.X509Certificate`:

```typescript
if (!cert || !issuerCert) {
  return true;  // ACCEPTS ANY SIGNATURE
}
```

**Attack vector:**
1. Attacker crafts a fake App Store webhook notification with arbitrary payload (e.g., `DID_RENEW`, `EXPIRED`, `REFUND`)
2. Attacker generates their own JWS token with a fake certificate chain in the `x5c` header
3. If Lambda runtime is downgraded from Node 22 or the `X509Certificate` module is missing for any reason, the verification function returns `true` without validating the signature
4. The webhook handler processes the payload as legitimate, updating subscription status and entitlements

**Impact**: Subscription fraud (activate premium entitlements without payment), service disruption (expire active subscriptions via fake `EXPIRED`/`REVOKE` events), privilege escalation (upgrade free users to paid tiers), financial loss (trigger refund/credit processing). Failure is silent — the system operates normally with no indication that signature verification is broken.

**Fix**: Fail closed. Never accept an unverified signature:

```typescript
if (!cert || !issuerCert) {
  Logger().error('[AppleCert] X509Certificate not available — rejecting request');
  return false;  // was: return true
}
```

Also pin the Node.js runtime version in `package.json` and SST config. Any runtime downgrade must include re-testing of signature validation before deployment.

---

**Vulnerability 2 — Webhook endpoint has no authentication or IP allowlist** (lines 85-100)

The endpoint at `/v1/subscriptions/appstore/webhook` has no API key, no IP restriction, and no API Gateway authorizer. Any attacker on the internet can POST to it.

**Attack vector:** With Vulnerability 1 present, an attacker can send arbitrary subscription events from any IP with no credentials. Even with signature verification working correctly, lack of IP filtering enables replay attacks and brute-forcing notification UUIDs to bypass idempotency checks.

**Impact**: Public endpoint is fully exploitable. Bulk fake webhooks cause DynamoDB throttling, CloudWatch noise, incorrect entitlement state, and unwanted email notifications to users ("Your subscription has expired").

**Fix**: Add an IP allowlist for Apple's webhook IP ranges at the API Gateway resource policy level (infrastructure-level enforcement is preferable to application-level):

```typescript
const APPLE_WEBHOOK_IP_RANGES = ['17.0.0.0/8']; // verify current ranges from Apple docs

const requesterIP = event.requestContext?.http?.sourceIp;
if (!APPLE_WEBHOOK_IP_RANGES.some(range => isIPInRange(requesterIP, range))) {
  Logger().warn('[AppStore Webhook] Request from non-Apple IP', { requesterIP });
  return { statusCode: 403 };
}
```

---

**Vulnerability 3 — Bundle ID check is ineffective when signature verification is broken** (lines 130-139)

Bundle ID mismatch checking occurs after the JWS signature is already processed. If Vulnerability 1 is present, an attacker who knows the expected bundle ID (trivially discoverable from the mobile app binary) can craft a payload that passes both checks.

**Attack vector:** Attacker reads `com.demandgetter.app` from the app binary, crafts a JWS payload with the correct bundle ID and a forged signature. Fallback accepts it → bundle ID check passes → entitlements updated for an arbitrary user.

**Impact**: The bundle ID check provides zero defense when signature verification is broken. It is a weak layer that depends entirely on the upstream check working.

**Fix**: The primary fix is Vulnerability 1 (fail closed). Bundle ID validation is correctly placed as defense-in-depth but cannot be relied upon alone.

**Behavioral note**: Lines 146-153 intentionally allow sandbox receipts in production (for TestFlight). This is acceptable but ensure TestFlight builds never reach production users. Consider feature-flag gating to disable sandbox receipt acceptance in the production tier.

| Lines | Issue | Attack Vector | Priority |
|-------|-------|---------------|----------|
| 117-124 | Fallback accepts any signature | X509Certificate unavailable → signature validation no-op | Critical |
| 85-100 | No IP allowlist | Public endpoint accepts webhooks from anywhere | Critical |
| 130-139 | Bundle ID check after broken sig | Ineffective defense if sig validation bypassed | High |
| 146-153 | Sandbox/prod environment mixing | Test transactions accepted in production by design | Medium |

---

**Why this is better than default Claude:** Default Claude would flag the `return true` fallback as risky. This agent explains the specific trigger condition (Node.js runtime downgrade), traces the full attack chain from forged JWS to entitlement modification, distinguishes between the signature failure (catastrophic) and the bundle ID check (decorative without the signature fix), and notes the behavioral sandbox/prod mixing as a separate medium-risk finding that wasn't in the original question.
