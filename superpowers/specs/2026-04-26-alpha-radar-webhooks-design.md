# Alpha & Radar Webhooks — Design

**Date:** 2026-04-26
**Status:** Draft
**Audience:** External API users (developers, traders, integrations)

## 1. Goal

Re-enable a public outbound webhook product so external customers can receive real-time delivery of:

- **Radar** — internal `alpha_insights` Redis stream (high-impact market intelligence events).
- **Alpha** — internal `signal_created` Redis stream (actionable trade signals with entry, SL, TP, leverage, RR).

Each successful enqueue charges **$0.50** from the user's credit balance via the canonical `credit.service.deductBalance` ledger path. Pre-paid pricing makes the product self-funding under abuse: an attacker who tries to flood deliveries through their own webhook simply funds Gigabrain.

A previous version of this product was removed in commit `beed758` (Feb 15 2026) after a DDoS incident. The design below restores the deleted module, hardens it against the failure modes that allowed the prior attack to succeed, and adapts it to the new streaming architecture introduced by PR #184 ([feat: generic webhook system + signal dispatcher](https://github.com/GigabrainGG/intelligence-monorepo/pull/184)).

## 2. Out of scope (v1)

- **`signal_outcomes` feed** — outcomes (open/active/win/lose) are deterministic from the entry/SL/TP shipped in `signal_created` plus current price; users can compute these locally. Not exposed in v1.
- **Webhook batching / digest** — one event per HTTP POST.
- **Replay API** — users can re-fetch from the GraphQL/REST alpha + signal endpoints if needed.
- **Per-feed pricing tiers** — both feeds are priced equally at $0.50.
- **Webhook subscriptions for daemon agents** — daemon agents already have their own internal webhook system from PR #184. This design is the **public** counterpart, not a replacement.

## 3. Architecture

```
[alpha_processor] ─xadd─> alpha_insights (existing)
[signal_processor] ─xadd─> signal_created (existing, from PR #184)
                                │
                                ↓ (separate consumer group: "public-webhook-dispatcher")
                  ┌─────────────────────────────────┐
                  │   NestJS PublicWebhookConsumer  │  ← reads both streams
                  └─────────────────────────────────┘
                                ↓
              find active subscriptions matching `feed`
                                ↓
              for each: deductBalance($0.50) ──┬─ ok    → enqueue Bull job
                                               └─ fail  → suspend subscription
                                ↓
                  ┌─────────────────────────────────┐
                  │   Bull queue: webhook-delivery  │
                  └─────────────────────────────────┘
                                ↓
              POST {url} with HMAC sig + retry policy
              → record outcome in webhook_deliveries
              → update consecutive_failures, auto-suspend if threshold
```

Two consumer groups read each stream — Python `signal_dispatcher` for daemon-internal use (existing) and NestJS `public-webhook-dispatcher` for public API (new). Independent, no contention.

## 4. Data model

Both tables live in the `users` database (where credit ledger and rate-limit state live). Keeps charging atomic.

### `webhook_endpoints`

```sql
CREATE TABLE webhook_endpoints (
  id              UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  user_id         UUID NOT NULL REFERENCES users(id) ON DELETE CASCADE,
  url             TEXT NOT NULL,                 -- HTTPS user URL
  name            TEXT NOT NULL,                 -- user label, 1-64 chars
  feed            TEXT NOT NULL,                 -- 'radar' | 'alpha'
  secret_key      TEXT NOT NULL,                 -- HMAC secret (plaintext, see §8.2)
  is_active       BOOLEAN NOT NULL DEFAULT TRUE,
  suspension_reason TEXT,                        -- 'balance' | 'failures' | NULL
  consecutive_failures INT NOT NULL DEFAULT 0,
  total_deliveries INT NOT NULL DEFAULT 0,
  failed_deliveries INT NOT NULL DEFAULT 0,
  last_delivery_at TIMESTAMPTZ,
  created_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  updated_at      TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  deleted_at      TIMESTAMPTZ
);

CREATE INDEX idx_webhook_endpoints_user ON webhook_endpoints(user_id) WHERE deleted_at IS NULL;
CREATE INDEX idx_webhook_endpoints_feed_active ON webhook_endpoints(feed) WHERE is_active = TRUE AND deleted_at IS NULL;
```

The partial index on `(feed)` is the hot read path for the dispatcher: "all active subscriptions for feed=alpha".

### `webhook_deliveries`

