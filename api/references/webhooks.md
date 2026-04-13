# Webhook Signature Verification

## Table of Contents
- [How it works](#how-it-works)
- [Verification algorithm](#verification-algorithm)
- [Raw body handling](#raw-body-handling)
- [Common mistakes](#common-mistakes)
- [Express middleware example](#express-middleware-example)
- [Next.js handler example](#nextjs-handler-example)

---

## How it works

When Celper delivers a webhook event, each HTTP request includes two headers:

| Header | Value | Example |
|--------|-------|---------|
| `X-Celper-Signature` | `sha256={hex_hmac}` | `sha256=a1b2c3...` |
| `X-Celper-Timestamp` | Unix seconds (integer) | `1712404800` |

The HMAC is computed over the concatenation of the Unix timestamp, a literal period (`.`), and the raw JSON body — i.e., the signed message is `"{timestamp}.{payload}"`. This binds the signature to a specific point in time, preventing replay attacks.

## Verification algorithm

```typescript
import crypto from 'crypto';

function verifyWebhook(payload: string, secret: string, signature: string, timestamp: number): boolean {
  // 1. Reject stale timestamps (>5 minutes)
  const now = Math.floor(Date.now() / 1000);
  if (Math.abs(now - timestamp) > 300) return false;

  // 2. Recompute HMAC over "{timestamp}.{payload}"
  const expected = 'sha256=' + crypto
    .createHmac('sha256', secret)
    .update(`${timestamp}.${payload}`)
    .digest('hex');

  // 3. Constant-time comparison (length guard + timingSafeEqual)
  if (signature.length !== expected.length) return false;
  return crypto.timingSafeEqual(Buffer.from(signature), Buffer.from(expected));
}
```

Use `crypto.timingSafeEqual` — never compare signatures with `===`.

## Raw body handling

Always read the request body as raw text *before* parsing as JSON. The signature is computed over the exact bytes Celper sent. If you parse to JSON and re-serialize, whitespace or key ordering may change, causing the HMAC to mismatch.

- **Next.js:** `const raw = await request.text();` then `JSON.parse(raw)` after verification
- **Express:** use `express.raw({ type: 'application/json' })` on the webhook route, or use the `verify` callback on `express.json()` to capture the raw buffer

## Common mistakes

These are the three most frequent causes of webhook signature failures:

1. **Missing timestamp in HMAC input** — computing HMAC over `payload` alone instead of `${timestamp}.${payload}`. This is the most common cause of signature mismatches.

2. **No timestamp staleness check** — without rejecting old timestamps (`abs(now - timestamp) > 300`), captured payloads can be replayed indefinitely.

3. **String comparison (`===`)** — vulnerable to timing attacks. Always use `crypto.timingSafeEqual` with a length guard. The function throws if buffer lengths differ, so check `.length` first.

## Express middleware example

```typescript
import crypto from 'crypto';
import express from 'express';

function createWebhookMiddleware(secret: string) {
  return (req: express.Request, res: express.Response, next: express.NextFunction) => {
    const signature = req.headers['x-celper-signature'] as string | undefined;
    const timestampStr = req.headers['x-celper-timestamp'] as string | undefined;

    if (!signature || !timestampStr) {
      return res.status(401).json({ error: 'Missing signature headers' });
    }

    const timestamp = parseInt(timestampStr, 10);
    if (isNaN(timestamp)) {
      return res.status(401).json({ error: 'Invalid timestamp' });
    }

    // rawBody must be preserved — see "Raw body handling" above
    const rawBody = typeof req.body === 'string' ? req.body : (req as any).rawBody?.toString('utf-8');
    if (!rawBody) {
      return res.status(500).json({ error: 'Raw body not available for verification' });
    }

    if (!verifyWebhook(rawBody, secret, signature, timestamp)) {
      return res.status(401).json({ error: 'Invalid webhook signature' });
    }

    next();
  };
}

// Usage:
// app.post('/webhooks/celper', express.raw({ type: '*/*' }), createWebhookMiddleware(SECRET), handler);
```

## Next.js handler example

```typescript
import { NextRequest, NextResponse } from 'next/server';

export async function POST(request: NextRequest) {
  const signature = request.headers.get('X-Celper-Signature');
  const timestampStr = request.headers.get('X-Celper-Timestamp');

  if (!signature || !timestampStr) {
    return NextResponse.json({ error: 'Missing signature headers' }, { status: 401 });
  }

  const timestamp = parseInt(timestampStr, 10);
  if (isNaN(timestamp)) {
    return NextResponse.json({ error: 'Invalid timestamp' }, { status: 401 });
  }

  // Read raw text FIRST — before any JSON parsing
  const payload = await request.text();

  if (!verifyWebhook(payload, process.env.CELPER_WEBHOOK_SECRET!, signature, timestamp)) {
    return NextResponse.json({ error: 'Invalid signature' }, { status: 401 });
  }

  // Signature verified — now parse
  const event = JSON.parse(payload);
  // ... handle event ...

  return NextResponse.json({ received: true }, { status: 200 });
}
```
