# Quickstart: Celper AI API

Get up and running with the Celper AI API in 5 minutes.

## 1. Get your API key

1. Log in to the Celper platform at `https://platform.celper.ai`
2. Go to **Settings > API Management**
3. Click **Generate API Key**
4. Select the scopes you need (at minimum: `read:jobs`, `read:candidates`, `write:candidates`)
5. Copy the key — it starts with `celp_` and is shown only once

## 2. Make your first request

Verify your key works by listing job positions:

```bash
curl -s https://api.celper.ai/v1/job-positions \
  -H "X-API-Key: celp_YOUR_KEY_HERE" | jq
```

Expected response:
```json
{
  "success": true,
  "data": [
    {
      "id": "abc123",
      "title": "Software Engineer",
      "status": "active",
      "createdAt": "2026-01-15T10:00:00.000Z"
    }
  ],
  "requestId": "550e8400-e29b-41d4-a716-446655440000",
  "pagination": { "cursor": null, "hasMore": false }
}
```

## 3. Import a candidate

```bash
curl -s -X POST https://api.celper.ai/v1/candidates \
  -H "X-API-Key: celp_YOUR_KEY_HERE" \
  -H "Content-Type: application/json" \
  -H "Idempotency-Key: $(uuidgen)" \
  -d '{
    "jobPositionId": "YOUR_JOB_ID",
    "firstName": "Jane",
    "lastName": "Doe",
    "email": "jane.doe@example.com",
    "cvText": "Jane Doe - Software Engineer with 5 years experience in TypeScript and React..."
  }' | jq
```

The `Idempotency-Key` header ensures you can safely retry this request if it fails mid-flight — the same key with the same body will return the cached result instead of creating a duplicate.

If `cvText` is provided, Celper will automatically score the CV against the job position requirements and return `cvScore` in the response.

### TypeScript example

```typescript
const API_KEY = 'celp_YOUR_KEY_HERE';
const BASE_URL = 'https://api.celper.ai/v1';

async function importCandidate(jobPositionId: string, candidate: {
  firstName: string;
  lastName: string;
  email: string;
  cvText?: string;
}) {
  const res = await fetch(`${BASE_URL}/candidates`, {
    method: 'POST',
    headers: {
      'X-API-Key': API_KEY,
      'Content-Type': 'application/json',
      'Idempotency-Key': crypto.randomUUID(), // or use the 'uuid' package in Node <19
    },
    body: JSON.stringify({ jobPositionId, ...candidate }),
  });

  const json = await res.json();

  if (!json.success) {
    throw new Error(`${json.error}: ${json.message}`);
  }

  return json.data; // { candidateId, status, cvScore, ... }
}
```

## 4. List candidates with pagination

Fetch all candidates for a job position, automatically walking through pages:

```typescript
async function listAllCandidates(jobPositionId: string) {
  const candidates = [];
  let cursor: string | undefined;

  do {
    const url = new URL(`${BASE_URL}/candidates`);
    url.searchParams.set('jobPositionId', jobPositionId);
    url.searchParams.set('limit', '200');
    if (cursor) url.searchParams.set('cursor', cursor);

    const res = await fetch(url, {
      headers: { 'X-API-Key': API_KEY },
    });
    const json = await res.json();

    if (!json.success) throw new Error(`${json.error}: ${json.message}`);

    candidates.push(...json.data);
    cursor = json.pagination?.hasMore ? json.pagination.cursor : undefined;
  } while (cursor);

  return candidates;
}
```

## 5. Set up a webhook

Register a webhook to get notified when interview analysis is complete:

```bash
curl -s -X POST https://api.celper.ai/v1/webhooks \
  -H "X-API-Key: celp_YOUR_KEY_HERE" \
  -H "Content-Type: application/json" \
  -d '{
    "url": "https://your-server.com/webhooks/celper",
    "events": ["analysis_ready", "candidate_status_changed"]
  }' | jq
```

The response includes a `secret` field — **store this securely**. You'll need it to verify that incoming deliveries are actually from Celper.

When a delivery arrives, it includes `X-Celper-Signature` and `X-Celper-Timestamp` headers. For the full verification algorithm and ready-to-use middleware, see **`references/webhooks.md`**.

## Next steps

- Browse all endpoints: `GET /v1/docs`
- Invite a candidate to interview: `POST /v1/candidates/{id}/invite`
- Fetch interview analysis: `GET /v1/candidates/{id}/interview-analysis`
- Monitor delivery history: `GET /v1/webhooks/{id}/deliveries`