```sql
CREATE TABLE webhook_deliveries (
  id                   UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  webhook_endpoint_id  UUID NOT NULL REFERENCES webhook_endpoints(id) ON DELETE CASCADE,
  event_id             TEXT NOT NULL,            -- alpha_id or signal_id (idempotency)
  feed                 TEXT NOT NULL,            -- 'radar' | 'alpha'
  status               TEXT NOT NULL,            -- 'pending' | 'success' | 'failed'
  failure_kind         TEXT,                     -- 'balance' | 'enqueue' | 'http_4xx' | 'http_5xx' | 'timeout' | 'tls' | 'dns' | 'connection' | NULL
  http_status_code     INT,
  response_time_ms     INT,
  error_message        TEXT,
  charged_amount       NUMERIC(10,4) NOT NULL,   -- 0.5000 or 0.0000
  refunded_amount      NUMERIC(10,4) NOT NULL DEFAULT 0,  -- non-zero when refunded by sweeper or enqueue-fail path
  attempt_count        INT NOT NULL DEFAULT 0,
  payload              JSONB NOT NULL,           -- what we POST'd
  payload_version      INT NOT NULL DEFAULT 1,   -- bump when payload contract changes
  created_at           TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  completed_at         TIMESTAMPTZ
);

CREATE INDEX idx_webhook_deliveries_pending_sweep
  ON webhook_deliveries(created_at)
  WHERE status = 'pending';

CREATE UNIQUE INDEX uq_webhook_delivery_event ON webhook_deliveries(webhook_endpoint_id, event_id);
CREATE INDEX idx_webhook_deliveries_endpoint_time ON webhook_deliveries(webhook_endpoint_id, created_at DESC);
```

The unique index on `(webhook_endpoint_id, event_id)` is the **idempotency guarantee**: if the Redis consumer reprocesses a message (e.g. consumer crashed before `xack`), the second insert fails with `unique_violation` and we skip — no double-charge.

A nightly cron purges deliveries older than 30 days.

### Feed → stream mapping

```ts
// backend/src/webhooks/constants.ts
export const FEED_TO_STREAM = {
  radar: 'alpha_insights',
  alpha: 'signal_created',
} as const;

export type Feed = keyof typeof FEED_TO_STREAM;
```

User-facing terms (`radar`, `alpha`) deliberately invert the historical internal naming — `alpha_insights` was renamed to "Radar" when actionable trade signals shipped, and "Alpha" was promoted to the more valuable signal product. This module is the bridge; never let internal stream names leak into payloads or API responses.

## 5. Public API

CRUD lives at `/v1/webhooks` behind `ApiKeyGuard` (programmatic), and behind `JwtAuthGuard` GraphQL for the dashboard UI. Same service layer for both.

### REST

**Preconditions** (apply to `POST /v1/webhooks` and to `PATCH /v1/webhooks/:id` when transitioning `isActive=true`):
- User must have at least one `credit_ledger` entry with `type='purchase'` AND `source ∈ {coinbase,stripe,moonpay}`.
- Current `creditBalance ≥ $5.00`.
- See §8.1 for full rationale.

| Method | Path | Body / Query | Returns |
|---|---|---|---|
| `POST` | `/v1/webhooks` | `{url, name, feed}` | `{webhook, secret}` — secret only shown once |
| `GET` | `/v1/webhooks` | — | `[webhook, …]` |
| `GET` | `/v1/webhooks/:id` | — | `webhook` |
| `PATCH` | `/v1/webhooks/:id` | `{url?, name?, isActive?}` (feed immutable) | `webhook` |
| `DELETE` | `/v1/webhooks/:id` | — | `204` |
| `POST` | `/v1/webhooks/:id/rotate-secret` | — | `{secret}` |
| `POST` | `/v1/webhooks/:id/test` | — | `{deliveryId}` — synthetic event, free |
| `GET` | `/v1/webhooks/:id/deliveries` | `?limit=50&cursor=…` | paginated delivery log |

### GraphQL (frontend)

Mirrors the above as `webhooks`, `webhook(id)`, `createWebhook`, `updateWebhook`, `deleteWebhook`, `webhookDeliveries(webhookId)`, `rotateWebhookSecret(id)`, `testWebhook(id)`.

### Validation (creation/update)

- `url`: `https://` only (HTTP allowed in dev profile only). Must parse cleanly. Max 2048 chars. **DNS resolution check** rejects:
  - RFC1918 ranges (`10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`)
  - Loopback (`127.0.0.0/8`, `::1`)
  - Link-local (`169.254.0.0/16`)
  - Cloud metadata IPs (`169.254.169.254`)
  - This is **SSRF defense** — without it, an attacker could point webhooks at our own internal services.
- `name`: 1-64 chars, `[a-zA-Z0-9 _-]`.
- `feed`: must be `radar` or `alpha`.
- `feed` is **immutable on update**. Force delete + recreate to change feeds. Avoids race where in-flight events are dispatched against the wrong feed.

