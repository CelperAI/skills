---
name: celper
description: 'Celper AI REST API integration guide. Invoke-only via /celper. Use this whenever building an integration against the Celper API, getting started with Celper, importing or managing candidates via API, listing job positions, setting up webhooks, verifying webhook signatures, handling API authentication or scopes, dealing with pagination, rate limiting, idempotency, or debugging API errors — even if the user does not say "API" explicitly. Covers all endpoints, authentication, error handling, and client-side best practices.'
metadata:
  author: celper-ai
  version: "2.0.0"
---

# Celper AI API Integration Guide

This skill helps you integrate with the Celper AI public REST API. It covers authentication, all available endpoints, pagination, idempotency, rate limiting, webhook consumption, and error handling — everything you need to build a working integration.

**Base URL:** `https://api.celper.ai/v1`

**API docs:** `GET /v1/docs` returns the OpenAPI 3.1 specification. A Swagger UI is also available at that endpoint.

For a quick walkthrough of making your first requests, read **`references/quickstart.md`**.

## Authentication

Every endpoint (except `GET /v1/health` and `GET /v1/docs`) requires an API key.

**Header:** `X-API-Key`
**Key format:** `celp_` followed by 32 lowercase hex characters (37 characters total).

```
X-API-Key: celp_a1b2c3d4e5f6a1b2c3d4e5f6a1b2c3d4
```

Generate API keys from the Celper platform dashboard under **Settings > API Management**. Keys can optionally be restricted to specific IP addresses via an allowlist.

If authentication fails, you'll receive one of these errors:

| Error | Meaning |
|-------|---------|
| `UNAUTHORIZED` (401) | Key is missing, malformed, revoked, or expired |
| `IP_NOT_ALLOWED` (403) | Your IP is not in the key's allowlist |
| `API_DISABLED` (403) | API access is disabled for this company |
| `ACCOUNT_SUSPENDED` (403) | Company account is suspended |
| `FORBIDDEN` (403) | The user who created this key no longer has access |

### Scopes

Each API key is granted a set of scopes that control what it can access. Request the scopes you need when generating the key.

| Scope | What it allows |
|-------|---------------|
| `read:jobs` | List and retrieve job positions |
| `read:candidates` | Fetch candidate data, CV scores, interview analysis |
| `write:candidates` | Import, update, and delete candidates |
| `action:invite` | Send interview invitations to candidates |
| `action:decide` | Select or reject candidates |
| `manage:webhooks` | Create, update, delete, and test webhooks |

If your key lacks a required scope, you'll get `SCOPE_INSUFFICIENT` (403).

The `manage:webhooks` scope additionally requires your user role to be **owner** or **admin** — member-level users cannot manage webhooks even with the scope.

## Endpoint Reference

### System

```
GET  /v1/health                              No auth required
GET  /v1/docs                                OpenAPI spec / Swagger UI
```

### Job Positions

```
GET  /v1/job-positions                       Scope: read:jobs
GET  /v1/job-positions/{jobPositionId}        Scope: read:jobs
```

**Query params** for listing: `limit` (1-200, default 50), `cursor`, `privacy` (`public` | `private`).

### Candidates

```
GET    /v1/candidates                                        Scope: read:candidates
POST   /v1/candidates                                        Scope: write:candidates
POST   /v1/candidates/bulk                                   Scope: write:candidates
GET    /v1/candidates/{candidateId}                          Scope: read:candidates
PATCH  /v1/candidates/{candidateId}                          Scope: write:candidates
DELETE /v1/candidates/{candidateId}                          Scope: write:candidates
GET    /v1/candidates/{candidateId}/status                   Scope: read:candidates
GET    /v1/candidates/{candidateId}/cv-score                 Scope: read:candidates
GET    /v1/candidates/{candidateId}/cv-analysis              Scope: read:candidates
GET    /v1/candidates/{candidateId}/cv-document              Scope: read:candidates
GET    /v1/candidates/{candidateId}/interview-analysis       Scope: read:candidates
POST   /v1/candidates/{candidateId}/invite                   Scope: action:invite
POST   /v1/candidates/{candidateId}/select                   Scope: action:decide
POST   /v1/candidates/{candidateId}/reject                   Scope: action:decide
```

**Listing** requires `jobPositionId` as a query param. Optional filters: `status`, `importSource`.

