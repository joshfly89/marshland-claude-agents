# Security Agent Examples

The security agent reviews code the way an attacker reads it — looking for the path through, not the happy path. These examples show it catching vulnerabilities that pass code review and TypeScript and only surface in production.

---

## Example 1: Lambda IAM Permissions Too Wide in SST v3

**Input — infrastructure code sent for review:**

```typescript
// infra/api.ts
export const api = new sst.aws.Function("Api", {
  handler: "packages/functions/src/api.handler",
  link: [table, bucket],
  permissions: [
    {
      actions: ["dynamodb:*"],
      resources: ["*"],
    },
    {
      actions: ["s3:*"],
      resources: ["*"],
    },
  ],
});
```

**Security agent output:**

These permissions are account-wide. `dynamodb:*` on `*` means this Lambda can read, write, and delete any DynamoDB table in your AWS account — including your other SST apps' tables if they're in the same account (`prod`, `staging`, `dev`). Same for S3.

The blast radius if this function is compromised (SSRF, dependency confusion, a malicious npm package): your entire account's data.

Narrow to what the function actually needs:

```typescript
export const api = new sst.aws.Function("Api", {
  handler: "packages/functions/src/api.handler",
  link: [table, bucket],
  permissions: [
    {
      actions: [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:TransactWriteItems",
      ],
      resources: [table.arn, `${table.arn}/index/*`],
    },
    {
      actions: ["s3:GetObject", "s3:PutObject"],
      resources: [`${bucket.arn}/uploads/*`],
    },
  ],
});
```

SST's `link` already handles resource binding for most SDK calls — check whether these explicit permissions are even necessary before adding them. If you linked `table`, DynamoDB access for that table may already be granted.

Also: you have `s3:DeleteObject` missing if you need it, and `s3:ListBucket` if you're doing prefix listings — add only what you use. Least privilege isn't just a security rule; it's a documentation of what the function actually does.

---

**Why this is better than default Claude:** Default Claude flags the broad permissions and says to narrow them. This agent explains the specific cross-app blast radius in a multi-stage SST account, notes that `link` may already grant access (reducing the permissions you need to add manually), and frames least privilege as operational documentation — not just compliance theater.

---

## Example 2: JWT Decode Without Verification in a tRPC Context

**Input — code sent for review:**

```typescript
// packages/functions/src/trpc/context.ts
import jwt from 'jsonwebtoken';

export const createContext = async ({
  req,
}: CreateAWSLambdaContextOptions<APIGatewayProxyEventV2>) => {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return { userId: null, userEmail: null };
  }

  try {
    const decoded = jwt.decode(token) as { sub: string; email: string } | null;
    return {
      userId: decoded?.sub ?? null,
      userEmail: decoded?.email ?? null,
    };
  } catch {
    return { userId: null, userEmail: null };
  }
};
```

**Security agent output:**

**Critical.** `jwt.decode()` does not verify the signature. It base64-decodes the payload and returns it as-is. Any request with a crafted JWT — `{"sub": "user_of_your_choice", "email": "admin@yourcompany.com"}` — will pass context creation with those values treated as authenticated.

Every `protectedProcedure` in your tRPC router that trusts `ctx.userId` is now an authentication bypass. An attacker doesn't need credentials — they just need to forge a token.

Fix with `aws-jwt-verify`, which is purpose-built for Cognito and handles JWKS key rotation:

```typescript
import { CognitoJwtVerifier } from 'aws-jwt-verify';

const verifier = CognitoJwtVerifier.create({
  userPoolId: process.env.COGNITO_USER_POOL_ID!,
  tokenUse: 'access',
  clientId: process.env.COGNITO_CLIENT_ID!,
});

export const createContext = async ({
  req,
}: CreateAWSLambdaContextOptions<APIGatewayProxyEventV2>) => {
  const token = req.headers.authorization?.split(' ')[1];

  if (!token) {
    return { userId: null, userEmail: null };
  }

  try {
    const payload = await verifier.verify(token);
    return { userId: payload.sub, userEmail: payload.email ?? null };
  } catch {
    return { userId: null, userEmail: null };
  }
};
```

The verifier caches the JWKS in Lambda memory between invocations — no network call on warm starts.

Don't use `jwt.verify()` with a manually fetched public key unless you're handling JWKS rotation yourself. `aws-jwt-verify` does this correctly for Cognito.

---

**Why this is better than default Claude:** Default Claude would likely spot the `decode` vs `verify` distinction. This agent explains *exactly* what a forged token looks like, names the specific downstream impact (every `protectedProcedure` is bypassed), recommends `aws-jwt-verify` over `jsonwebtoken` for Cognito specifically, and explains the caching behavior that matters for Lambda cold start cost.

---

## Example 3: S3 Presigned Upload URL With Unconstrained Content-Type

**Input — code sent for review:**

```typescript
// packages/functions/src/uploads/getUploadUrl.ts
export const getUploadUrl = protectedProcedure
  .input(
    z.object({
      filename: z.string(),
      contentType: z.string(),
      folder: z.enum(['avatars', 'attachments']),
    })
  )
  .mutation(async ({ ctx, input }) => {
    const key = `${input.folder}/${ctx.userId}/${input.filename}`;

    const url = await getSignedUrl(
      s3Client,
      new PutObjectCommand({
        Bucket: process.env.BUCKET_NAME!,
        Key: key,
        ContentType: input.contentType,
      }),
      { expiresIn: 3600 }
    );

    return { url, key };
  });
```

**Security agent output:**

Three issues.

**Issue 1 — user-controlled ContentType enables stored XSS:** If your S3 bucket serves objects directly (or via CloudFront without `Content-Type` override), an attacker sets `contentType: "text/html"` and uploads an HTML file. If another user's browser fetches it, it executes as a page in the context of your CloudFront domain. That's stored XSS with your CDN domain as the origin.

Allowlist acceptable types server-side:

```typescript
const ALLOWED_CONTENT_TYPES: Record<string, string[]> = {
  avatars: ['image/jpeg', 'image/png', 'image/webp'],
  attachments: ['application/pdf', 'image/jpeg', 'image/png'],
};

const allowed = ALLOWED_CONTENT_TYPES[input.folder];
if (!allowed.includes(input.contentType)) {
  throw new TRPCError({ code: 'BAD_REQUEST', message: 'Content type not allowed' });
}
```

**Issue 2 — `filename` is unsanitized:** `input.filename` goes directly into the S3 key. While S3 doesn't have path traversal in the traditional sense, filenames like `../../../shared/config` create keys outside the user's folder prefix. Other users with read access to `shared/` can see it. Sanitize before key construction:

```typescript
const safeFilename = path.basename(input.filename).replace(/[^a-zA-Z0-9._-]/g, '_');
const key = `${input.folder}/${ctx.userId}/${safeFilename}`;
```

**Issue 3 — 1-hour expiry for an interactive upload:** 3600 seconds is long for a presigned upload URL. If the URL leaks (logs, analytics, shared accidentally), an attacker has an hour to upload to your bucket under the authenticated user's path. Use 300 seconds — long enough for any upload, short enough to limit exposure.

---

**Why this is better than default Claude:** Default Claude would flag the unsanitized filename. This agent catches all three, explains the stored XSS vector specific to CloudFront/S3 serving (not just "could be dangerous"), gives the allowlist pattern keyed to the existing `folder` enum, and gives a concrete expiry recommendation with the reasoning.