### Rate limits (CRUD)

- 60 req/min per API key (existing `RateLimitService` default).
- **Hard cap: 5 active webhooks per user** (matches old code; conservative for v1).
- `POST /test`: 10 calls/hour per webhook.

### Test endpoint

Sends a synthetic payload to the user's URL with `X-Gigabrain-Test: 1` header so they can distinguish from real events. Free (no charge), bypasses Redis stream entirely. Useful for setup/debug and reduces support burden.

**Caps:** 10 calls/hour per webhook, **30 calls/hour per user account** (across all their webhooks). Prevents test endpoint from becoming an abuse vector.

## 6. Stream consumer architecture

### Consumer pattern

A NestJS singleton service `WebhookStreamConsumer` (decorated `@Injectable`, started via `OnModuleInit`) opens an `ioredis` connection and runs an async loop:

```ts
// pseudocode
const STREAMS = ['alpha_insights', 'signal_created'];
const GROUP = 'public-webhook-dispatcher';
const CONSUMER = `nestjs-${process.env.HOSTNAME ?? 'unknown'}`;

await ensureConsumerGroup(STREAMS, GROUP, '$', { mkstream: true });

while (running) {
  const batches = await redis.xreadgroup(
    'GROUP', GROUP, CONSUMER,
    'COUNT', 50, 'BLOCK', 5000,
    'STREAMS', ...STREAMS, '>', '>',
  );
  if (!batches) continue;
  for (const [stream, messages] of batches) {
    const feed = STREAM_TO_FEED[stream];  // 'radar' | 'alpha'
    for (const [id, data] of messages) {
      try {
        await this.dispatcher.dispatch(feed, id, data);
        await redis.xack(stream, GROUP, id);
      } catch (e) {
        logger.error(`failed to dispatch ${id}: ${e.message}`);
        // do NOT xack — message will be reclaimed and retried by a future xreadgroup with id '0'
      }
    }
  }
}
```

**Consumer group naming:** `public-webhook-dispatcher`. Distinct from `signal_processor`, `workflow-engine`, `signal_dispatcher` so each consumer gets its own copy.

**Horizontal scaling:** when running multiple NestJS replicas, each replica is a distinct *consumer* in the same group, so Redis fans messages between them. Pending entries for a crashed consumer can be claimed by a sibling via `XAUTOCLAIM`. v1 ships with a single replica; multi-replica claim logic is deferred unless we hit throughput limits.

**Pending entries on restart:** at startup, before the main loop, run `XPENDING` and process any unacked entries the previous consumer instance owned (via `XCLAIM`).

**Failure behavior:** if `dispatch` throws, we don't `xack`. The message stays in pending. On next iteration we read `>` (only new messages), so retry happens via the pending-claim loop on next restart, or via a periodic `XAUTOCLAIM` job (every 60s, claim entries idle > 5 minutes).

### Dispatch logic

