---
title: "Killing KV in the hot path"
subtitle: "How I cut Cloudflare KV from 6-9 operations per request to zero, and what I replaced it with."
section: dev
type: case-study
project: TrackerSync
date: 2026-03-04
tags:
  - publish/dev
  - cloudflare
  - workers
  - d1
  - durable-objects
  - rate-limiting
  - case-study
status: prototype
---

> [!summary]
> TrackerSync is a free, serverless Fitbit → Garmin migration tool I run on Cloudflare's free tier. In February 2026, every normal API request was spending **6 to 9 KV operations** before doing any actual work. At even modest traffic that meant burning through the free-tier KV ceiling in hours. This is the story of moving rate-limiting to a Durable Object, metadata to D1, and shrinking KV's role to "graceful degradation only." Result: **zero KV ops on the happy path.**

## The shape of the problem

TrackerSync's hot path looks deceptively simple:

```
client → /api/upload → /api/validate → /api/convert → /api/download
```

Underneath, each of those endpoints was hitting Cloudflare KV multiple times before it even touched the user's file:

| Endpoint | KV ops/request (before) | KV ops/request (after) |
| --- | ---: | ---: |
| `/api/upload` | 6 – 9 | 0 |
| `/api/validate` | 6 – 9 | 0 |
| `/api/convert` | 7 – 10 | 0 |
| `/api/download/*` | 6 – 9 | 0 |
| `/api/usage/*` | 3 | 0 |

The 6-9 wasn't one careless KV call repeated; it was four legitimate concerns layered on top of each other:

1. **Suspicious-client check** in global middleware. KV read.
2. **Blocked-client check.** KV read.
3. **Multi-tier rate-limiter.** Up to 3 KV reads (per-IP, per-fingerprint, per-cookie).
4. **Health probe churn.** A background "is KV alive?" probe that wrote, read, and deleted a sentinel key — running often enough to dominate the budget at low traffic.

Each one made sense on its own. Stacked, they were a free-tier killer.

## Why KV was the wrong tool

KV is wonderful for what it's designed for: globally-replicated, eventually-consistent reads. The cache-style reads are practically free in latency terms. But "rate limit" and "did this IP misbehave?" have requirements KV doesn't meet well:

- **Atomicity.** Two concurrent requests need to agree on whether a counter has been incremented. KV is last-write-wins.
- **Read-your-writes consistency.** A blocked client should be blocked *immediately*, not after KV's propagation window.
- **Pricing model.** KV charges per operation; D1 charges per row-scan. Counters that get hit on every request are cheaper as rows than as KV keys.

Once you say it out loud — *"I need atomic counters with read-your-writes consistency"* — the answer is obvious: **Durable Objects**, with **D1** as the durable record.

## The new architecture