**Single candidate endpoints** (GET/PATCH/DELETE on `/{candidateId}`) also require `jobPositionId` as a query param.

**Import body** (POST `/v1/candidates`):
```json
{
  "jobPositionId": "string (required)",
  "firstName": "string (required, max 100)",
  "lastName": "string (required, max 100)",
  "email": "string (required, valid email)",
  "phone": "string (optional, max 50)",
  "cvText": "string (optional, max 100k chars)",
  "notes": "string (optional, max 10k chars)",
  "skipAnalysis": false
}
```

If `cvText` is provided and `skipAnalysis` is false (the default), Celper will automatically analyze the CV against the job position requirements.

**Bulk import** (POST `/v1/candidates/bulk`): wraps a `candidates` array (1-500 items) with a shared `jobPositionId` and optional `skipAnalysis`.

**Invite body** (POST `.../invite`):
```json
{
  "jobPositionId": "string (required)",
  "emailSubject": "string (optional, max 200)",
  "emailBody": "string (optional, max 5000)"
}
```

**Select/Reject body**: `{ "jobPositionId": "string" }`

**Update body** (PATCH `/{candidateId}`): any subset of `firstName`, `lastName`, `email`, `phone`, `notes`. At least one field must be provided.

### Webhooks

```
GET    /v1/webhooks                                  Scope: manage:webhooks
POST   /v1/webhooks                                  Scope: manage:webhooks
GET    /v1/webhooks/{webhookId}                      Scope: manage:webhooks
PATCH  /v1/webhooks/{webhookId}                      Scope: manage:webhooks
DELETE /v1/webhooks/{webhookId}                      Scope: manage:webhooks
POST   /v1/webhooks/{webhookId}/test                 Scope: manage:webhooks
POST   /v1/webhooks/{webhookId}/rotate-secret        Scope: manage:webhooks
GET    /v1/webhooks/{webhookId}/deliveries           Scope: manage:webhooks
```

**Register body**:
```json
{
  "url": "https://your-server.com/webhooks/celper",
  "events": ["analysis_ready", "candidate_status_changed"],
  "statusFilters": ["interviewed", "selected"]
}
```

**Available events:**

| Event | Fires when |
|-------|-----------|
| `analysis_ready` | CV or interview analysis completes |
| `candidate_status_changed` | Candidate moves to a new status |
| `cv_analysis_complete` | CV parsing and scoring finishes |
| `bulk_import_complete` | A bulk import job finishes |

URLs must use HTTPS. Private/internal addresses (localhost, RFC 1918 ranges, link-local, cloud metadata endpoints) are rejected.

When you register a webhook, the response includes a `secret` field. Store this securely — you'll need it to verify delivery signatures. For signature verification details, read **`references/webhooks.md`**.

## Response Format

Every response follows a consistent envelope.

**Success:**
```json
{
  "success": true,
  "data": { ... },
  "requestId": "uuid",
  "pagination": { "cursor": "base64_or_null", "hasMore": true }
}
```

**Error:**
```json
{
  "success": false,
  "error": "ERROR_CODE",
  "message": "Human-readable description",
  "requestId": "uuid"
}
```

**Standard response headers:**
- `X-Request-Id` — unique request identifier (matches `requestId` in body)
- `X-API-Version` — always `v1`
- `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

## Error Codes

| Code | HTTP | Meaning | What to do |
|------|------|---------|-----------|
| `UNAUTHORIZED` | 401 | Invalid, missing, revoked, or expired API key | Check your key is correct and active |
| `SCOPE_INSUFFICIENT` | 403 | Key lacks the required scope | Generate a new key with the needed scope |
| `IP_NOT_ALLOWED` | 403 | Caller IP not in the key's allowlist | Add your IP to the allowlist in Settings |
| `API_DISABLED` | 403 | API access is disabled for this company | Enable API access in Settings > API Management |
| `ACCOUNT_SUSPENDED` | 403 | Company account is suspended | Contact Celper support |
| `FORBIDDEN` | 403 | Key creator no longer belongs to the company | Generate a new key with an active user |
| `NOT_FOUND` | 404 | Resource does not exist | Verify the ID and that `jobPositionId` is correct |
| `VALIDATION_ERROR` | 400 | Request body or query param validation failed | Check `details` array in response for field-level errors |
| `INVALID_BODY` | 400 | Request body is not valid JSON | Ensure `Content-Type: application/json` and valid JSON |
| `DUPLICATE_CANDIDATE` | 409 | Email already exists for this job position | Use a different email or fetch the existing candidate |
| `IDEMPOTENCY_CONFLICT` | 409 | Same idempotency key used with a different body | Use a new idempotency key for different requests |
| `RATE_LIMITED` | 429 | Rate limit exceeded | Wait and retry (see Rate Limiting below) |
| `INTERNAL_ERROR` | 500 | Unexpected server error | Retry with backoff; contact support if persistent |

## Pagination

All list endpoints use cursor-based pagination sorted by `createdAt` descending (newest first).

- **Default page size:** 50
- **Maximum page size:** 200
- Pass `cursor` from the previous response's `pagination.cursor` to get the next page
- When `pagination.hasMore` is `false`, you've reached the end

**Pagination loop:**
```typescript
let cursor: string | undefined;