A single event from the stream fans out to **every** matching subscription. Each subscription is an independent delivery and is charged independently — if a user has 2 webhooks both subscribed to `alpha`, they pay $1.00 per event (deliberate, mirrors Stripe's per-endpoint billing model).

```ts
async dispatch(feed: Feed, eventId: string, data: Record<string, string>) {
  const subs = await this.webhookService.findActiveByFeed(feed);
  if (subs.length === 0) return;

  // Per-subscription, sequential within a single event. The Promise.allSettled
  // pattern would parallelize Postgres writes from a single Node process — fine
  // at low scale, but risks contention. v1 ships sequential; revisit if
  // stream throughput exceeds ~50 subs × 100 events/min.
  // Deterministic ordering for replay-debuggability.
  for (const sub of subs.sort((a, b) => a.createdAt.getTime() - b.createdAt.getTime())) {
    await this.deliverOne(sub, feed, eventId, data);
  }
}

async deliverOne(sub: WebhookEndpoint, feed: Feed, eventId: string, data: any) {
  // SINGLE TRANSACTION: idempotency row + charge.
  // If we crash after commit but before queue.add, the sweeper (§7.5) will
  // refund and mark the delivery failed — no permanently-overcharged user.
  const result = await this.dataSource.transaction(async (tx) => {
    const exists = await tx.findOne(WebhookDelivery, {
      where: { webhookEndpointId: sub.id, eventId },
    });
    if (exists) return { skipped: true };  // duplicate event for this endpoint

    // Charge via canonical credit_ledger path. The deductBalance call must
    // accept an external transaction handle so it joins the same txn.
    const charged = await this.creditService.deductBalance(sub.userId, 0.5, {
      reason: 'webhook_delivery',
      metadata: { webhookId: sub.id, feed, eventId },
      tx,  // join existing transaction
    });
    if (!charged.ok) return { insufficient: true };

    const delivery = await tx.save(WebhookDelivery, {
      webhookEndpointId: sub.id,
      eventId, feed,
      status: 'pending',
      payload: buildPayload(feed, data),
      chargedAmount: 0.5,
    });
    return { delivery };
  });

  if (result.skipped) return;
  if (result.insufficient) {
    await this.webhookService.suspend(sub.id, 'balance');
    return;  // no delivery row created — nothing to mark failed
  }

  // Enqueue OUTSIDE the txn (Bull/Redis is not transactional with Postgres).
  // If this fails, sweeper will catch it.
  try {
    await this.queue.add('deliver', { deliveryId: result.delivery.id }, retryOpts);
  } catch (e) {
    // Best-effort immediate refund + mark. If THIS fails, sweeper still catches.
    await this.refundAndMarkFailed(result.delivery.id, 'enqueue', e.message);
    throw e;  // re-throw so consumer doesn't xack
  }
}
```

**Why this ordering works:**

1. **Duplicate event rejection** is the first check inside the txn. If the row exists, we skip without charging. Re-delivery on Redis stream replay is safe.
2. **Charge + delivery row are atomic.** Either both happen or neither does. No "charged but no delivery" state. No "delivered but no charge" state.
3. **Enqueue is outside the txn.** This is the unavoidable gap: a process crash between `commit` and `queue.add` leaves a `pending` delivery with no Bull job. The **sweeper job** (§7.5) reconciles by refunding and marking these `failed` after a grace window.
4. **`creditService.deductBalance` must support transaction injection.** This is a real change to the existing service — currently it manages its own queryRunner. The implementation needs to optionally accept a `tx` parameter; if not provided, it opens its own transaction (preserving existing call sites).

## 7. Dispatcher (Bull queue)

### Job payload

```ts
{ deliveryId: string }
```

Minimal — the Bull job just carries the delivery row PK; everything else is read from Postgres. Keeps the queue light and lets us see/cancel in the BullBoard UI.

### Retry policy

```ts
{
  attempts: 3,
  backoff: { type: 'exponential', delay: 5000 },  // 5s, 25s, 125s
  removeOnComplete: 100,
  removeOnFail: 500,
}
```

3 attempts, exponential backoff. **No re-charging on retry** — user paid once for the event, retries are on us.

### Retryable vs permanent failures

| Outcome | Treatment |
|---|---|
| 2xx | Success. |
| 3xx | Permanent fail (we don't follow redirects). |
| 4xx | Permanent fail (no retry — the user's URL rejected). |
| 5xx | Retry. |
| Timeout (10s) | Retry. |
| Connection error / DNS / TLS | Retry. |

After 3 failed attempts:
- Mark `webhook_deliveries.status = 'failed'`.
- `consecutive_failures += 1` on the endpoint.
- If `consecutive_failures >= 10`: auto-suspend with `suspension_reason = 'failures'` and email the user.

On success: `consecutive_failures = 0`, `total_deliveries += 1`, `last_delivery_at = NOW()`.

The 1000/hour per-webhook delivery cap (§8.5) is enforced **only at dispatch time**, never on retries — once a delivery is enqueued, retries are owed to the user. Hitting the cap after the initial enqueue does not affect Bull jobs already in flight.

### 7.5 Reconciliation sweeper

A NestJS `@Cron('*/1 * * * *')` job (every 60s) reconciles drift:

```ts
async sweepStuckPending() {
  // Find deliveries that have been 'pending' for > 5 minutes
  const stuck = await this.deliveryRepo.find({
    where: { status: 'pending', createdAt: LessThan(fiveMinutesAgo) },
    take: 200,
  });

  for (const d of stuck) {
    // Check Bull for a job with this deliveryId
    const job = await this.queue.getJob(d.id);  // or scan active+waiting+delayed
    if (job && !job.isCompleted() && !job.isFailed()) {
      continue;  // job alive, leave it alone
    }
    // No live job exists (process died after commit, before enqueue —
    // OR Bull lost the job). Refund the user, mark the delivery failed.
    await this.refundAndMarkFailed(d.id, 'enqueue', 'sweeper: no live bull job');
  }
}
```

5-minute grace window is conservative — Bull retries with our backoff schedule complete in < 3 minutes. The sweeper is a safety net, not the primary path.

**Critical invariant:** the Bull processor MUST transition `webhook_deliveries.status` from `'pending'` to `'success'` or `'failed'` *within* the same job execution, before the worker returns/throws. Otherwise: HTTP succeeds → process crashes before status update → Bull eventually removes the job via `removeOnFail` → sweeper finds `pending` row, sees `getJob(id)` returns null, and incorrectly refunds a delivery that was actually successful.

Implementation rules to enforce this:
1. Wrap the HTTP call + status update in a `try/finally`. The `finally` block writes a status (defaulting to `'failed' / failure_kind='processor_crash'`) so even an unexpected throw transitions the row.
2. The status update is a single atomic `UPDATE` (no read-modify-write).
3. If the status update itself throws, re-throw — let Bull mark the job failed and retry, where the idempotent processor will read `status='pending'` and try again. (The status update is the only line of defense; if it can't reach Postgres, the system is degraded regardless.)

### 7.6 XAUTOCLAIM for stream resilience

A second cron `@Cron('*/1 * * * *')` claims pending Redis stream entries that have been idle > 5 minutes (i.e., a consumer started processing them and crashed):

```ts
for (const stream of ['alpha_insights', 'signal_created']) {
  await redis.xautoclaim(
    stream, GROUP, CONSUMER, /* min-idle-time */ 300_000, '0',
    'COUNT', 50,
  );
  // Reclaimed entries get re-processed via the same dispatch path,
  // and the idempotency row blocks duplicates.
}
```

Without this, a single crash could leave entries in pending forever. With it, recovery is automatic within 5 minutes.

**Multi-replica note:** when v1's single-replica assumption is relaxed (deferred per §6), both this cron and the §7.5 sweeper run on every replica. The sweeper is naturally idempotent (its `UPDATE` is conditional on `status='pending'`). `XAUTOCLAIM` from N replicas is also Redis-side serialized, but wastes work — wrap both crons in a Redis-based distributed lock (e.g., `redlock`) before scaling out. Single-replica v1 doesn't need this.

### Outbound HTTP

- `axios` with `timeout: 10000`, `validateStatus: () => true` (we treat status manually).
- `User-Agent: GigabrainWebhook/1.0`.
- HMAC signing via `WebhookSigningService` (resurrected verbatim from `beed758^`).

### Payload shape

```json
{
  "id": "<delivery_id>",
  "type": "webhook.delivery",
  "feed": "alpha",                 // or "radar"
  "event_id": "<alpha_id or signal_id>",
  "delivered_at": "2026-04-26T12:34:56Z",
  "data": { /* feed-specific */ }
}
```

**Radar feed `data`:**
```json
{
  "id": "<alpha_id>",
  "title": "...",
  "insight": "...",
  "category": "market_structure",
  "impact_rating": 4,
  "directional_bias": "bullish",
  "confidence": "high",
  "affected_tokens": ["BTC"]
}
```

**Alpha feed `data`:**
```json
{
  "id": "<signal_id>",
  "token": "BTC",
  "direction": "long",
  "entry": { "price": 67200, "type": "limit" },
  "stop_loss": { "price": 65800 },
  "take_profits": [{ "price": 70000, "size_pct": 100 }],
  "leverage": 3,
  "risk_reward": 2.0,
  "rationale": "...",
  "confidence": "high"
}
```

Both shapes deliberately strip Gigabrain-internal fields (deduplication keys, vector IDs, internal scoring metadata). What ships is a stable public contract.

### Headers

```
Content-Type: application/json
User-Agent: GigabrainWebhook/1.0
X-Gigabrain-Signature: t=1714137296,v1=<hmac_sha256>
X-Gigabrain-Webhook-Id: <delivery_id>
X-Gigabrain-Feed: alpha
X-Gigabrain-Test: 1   (only on /test endpoint deliveries)
```

## 8. Security & DDoS hardening

The previous webhook module was DDoS'd while credit deduction was active. The most likely vectors and our mitigations:

### 8.1 Mass account creation with free signup credits

**Hypothesis:** attacker registers many accounts, each with $X free credit, and uses them to flood deliveries (or exhaust our queue/credit-pool). This is the most plausible vector for the original DDoS given the timeline (deduction was removed *before* the module was — implying the deduction wasn't gating the attack).

**Mitigations:**
- Webhook creation **requires** the user to have made at least one purchase from a paid source (`credit_ledger` entry with `type='purchase'` AND `source ∈ {coinbase,stripe,moonpay}`). Grants, referrals, and signup credits don't qualify. This applies at:
  - **Registration** (`POST /v1/webhooks`) — reject with 402 if no qualifying ledger entry exists.
  - **Re-activation** (`PATCH /v1/webhooks/:id` with `isActive=true` after suspension) — same check.
  - Once the user has made any qualifying purchase, future re-activations don't re-check (we don't punish lapsed customers).
- Minimum balance to activate: ≥ $5.00 (10 deliveries' worth). Suspended endpoints can be re-activated only with ≥ $5.00 balance.
- Hard cap of **5 active webhooks per user, total** (across both feeds combined). A user can create 5 endpoints all on `alpha`, all on `radar`, or any mix. Per-event fan-out cost: 5 × $0.50 = $2.50 max per event.
- If `User.creditBalance < $0.50` (one delivery), the dispatcher's `deductBalance` returns `insufficient` and the endpoint flips to `suspension_reason='balance'`. The user keeps their config; topping up + re-activating brings it back.

### 8.2 Secret storage

Old code stored `secretKey` in plaintext. v1 keeps plaintext but adds:
- `secret_key` column never returned in any API response except `POST /v1/webhooks` (creation) and `POST /v1/webhooks/:id/rotate-secret`.
- Audit log entry on rotation.
- Future hardening (out of scope): KMS-backed column encryption — flagged in `docs/superpowers/plans/`.

### 8.3 SSRF (with DNS-rebinding protection)

URL validation rejects private/loopback/link-local/metadata IPs **at registration time** AND **at delivery time**.

The delivery-time check is enforced via a custom `https.Agent` whose `lookup` function intercepts every name resolution and refuses if the resolved IP is in any blocked range. Crucially, the resolved IP is **pinned for the connection** — `axios` connects to that IP and sends the original `Host` header. This blocks classic DNS-rebinding TOCTOU where an attacker returns a public IP at validation, then a private IP at the actual TCP connect.

```ts
import { Agent } from 'https';
import { lookup } from 'dns';

export const safeWebhookAgent = new Agent({
  lookup: (hostname, options, callback) => {
    lookup(hostname, options, (err, address, family) => {
      if (err) return callback(err);
      if (isPrivateOrMetadata(address)) {
        return callback(new Error(`SSRF blocked: ${hostname} resolved to ${address}`));
      }
      callback(null, address, family);
    });
  },
});

// Use in axios: { httpsAgent: safeWebhookAgent }
```

Blocked ranges:
- IPv4: `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `127.0.0.0/8`, `169.254.0.0/16` (incl. `169.254.169.254` AWS IMDS), `100.64.0.0/10` (CGNAT), `0.0.0.0/8`.
- IPv6: `::1`, `fc00::/7`, `fe80::/10`, `::ffff:0:0/96` (IPv4-mapped — common rebinding bypass).

Use `is-private-ip` or `ip-cidr` npm package — do not hand-roll. Verify the chosen package handles `::ffff:0:0/96` by default (test with `::ffff:127.0.0.1` → must reject).

### 8.4 HMAC signature

Header: `X-Gigabrain-Signature: t=<unix_seconds>,v1=<hex_hmac>`.

**Canonical form** (must match exactly between sender and receiver):

```
signed_payload = <timestamp_string> + "." + <raw_request_body>
hmac_hex       = hex(HMAC_SHA256(secret_bytes, signed_payload_bytes))
```

Where:
- `timestamp_string` is the Unix-epoch seconds rendered as a base-10 ASCII integer with no leading zeros (e.g. `"1714137296"`).
- `.` is a literal ASCII period (0x2E).
- `raw_request_body` is the unmodified UTF-8 body bytes that were sent over the wire — **no re-serialization**.
- `secret_bytes` is the secret as UTF-8 bytes (the secret itself is hex from `crypto.randomBytes(32).toString('hex')`).

Receivers MUST:
1. Reject if `|now - t| > 300` seconds (5-minute skew).
2. Constant-time-compare `v1` to the expected HMAC.

Verbatim implementation lives in `WebhookSigningService`, restored from `beed758^`.

### 8.5 Rate limit on dispatcher

A separate per-webhook delivery cap (1000 deliveries/hour default, configurable) prevents a misconfigured user URL from causing local queue starvation. If a webhook hits its cap, future deliveries that hour **skip silently** — no Postgres `webhook_deliveries` row is written, no charge is applied. A counter metric `webhook.rate_limited{webhookId}` increments so users can see this in their dashboard.

### 8.6 Bull queue isolation

Webhook delivery jobs run in their own Bull queue (`webhook-delivery`) on a dedicated Redis DB. Even if the queue is overwhelmed, it can't affect chat, agent, or other queues.

## 9. Frontend integration

New dashboard page: `Settings → Developer → Webhooks` ([gigabrain-frontend/src/app/(dashboard)/settings/webhooks/](gigabrain-frontend/src/app/(dashboard)/settings/webhooks/)).

### Page: Webhook list (`/settings/webhooks`)

- Table: name, feed (badge: "Radar" / "Alpha"), URL, status (Active / Suspended-balance / Suspended-failures / Paused), success rate, last delivery, actions (View, Edit, Test, Delete).
- "+ New webhook" CTA top right.
- Empty state with explainer: "Webhooks deliver real-time Radar insights and Alpha signals to your endpoint. $0.50 per event."

### Modal: Create webhook

1. Fields: URL, name, feed (radio: Radar / Alpha — explain each).
2. On submit → show **secret key reveal screen** ("Copy this now — you won't see it again. We use it to sign every payload so you can verify authenticity."). Copy button + "Got it, hide" dismiss.

### Page: Webhook detail (`/settings/webhooks/[id]`)

- Header: name, status badge, edit / test / delete actions, rotate secret.
- Stats: total deliveries, success rate, last delivery, consecutive failures.
- Delivery log (paginated, last 30 days): timestamp, event ID, status, HTTP code, response time, expand for payload + error.

### Sidebar nav

No new top-level item; lives under existing **Settings → Developer** (alongside API keys).

## 10. Observability

- **Metrics** (statsd / Prometheus, whatever the rest of the backend uses):
  - `webhook.dispatch.count{feed}`
  - `webhook.dispatch.duration_ms{feed}`
  - `webhook.delivery.success{feed}`
  - `webhook.delivery.failure{feed,reason}`
  - `webhook.suspension{reason}`
  - `webhook.charge.usd_total{feed}` — revenue counter.
- **Logs:** structured JSON, including `userId`, `webhookId`, `eventId`, `feed`, `latencyMs`, `httpStatus`, `error`.
- **Alerts:** all alert thresholds are computed from `webhook_deliveries` in Postgres, not Bull's truncated history.
  - Page on consumer-loop crash (no `xreadgroup` calls in 60s).
  - Page on Bull `webhook-delivery` queue size > 10,000.
  - Page if `webhook_deliveries` rows with `status='pending' AND created_at < NOW() - 10 minutes` is non-zero (sweeper not keeping up).
  - Slack on `webhook.delivery.failure` rate > 50% over 15-min window (likely a downstream user URL outage but worth knowing).
  - Slack on suspensions/hour > 10 (could indicate widespread issue or attack).

## 11. Migration & rollout

1. **Phase 1** — backend: migration for tables, restored `webhooks` module, stream consumer, Bull processor, sweeper, XAUTOCLAIM cron. Behind a feature flag `WEBHOOKS_PUBLIC_ENABLED` (default off). **Prerequisite work item:** extend `creditService.deductBalance` to optionally accept an external transaction handle (`tx`); audit existing call sites to confirm backwards compatibility (the new param is optional). This change touches the canonical credit ledger path and must be reviewed by someone who knows that code.
2. **Phase 2** — frontend dashboard pages shipped, flag remains off in production. Pages render but the GraphQL queries return empty/disabled state when flag is off.
3. **Phase 3** — dogfood: enable for internal team accounts only (allowlist via env). Confirm metrics + delivery via test endpoints.
4. **Phase 4** — **gated on completing the original-DDoS postmortem** (§12). Once postmortem confirms the §8 hardening covers the original attack class (or identifies new mitigations), enable for all paid users. Monitor for 24h. Watch DDoS detection metrics (creation rate, failed deductions, suspension rate).
5. **Phase 5** — public announcement and Mintlify docs update.

Each phase = its own PR. Order: backend (1 PR) → frontend (1 PR) → docs/Mintlify (1 PR).

### Kill switch / rollback runbook

If a problem surfaces post-rollout:

1. Set `WEBHOOKS_PUBLIC_ENABLED=false` and redeploy. New deliveries stop being enqueued (consumer skips dispatch); existing Bull jobs continue draining.
2. To halt in-flight deliveries: pause the Bull queue via BullBoard admin UI (`webhook-delivery` queue → Pause). Consumer keeps reading streams but enqueues are blocked.
3. To halt stream consumption entirely: stop the `WebhookStreamConsumer` instance. Pending Redis stream entries remain in the consumer group's pending list and resume on next start.

This runbook is encoded in `docs/runbooks/webhooks-rollback.md` (created in Phase 1).

## 12. Open questions / known unknowns

- **Original DDoS root cause is not definitively known.** §8.1 hypothesis (mass account creation w/ free signup credits) is the best fit given the timeline (deduction was removed *before* the module was; suggests deduction wasn't actually gating the attack). **Phase 4 of rollout is gated on completing this postmortem** — grep historical logs, Slack incident channels, CloudWatch/Datadog from the Feb 6–15 2026 window for the actual incident details. If postmortem reveals a different attack class than §8.1 anticipates, expand §8 mitigations before enabling.
- **KMS column encryption for `secret_key`** — deferred. v1 stores plaintext (matches old code). Risk: DB compromise leaks all webhook signing secrets. Tracked as follow-up.
- **Webhook templating** — daemon `WebhookService` (PR #184) supports filter matching (`gte`, `any-of`, etc.). Public webhooks v1 ships **no filtering** — every event matching feed gets delivered. Filters can be a v2 add (charged extra?).

## 13. Reuse from `beed758^`

Files to restore verbatim or near-verbatim:

| Old file | New location | Changes |
|---|---|---|
| `src/webhooks/services/webhook-signing.service.ts` | same | none — copy as-is |
| `src/webhooks/dto/webhook.dto.ts` | same | add `feed` to CreateDto |
| `src/webhooks/webhook.controller.ts` | same | add `:id/test`, `:id/deliveries`, `:id/rotate-secret` |
| `src/webhooks/webhook.resolver.ts` | same | add new mutations & queries |
| `src/webhooks/services/webhook.service.ts` | same | add `feed` validation, SSRF check, `suspend()`, `findActiveByFeed()` |
| `src/webhooks/entities/webhook-endpoint.entity.ts` | same | add `feed`, `suspensionReason`, `consecutiveFailures` |
| `src/webhooks/entities/webhook-delivery.entity.ts` | same | add `feed`, `eventId`, unique index |
| `src/webhooks/processors/webhook-delivery.processor.ts` | same | drop `chargeUser` (charging is now at enqueue, in dispatcher service); old code already injects HMAC `X-GigaBrain-Signature` header — KEEP that (rebrand to `X-Gigabrain-Signature` with lowercase `b`); add `X-Gigabrain-Feed`, `X-Gigabrain-Test` headers; add retry classifier (failure_kind population); use `safeWebhookAgent` for SSRF-protected outbound. |
| `src/webhooks/webhooks.module.ts` | same | + `WebhookStreamConsumer` provider, ApolloFederation registration |

New files:

- `src/webhooks/services/webhook-stream-consumer.service.ts` — NestJS Redis stream consumer.
- `src/webhooks/services/webhook-dispatcher.service.ts` — orchestrates idempotency-row → charge → enqueue.
- `src/webhooks/services/webhook-delivery.service.ts` — already existed; keep + extend.
- `src/webhooks/constants.ts` — `FEED_TO_STREAM` map.
- `src/webhooks/payload-builders.ts` — feed-specific payload shaping.
- `src/migrations/<timestamp>-CreateWebhookTables.ts` — schema migration.

## 14. Tests

- **Unit:** signing service (incl. canonical form §8.4), payload builders, URL validator (incl. SSRF cases), retry classifier, suspension state machine, paid-source ledger predicate.
- **Integration:**
  - Create webhook → publish event to test stream → verify delivery row + Bull job + HTTP POST received by mock server with valid HMAC.
  - DNS-rebinding attack: validation passes, agent's `lookup` rejects at delivery — verify no HTTP request was made.
- **Charge correctness:**
  - Balance below threshold → suspension + no delivery row created (insufficient case).
  - Balance succeeds → ledger debit recorded atomically with delivery row insert.
  - Enqueue failure path → refund recorded in ledger AND `refunded_amount` set on delivery.
- **Idempotency:** replay same event ID for same endpoint → second call short-circuits in txn, no double charge, no second delivery row.
- **Crash safety:** simulate process kill between txn commit and `queue.add` (e.g., kill the connection mid-call) → assert pending delivery exists with charge → run sweeper → assert refund issued and delivery marked failed with `failure_kind='enqueue'`.
- **XAUTOCLAIM resilience:** consumer A starts processing entry, dies before xack. Consumer B (or sweeper-restart) reclaims and processes. Idempotency row blocks duplicate. Net result: exactly one delivery, exactly one charge.
- **Concurrency:** two replicas reading the same stream → each event delivered exactly once across the consumer group (Redis consumer-group semantics, plus our idempotency belt-and-suspenders).
- **Alerting source:** assertions read from Postgres `webhook_deliveries`, not Bull, since `removeOnFail: 500` truncates Bull's history.

## 15. Repos changed

| Repo | Change |
|---|---|
| `gigabrain-backend` | New `webhooks` module + migration + frontend GraphQL schema. Bulk of work. |
| `gigabrain-frontend` | Settings → Developer → Webhooks pages. |
| `intelligence-monorepo` | **No changes.** Streams already publish (PR #184). |
| `docs` (Mintlify) | New Webhooks doc page (deferred to phase 5). |
