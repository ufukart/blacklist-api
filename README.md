# 🚫 Blacklist API

> **IP Reputation Lookup Service** — Check whether any IP address appears on the blacklist with a single GET request. No authentication required.

[![Status](https://img.shields.io/badge/status-live-brightgreen)](https://blacklist.zumbo.net/health)
[![Auth](https://img.shields.io/badge/auth-none-blue)](#)
[![Rate Limit](https://img.shields.io/badge/rate%20limit-10%20req%2Fs-orange)](#rate-limiting)
[![IPv4](https://img.shields.io/badge/IPv4-supported-blueviolet)](#)

---

## Overview

Blacklist API provides instant IP reputation lookups against a continuously updated dataset of known malicious addresses. Both exact IP matches and CIDR block containment are evaluated on every request.

| Stat | Value |
|---|---|
| Blacklisted IPs | 234,000+ |
| CIDR Blocks | 5,600+ |
| Updates | Automatic, zero-downtime |
| Auth | None required |

**Base URL:** `https://blacklist.zumbo.net`

---

## Quick Start

```bash
# Check an IP address
curl "https://blacklist.zumbo.net/check?ip=1.2.3.4"

# Response (listed)
{ "listed": true }

# Response (clean)
{ "listed": false }
```

---

## Endpoints

### `GET /check`

Checks a given IP address against the blacklist. Evaluates both exact matches and CIDR block containment. Supports IPv4 and IPv6.

**Query Parameters**

| Parameter | Type | Required | Description |
|---|---|---|---|
| `ip` | string | ✅ | IPv4 or IPv6 address to look up |

**Examples**

```bash
# IPv4
curl "https://blacklist.zumbo.net/check?ip=1.2.3.4"

# IPv6
curl "https://blacklist.zumbo.net/check?ip=2001:db8::1"
```

**Responses**

```jsonc
// 200 — Listed
{ "listed": true }

// 200 — Clean
{ "listed": false }

// 400 — Invalid IP
{ "error": "Geçersiz IP adresi" }
```

---

### `GET /meta`

Returns metadata about the active dataset: total record counts and the timestamp of the last successful update.

```bash
curl "https://blacklist.zumbo.net/meta"
```

```json
{
  "last_updated": "2026-03-22T05:09:40Z",
  "total_parsed_ips": 234002,
  "total_parsed_cidrs": 5641
}
```

---

### `GET /health`

Reports whether the service is ready to handle requests. Returns `503` while the dataset is still loading on cold start. Suitable for Kubernetes readiness probes and load balancer health checks.

```bash
curl "https://blacklist.zumbo.net/health"
```

```jsonc
// 200 — Ready
{ "status": "healthy" }

// 503 — Dataset loading
{ "status": "loading" }
```

---

## Rate Limiting

Requests are throttled per source IP using a **token bucket** algorithm.

| Rule | Value | Description |
|---|---|---|
| Rate | `10 req/s` | Sustained throughput (token refill rate) |
| Burst | `30 req` | Maximum instantaneous capacity |
| TTL | `10 minutes` | Idle IP eviction window |

When the limit is exceeded:

```json
{ "error": "Too Many Requests" }
```

> IPs that have been inactive for more than 10 minutes are automatically evicted from memory.

---

## Error Reference

All error responses are JSON objects with an `error` key.

| Code | Meaning | Description |
|---|---|---|
| `200` | OK | Query succeeded — inspect the `listed` field |
| `400` | Bad Request | `ip` parameter is missing or not a valid IP |
| `405` | Method Not Allowed | Only `GET` requests are accepted |
| `429` | Too Many Requests | Rate limit exceeded — back off and retry |
| `503` | Service Unavailable | Dataset not yet loaded (cold start) — retry shortly |

---

## Usage Examples

**JavaScript (fetch)**
```js
const res = await fetch("https://blacklist.zumbo.net/check?ip=1.2.3.4");
const { listed } = await res.json();

if (listed) {
  console.log("⚠️ IP is blacklisted");
} else {
  console.log("✅ IP is clean");
}
```

**Python (requests)**
```python
import requests

res = requests.get("https://blacklist.zumbo.net/check", params={"ip": "1.2.3.4"})
data = res.json()

if data["listed"]:
    print("⚠️ IP is blacklisted")
else:
    print("✅ IP is clean")
```

**Go**
```go
resp, err := http.Get("https://blacklist.zumbo.net/check?ip=1.2.3.4")
if err != nil {
    log.Fatal(err)
}
defer resp.Body.Close()

var result struct {
    Listed bool `json:"listed"`
}
json.NewDecoder(resp.Body).Decode(&result)
fmt.Println(result.Listed)
```

---

## How It Works

- **Dataset** is fetched and parsed on startup, then refreshed automatically in the background.
- **Zero-downtime updates** — the dataset is swapped atomically via `atomic.Pointer`. In-flight requests always see a complete, consistent snapshot; the old dataset is never torn down mid-request.
- **CIDR matching** — every query checks both exact IP matches and whether the IP falls within any blacklisted network range.
- **Memory efficiency** — idle client IPs are evicted from the rate limiter after 10 minutes of inactivity.

---

## License

MIT

---

> **Live API:** [blacklist.zumbo.net](https://blacklist.zumbo.net)
