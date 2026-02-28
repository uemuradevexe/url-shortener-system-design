# Design Doc: URL Shortener

**Version:** 1.0  
**Author:** Ricardo  

---

## 1. Overview

The service receives a long URL and returns a short URL that redirects to the original destination. The v1 focus is operational simplicity and low redirect latency, with an architecture that scales horizontally without structural changes. The application layer is fully stateless — all state lives in Redis and PostgreSQL.

---

## 2. API Design

### `POST /shorten`

Creates a short URL from a long URL.

**Request:**
```json
{
  "long_url": "https://example.com/very-long-article?utm=123",
  "custom_code": "my-link",          // optional
  "expires_at": "2026-12-31T23:59:59Z"  // optional
}
```

**Response `201 Created`:**
```json
{
  "short_url": "https://srt.ly/abc1234",
  "code": "abc1234",
  "expires_at": "2026-12-31T23:59:59Z"
}
```

**Errors:**
| Status | Situation |
|--------|-----------|
| `400`  | Invalid or malformed URL |
| `409`  | `custom_code` already in use |
| `422`  | Disallowed URL scheme (e.g. `ftp://`) |

---

### `GET /:code`

Redirects to the original URL.

**Response `302 Found`:**
```
Location: https://example.com/very-long-article?utm=123
```

**Errors:**
| Status | Situation |
|--------|-----------|
| `404`  | Code not found |
| `410 Gone` | Link has expired |

---

## 3. Short ID Generation

### Chosen strategy: Atomic counter + Base62

A global counter in Redis is atomically incremented (`INCR`) for every new URL. The resulting integer is encoded in Base62 (characters `a-z A-Z 0-9`).

```
ID: 1,000,000   → Base62 → "FXdiL"   (5 chars)
ID: 3,500,000,000 → Base62 → "7n42DGM" (7 chars)
```

With 7 Base62 characters: **62⁷ ≈ 3.5 trillion possible URLs.**

### Trade-offs of each approach

| Approach | Pros | Cons |
|----------|------|------|
| **Counter + Base62**  | No collisions, simple, short codes | Sequential IDs (predictable) |
| Hash (MD5/SHA256 truncated) | Deterministic for same URL | Collisions possible, extra verification needed |
| UUID | Globally unique | Too long for a short URL |
| Nanoid | Random, configurable length | Collisions possible, more complexity |
| Pre-generated ID pool | No latency at creation time | Operational overhead, storage cost |

### Handling collisions

With Counter + Base62, **collisions are impossible** — each increment produces a unique number. The only requirement is that the Redis counter is persisted (`AOF` or `RDB`) so it survives restarts.

For user-defined `custom_code`, uniqueness is enforced via a `UNIQUE` constraint in the database before writing.

---

## 4. Storage

### Source of truth: PostgreSQL

```sql
CREATE TABLE urls (
  id         BIGSERIAL PRIMARY KEY,
  code       VARCHAR(12) UNIQUE NOT NULL,
  long_url   TEXT NOT NULL,
  user_id    BIGINT,                         -- NULL = anonymous
  expires_at TIMESTAMPTZ,                    -- NULL = never expires
  created_at TIMESTAMPTZ DEFAULT NOW()
);

CREATE INDEX idx_urls_code ON urls (code);
```

### Cache layer: Redis

Redis serves two roles:

1. **Atomic counter** for ID generation (`INCR shortener:counter`)
2. **Read cache** for redirects (`GET shortener:url:{code}`)

```
KEY:   shortener:url:abc1234
VALUE: https://example.com/very-long-article
TTL:   aligned with the URL's expires_at (or 24h default for non-expiring URLs)
```

### Why this combination?

Redirects are a **read-heavy workload** with simple key-based access — the ideal Redis use case. PostgreSQL guarantees durability and enables future analytical queries (e.g. "top 10 URLs by user"). Using only Redis would be fragile (volatile data); using only PostgreSQL would be slow (disk latency on the hot path).

---

## 5. Redirect Flow

```
1. User accesses https://srt.ly/abc1234
2. API looks up Redis: GET shortener:url:abc1234
   ├── HIT  → check TTL expiration → 302 redirect
   └── MISS → query PostgreSQL
               ├── Not found  → 404
               ├── Expired    → 410 Gone + DELETE record
               └── Found      → populate Redis → 302 redirect
```

### 301 vs 302

**Choice: `302 Found` (temporary redirect)**

With `301`, the browser permanently caches the redirect and **never hits the server again**. That sounds great for performance, but it permanently eliminates the ability to:

- Count clicks and gather analytics
- Update or deactivate a short URL
- Expire links