do {
  const url = new URL('https://api.celper.ai/v1/candidates');
  url.searchParams.set('jobPositionId', jobId);
  url.searchParams.set('limit', '200');
  if (cursor) url.searchParams.set('cursor', cursor);

  const res = await fetch(url, { headers: { 'X-API-Key': apiKey } });
  const json = await res.json();

  for (const candidate of json.data) {
    // process candidate
  }

  cursor = json.pagination?.hasMore ? json.pagination.cursor : undefined;
} while (cursor);
```

## Idempotency

POST endpoints for candidate import and actions (invite, select, reject) support idempotency via the `Idempotency-Key` header. This lets you safely retry requests on network failures without creating duplicates.

```
Idempotency-Key: your-unique-uuid-here
```

How it works:
1. **New key** — request is processed normally; the response is cached for 24 hours
2. **Same key + same body** — the cached response is returned without reprocessing
3. **Same key + different body** — `409 IDEMPOTENCY_CONFLICT` error

Use a UUID or another unique identifier (e.g., your ATS record ID) as the key. The key is scoped to your company — different companies can reuse the same key value.

## Rate Limiting

Limits are enforced per company with a 60-second sliding window. The tightest applicable limit applies.

**Default limits:**

| Category | Requests/minute |
|----------|----------------|
| Global (all endpoints) | 60 |
| Read operations | 40 |
| Write operations | 20 |
| Actions (invite/select/reject) | 30 |
| Analysis endpoints | 15 |
| Storage (documents) | 10 |
| Bulk import | 3 |
| Webhook management | 20 |

**Response headers** (present on every response):

| Header | Meaning |
|--------|---------|
| `X-RateLimit-Limit` | Max requests in current window |
| `X-RateLimit-Remaining` | Requests remaining |
| `X-RateLimit-Reset` | Unix timestamp when the window resets |
| `Retry-After` | Seconds to wait (only on 429 responses) |

### Handling rate limits

When you receive a `429` response, read the `Retry-After` header and wait that many seconds before retrying. For sustained high-throughput workloads, implement exponential backoff:

```typescript
async function fetchWithRetry(url: string, options: RequestInit, maxRetries = 3): Promise<Response> {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    const res = await fetch(url, options);

    if (res.status === 429) {
      const retryAfter = parseInt(res.headers.get('Retry-After') || '5', 10);
      const delay = retryAfter * 1000 * Math.pow(2, attempt); // exponential backoff
      await new Promise(r => setTimeout(r, delay));
      continue;
    }

    return res;
  }

  throw new Error('Rate limit retries exhausted');
}
```

For bulk operations, prefer the `/v1/candidates/bulk` endpoint (up to 500 candidates per request) over individual imports to stay well within limits.

## Webhook Signature Verification

When you receive a webhook delivery, you should verify its signature to confirm it came from Celper and hasn't been tampered with. Each delivery includes:

- `X-Celper-Signature` header: `sha256={hex_hmac}`
- `X-Celper-Timestamp` header: Unix seconds

The HMAC is computed over `"{timestamp}.{raw_body}"` using your webhook secret. Timestamps older than 5 minutes should be rejected.

For the full verification algorithm, code examples (TypeScript, Express, Next.js), and common mistakes that cause signature mismatches, read **`references/webhooks.md`**.