<svg viewBox="0 0 720 360" xmlns="http://www.w3.org/2000/svg" role="img" aria-label="Before-and-after architecture comparison: KV-heavy hot path versus D1+DO hot path">
  <style>
    .label { font: 600 13px 'Helvetica Neue', Helvetica, Arial, sans-serif; fill: #111; }
    .sub   { font: 11px 'Helvetica Neue', Helvetica, Arial, sans-serif; fill: #555; }
    .box   { fill: #fff; stroke: #111; stroke-width: 1.5; }
    .dim   { fill: #f3f3f3; stroke: #aaa; stroke-width: 1; }
    .kv    { fill: #fff3cd; stroke: #b58900; stroke-width: 1.5; }
    .d1    { fill: #e6f4ea; stroke: #2c7a3f; stroke-width: 1.5; }
    .do    { fill: #e3edff; stroke: #1f4eb0; stroke-width: 1.5; }
    .arrow { stroke: #111; stroke-width: 1.5; fill: none; marker-end: url(#a); }
    .arrow-d{ stroke: #aaa; stroke-width: 1.5; fill: none; stroke-dasharray: 4 3; marker-end: url(#ad); }
    .title { font: 700 14px 'Helvetica Neue', Helvetica, Arial, sans-serif; fill: #111; }
  </style>
  <defs>
    <marker id="a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#111"/></marker>
    <marker id="ad" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse"><path d="M0,0 L10,5 L0,10 z" fill="#aaa"/></marker>
  </defs>

  <!-- BEFORE -->
  <text x="20" y="24" class="title">Before — every request touches KV 6-9 times</text>
  <rect x="20"  y="40" width="100" height="44" class="box"/><text x="70" y="66" text-anchor="middle" class="label">Client</text>
  <rect x="160" y="40" width="120" height="44" class="box"/><text x="220" y="60" text-anchor="middle" class="label">Worker</text><text x="220" y="76" text-anchor="middle" class="sub">middleware + handler</text>
  <rect x="320" y="20" width="140" height="36" class="kv"/><text x="390" y="42" text-anchor="middle" class="label">KV: suspicious</text>
  <rect x="320" y="62" width="140" height="36" class="kv"/><text x="390" y="84" text-anchor="middle" class="label">KV: blocked</text>
  <rect x="320" y="104" width="140" height="36" class="kv"/><text x="390" y="126" text-anchor="middle" class="label">KV: ratelimit ×3</text>
  <rect x="320" y="146" width="140" height="36" class="kv"/><text x="390" y="168" text-anchor="middle" class="label">KV: health probe</text>
  <rect x="500" y="60" width="100" height="44" class="box"/><text x="550" y="86" text-anchor="middle" class="label">Job</text>
  <path d="M120,62 H160" class="arrow"/>
  <path d="M280,55 H320" class="arrow"/>
  <path d="M280,68 H320" class="arrow"/>
  <path d="M280,82 H320" class="arrow"/>
  <path d="M280,95 H320" class="arrow"/>
  <path d="M460,82 H500" class="arrow"/>

  <!-- AFTER -->
  <text x="20" y="220" class="title">After — D1 + DO on the hot path, KV only for degraded mode</text>
  <rect x="20"  y="236" width="100" height="44" class="box"/><text x="70" y="262" text-anchor="middle" class="label">Client</text>
  <rect x="160" y="236" width="120" height="44" class="box"/><text x="220" y="256" text-anchor="middle" class="label">Worker</text><text x="220" y="272" text-anchor="middle" class="sub">slim middleware</text>
  <rect x="320" y="220" width="140" height="36" class="do"/><text x="390" y="242" text-anchor="middle" class="label">DO: AtomicLimiter</text>
  <rect x="320" y="262" width="140" height="36" class="d1"/><text x="390" y="284" text-anchor="middle" class="label">D1: metadata + counters</text>
  <rect x="320" y="304" width="140" height="36" class="dim"/><text x="390" y="326" text-anchor="middle" class="label">KV: fallback only</text>
  <rect x="500" y="256" width="100" height="44" class="box"/><text x="550" y="282" text-anchor="middle" class="label">Job</text>
  <path d="M120,258 H160" class="arrow"/>
  <path d="M280,250 H320" class="arrow"/>
  <path d="M280,278 H320" class="arrow"/>
  <path d="M280,322 H320" class="arrow-d"/>
  <path d="M460,278 H500" class="arrow"/>
</svg>

Three swaps did the work:

### 1. Rate-limiting moves into a Durable Object

`AtomicRateLimiter` is a Durable Object keyed by client signal (fingerprint + IP-hash + cookie blend). Inside the DO, increments are serialized — no race conditions, no double-spends. The DO's `storage` API is read-after-write consistent by construction, so a newly-blocked client stays blocked on the next request even if it lands in a different isolate.

The middleware became three lines:

```js
const limiter = env.RATE_LIMIT_DO.idForName(signal);
const stub = env.RATE_LIMIT_DO.get(limiter);
const decision = await stub.fetch(request);
```

### 2. Transient metadata moves from KV to D1

Uploads and conversions both leave a metadata breadcrumb that downstream endpoints need to read. These had been KV entries with a TTL. They're now D1 rows:

```sql
CREATE TABLE uploads (
  upload_id   TEXT PRIMARY KEY,
  metadata    TEXT,
  created_at  INTEGER,
  expires_at  INTEGER
);
CREATE INDEX idx_uploads_expires_at ON uploads(expires_at);
```

Expired rows are treated as missing on read and lazily deleted during maintenance. The TTLs (3600s for uploads, 7200s for conversions) didn't change; only the backing store did.

### 3. Health probes leave the hot path

The "is KV alive?" check used to run on every request as a write-read-delete cycle on a sentinel key. That was three KV ops *per request* just to know whether KV was sad. The new arrangement:

- **Hot path:** assume KV is fine; rely on D1+DO for correctness.
- **Cold path:** a single read-only `get()` of a known sentinel, run only when status endpoints or maintenance jobs ask.
- **Degraded mode:** if D1 fails, fall back to KV-based limiting. KV is the lifeboat, not the engine.

## The shape of the result

The first 24 hours after deployment told a clear story:

```text
KV ops/day
┌────────────────────────────────────────────────────┐
│ before  ████████████████████████████████ ~310,000 │
│ after   ▏           <500 (fallback + status pings) │
└────────────────────────────────────────────────────┘
                                       (~99.8% reduction)
```

D1 query volume went up, but D1 row reads on the free tier have a much larger budget than KV ops, and most reads hit the `uploads` / `conversions` indexes. The new bottleneck — if there is one — is D1 row-reads, which I now have a clean signal on.

The DO didn't blow up. That was the part I was most nervous about: Durable Objects serialize requests by key, so a misconfigured signal blend could turn a single noisy client into a global queue. The blend (`fingerprint_hash ⊕ ip_subnet`) keeps cardinality high enough that no single DO instance carries a meaningful slice of traffic.

## What this cost me

Three things to be honest about.

**Migration risk.** Moving the rate-limit decision to a new storage backend is the kind of change that, done badly, lets every request through for fifteen minutes. I shipped it behind a flag with KV as a fallback for the first week, watched the metrics, and only removed the flag once D1 + DO had carried a full traffic cycle without complaint.

**A new failure mode.** The DO is now on the critical path. If a Durable Object namespace has trouble, every request slows down or fails. The intelligent-fallback path covers it, but the *latency* of falling back is a real thing I now have to alert on.

**Per-tenant noise.** D1 is single-region (closest replica reads, single writer). At low scale this is fine; at higher scale I'll need to think about read replicas or sharding. That's a future problem, not today's.

## The takeaway

KV is a *great* primitive for the workloads it was designed for: globally-cached configuration, session-shaped reads, read-heavy lookups where eventual consistency is fine. The mistake I made — and I think it's a common one — was reaching for KV because it was the first key-value store at hand, not because it fit.

Rate limiting wants *atomicity*. Metadata wants *transactional reads with a TTL*. Both have better homes on Cloudflare's stack than KV. The hot path is cleaner, the free-tier headroom is back, and — quieter benefit — the worker is now easier to reason about, because each storage layer has one job.

---

*Project: [TrackerSync](https://trackersync.app) — free Fitbit → Garmin migration on Cloudflare's free tier.*