In practice, a short link that cannot be deactivated is a security liability — if the destination becomes malicious, there is no way to remediate it. The extra latency of `302` over `301` is absorbed by the Redis cache (< 2ms).

> **Simple rule:** use `301` only if you don't care about analytics and the destination is truly permanent forever. Use `302` for everything else.

---

## 6. Edge Cases

### Same URL submitted twice

Current behavior: **creates two distinct short codes**. No deduplication by `long_url`.

Rationale: deduplication requires an additional index on a `TEXT` column (expensive) and introduces ambiguity — the same destination URL may have different expiration dates or belong to different users. In v1, simplicity wins.

### Invalid URLs

Validation happens at request time, before any write:

- Must use `http://` or `https://` scheme
- Must have a valid hostname (regex + URL parsing)
- Must not point to the shortener itself (prevents redirect loops)
- Maximum length: 2,048 characters

Returns `400` with a descriptive error message.

### Link expiration

For URLs with `expires_at` set:

- Redis TTL is configured at cache write time
- On redirect, if the record exists but `expires_at < NOW()`, returns `410 Gone`
- A daily cleanup job runs `DELETE WHERE expires_at < NOW()`

### Original URL goes offline

The service **does not verify** whether the destination is reachable — this would be prohibitively expensive and would break legitimate use cases (private URLs, intranets). This is the user's responsibility. A v2 optional health check feature could be considered.

---

## 7. Scale

### Scenario: 1 million concurrent requests to the same link

**Problem:** Redis is single-threaded per shard. 1M req/s on a single key can saturate it.

**Layered solution:**

```
                    ┌─────────────────────┐
                    │   Cloudflare CDN    │  ← caches the 302 response at the edge
                    │   Cache-Control:    │    for 60s (configurable per link)
                    │   max-age=60        │
                    └──────────┬──────────┘
                               │ (only cache misses reach here)
                    ┌──────────▼──────────┐
                    │   Load Balancer     │
                    └──────────┬──────────┘
                               │
              ┌────────────────▼────────────────┐
              │      API Servers (N pods)       │  stateless, horizontal scaling
              └────────────────┬────────────────┘
                               │
              ┌────────────────▼────────────────┐
              │       Redis Cluster             │  sharding by key hash
              └────────────────┬────────────────┘
                               │ (only cache misses)
              ┌────────────────▼────────────────┐
              │   PostgreSQL Primary + Replicas │  replicas for analytical reads
              └─────────────────────────────────┘
```

The **CDN is the most impactful layer**: viral links are served directly from the edge, never reaching Redis.

### Capacity estimates

| Load | Bottleneck | Solution |
|------|------------|----------|
| 1,000 req/s | Redis (single instance) | Handles easily out of the box |
| 10,000 req/s | Redis + API pods | Redis Cluster, 3-5 API pods |
| 100,000 req/s | Everything | CDN + Redis Cluster + PG read replicas |
| 1M req/s | CDN | 99%+ of hits resolved at the edge |

### Read replicas on PostgreSQL

Analytical reads (dashboards, reports) must be routed to replicas, never to the primary. The primary only handles writes (`INSERT` of new URLs).

---

## 8. What We Would NOT Build in v1

| Feature | Reason to defer |
|---------|-----------------|
| **Real-time analytics** (clicks, geo, device) | Requires an event pipeline (Kafka + ClickHouse). High complexity, zero user expectation at day 1 |
| **Authentication and user accounts** | Dramatically expands scope. v1 accepts URLs without auth |
| **Custom domains** (`mysite.com/link`) | Requires complex routing, per-tenant SSL certificates, multi-tenancy in the schema |
| **Metrics dashboard** | Depends on analytics, which is already deferred |
| **Per-IP rate limiting** | Important, but can be delegated to the API Gateway or Cloudflare in v1 |
| **Malicious URL scanning** | Requires Safe Browsing API integration — v2, once the threat model is clearer |
| **URL deduplication** | Index cost on a TEXT column with no clear MVP benefit |
| **Pre-generated ID pool** | Operational overhead with no real gain over Counter + Base62 |

> **Principle:** v1 should be deployable by one person in one day. Each feature above deserves its own design doc before being considered.

---

## Stack Summary

| Layer | Technology | Rationale |
|-------|------------|-----------|
| API | Go (or Node.js) | Low latency, strong concurrency for I/O-bound workloads |
| Cache + Counter | Redis | Atomicity, speed, native TTL support |
| Database | PostgreSQL | Durability, UNIQUE constraints, operational familiarity |
| CDN | Cloudflare | Edge caching, DDoS protection, free at reasonable scale |
| Infra | Kubernetes (or Fly.io) | Horizontal scaling of API pods |
