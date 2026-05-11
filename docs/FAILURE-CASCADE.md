# Swiggy Scale Failure Cascade Analysis 💀

## Scenario Context

**Event:** India vs Pakistan World Cup Final, 8:05 PM IST  
**Promotion:** 50% off on all orders sent to 180 million users  
**Expected click-through rate:** ~8% = 14.4 million users opening the app  
**Concentrated spike window:** ~60 seconds from notification delivery  
**Business impact:** ₹4.2 crore in lost orders per minute during outage  

---

## Section 1: Traffic Simulation Math

### Peak RPS Calculation

```
Total notification recipients:        180,000,000
Click-through rate on major promos:   ~8%
Users who open the app:                14,400,000

Concentrated spike window:             ~60 seconds
Each user makes ~3 API calls in first 60s:
  - GET /restaurants (menu listing)    → ~10ms DB query
  - GET /restaurant/:id/details        → ~15ms DB query  
  - POST /orders                        → ~50ms (includes sync payment)

Peak RPS Calculation:
  10,000,000 users × 3 calls / 60 seconds = 500,000 RPS
```

**Context:** Our monolith handles ~12,000 RPS maximum. We are asking it to handle **41× its capacity**.

### Historical Comparison

| System | Peak RPS | Notes |
|--------|----------|-------|
| **Our Monolith (Capacity)** | ~12,000 RPS | 1 Node process, event loop saturation |
| **Our Scenario (Demand)** | ~500,000 RPS | 41x over capacity |
| Twitter at peak | ~300,000 RPS | 2016 metrics |
| Netflix global | ~800,000 RPS | Peak streaming traffic |
| Google Search | ~8,500,000 RPS | Google infrastructure |

---

## Section 2: Component Capacity Numbers

### The Monolith Hardware

```
Node.js Server:
  ├─ Instance type: t3.medium (2 vCPU, 4GB RAM)
  ├─ Process count: 1 (single process, no clustering)
  ├─ Event loop saturation: ~12,000–15,000 RPS
  ├─ Heap limit: 4GB → OOM at ~15,000 queued requests
  └─ Network: 1Gbps (will saturate under image load)

PostgreSQL Database:
  ├─ max_connections: 100 (hardcoded limit)
  ├─ Connection hold time (menu query): 10–20ms
  ├─ Connection hold time (payment query): 200–2,000ms ⚠️ AMPLIFIER
  ├─ Average query time: 20ms (non-payment)
  └─ No indexes on orders.user_id (O(n) scans)
```

### Database Pool Exhaustion Math

**Formula:** Connections held at any moment = 
```
(% regular RPS × query_hold_time_s) + (% payment RPS × payment_hold_time_s)
```

**Scenario Breakdown:**
- 70% of requests: menu queries (20ms hold) → `0.70 × RPS × 0.020`
- 30% of requests: payment calls (800ms avg hold) → `0.30 × RPS × 0.800`

**Calculation:**
```
Connections at any moment = (0.70 × RPS × 0.020) + (0.30 × RPS × 0.800)
                          = 0.014 × RPS + 0.240 × RPS
                          = 0.254 × RPS

Pool exhausted when: 0.254 × RPS = 100
                     RPS = 394

CRITICAL: With just 30% payment calls, the DB pool exhausts at 394 RPS.
```

At 500,000 RPS incoming, the pool saturates in **3 seconds** with 100% certainty.

### Node.js Event Loop Saturation

When DB connections are exhausted, requests queue in the event loop:
- Request processing time: 50ms → 500ms → 5,000ms
- At ~15,000 pending requests: Node process hits heap limit
- Crash: `FATAL ERROR: CALL_AND_RETRY_LAST Allocation failed - JavaScript heap out of memory`

---

## Section 3: The Full Failure Cascade

### ⚠️ Failure 1: PostgreSQL Connection Pool Exhaustion
**Severity:** CRITICAL  
**Triggers at:** ~394 RPS (within 3 seconds of spike)  
**Signal to users:** Connection timeout errors, 500 errors  
**What it causes next:** All database queries fail → entire application unreachable

```
Timeline:
T+0s: 500,000 RPS incoming requests
T+1s: 100 PostgreSQL connections filled with payment calls (800ms hold)
T+2s: Queue of 50,000 waiting requests in Node.js event loop
T+3s: New connection attempts fail entirely - database rejects ALL new connections
      Users see: "Service Unavailable - Database Connection Error"
```

**Root Math:** At 394 RPS with mixed queries, all 100 slots are consumed with no available connections for new requests.

---

### ⚠️ Failure 2: Synchronous Payment Call Amplification
**Severity:** CRITICAL (amplifier for Failure 1)  
**Triggers at:** Concurrent with Failure 1  
**Signal to users:** Timeouts on order submission  
**What it causes next:** DB connection pool exhaustion (multiplied by 20–200×)

```
Impact Chain:
POST /orders endpoint:
  1. Reserve DB connection
  2. BEGIN TRANSACTION
  3. INSERT into orders table (10ms)
  4. SYNCHRONOUSLY call Razorpay (200–2,000ms) ← HOLDS CONNECTION ENTIRE TIME
  5. COMMIT transaction
  6. Release connection

Hold time: 50ms min → 2,050ms max (41× longer than query alone)

Effect: At 500 RPS with 30% payment calls:
  - 150 payment requests in-flight
  - Each holds 1 connection for 1,000ms avg
  - 150 connections permanently held
  - Pool is exhausted (100 < 150)
  - No room for menu queries (GET /restaurants)
```

Payment calls act as a **force multiplier** on pool exhaustion.

---

### ⚠️ Failure 3: Node.js Event Loop Saturation
**Severity:** CRITICAL  
**Triggers at:** ~12,000 RPS (within 5 seconds)  
**Signal to users:** All requests timeout (60s+), application becomes completely unresponsive  
**What it causes next:** Process crashes with OOM error

```
Saturation Timeline:
T+3s:    DB pool exhausted, requests queue in event loop
T+5s:    15,000 pending requests queued
T+7s:    Node.js garbage collection can't keep up
T+8s:    Heap usage climbs to 3.5GB (87% of 4GB limit)
T+10s:   Memory pressure → More GC pauses (50–100ms each)
T+12s:   Heap hits 4GB limit
         FATAL: JavaScript heap out of memory
         Process exits with code 137

Users see: Connection reset / ERR_EMPTY_RESPONSE
```

---

### ⚠️ Failure 4: Promo Code Race Condition
**Severity:** HIGH  
**Triggers at:** T+10 seconds (concurrent with Failures 1–3)  
**Signal to users:** Over-redemption of promo code, discrepancies in records  
**What it causes next:** Financial loss, negative user sentiment

```
Code Execution (NOT ATOMIC):
1. SELECT remaining_budget FROM promo WHERE id = 123;     -- Query 1: Returns 10,000
2. [CONTEXT SWITCH / 100ms delay due to DB queue]
3. UPDATE promo SET remaining_budget = 9,999 WHERE id = 123;  -- Query 2

Race Condition:
Thread A: SELECT → sees budget 10,000 → [paused by GC]
Thread B: SELECT → sees budget 10,000 → [paused by GC]
Thread A: UPDATE budget to 9,999
Thread B: UPDATE budget to 9,999

Result: Budget should be 9,998, but is 9,999
At 10M users:
  - Expected overdraft: (10M × 500%) - (promo_budget)
  - Actual loss: ~3–5× promo_budget value
```

With concurrent requests in the thousands, the promo is exhausted in **seconds**, not hours.

---

### ⚠️ Failure 5: Static Asset NIC Saturation
**Severity:** CRITICAL  
**Triggers at:** ~T+15 seconds (concurrent with Failure 3)  
**Signal to users:** Images not loading, API calls timeout  
**What it causes next:** Cascading failures - NIC saturation prevents API responses

```
Asset Math:
Image serving from Node.js process:
  - Restaurant images per user session: ~20 images
  - Image size: 200KB avg (compressed, still large)
  - Users in first minute: 10M
  - Total transfer: 10M users × 20 images × 200KB
                  = 40,000,000,000 KB
                  = 40 TB / minute

Monolith NIC capacity: ~1 Gbps = 125 MB/s = 7.5 GB/minute

Time to saturate NIC: 40 TB / 7.5 GB/min = 5,333 minutes (NOT EXAGGERATED)
                     = Complete saturation after delivering first few MB

Network saturation means:
  - No API responses can be sent (even 200-byte JSON responses)
  - All TCP connections timeout
  - Users see: "Network timeout" → retry → timeout again
  - API completely unreachable behind the image transfer
```

The system is now **completely inaccessible to all users**.

---

### ⚠️ Failure 6: Monitoring Blindness (The Invisible Killer)
**Severity:** CRITICAL (extends MTTD)  
**Triggers at:** T+5s  
**Signal to SRE:** `500 errors spiking` (generic alert)  
**What it causes next:** MTTD increases from 5 minutes to 45+ minutes

```
What the on-call engineer sees:
- CloudWatch: 5xx error rate 95%
- Datadog: "Latency p99 = 45s" (but no drill-down)
- No distributed tracing: correlation IDs not in logs
- No structured logging: can't search "failed on payment validation"

Questions the engineer CANNOT answer:
1. Is the DB pool exhausted? (no, they can't query pg_stat_activity without DB access)
2. Is payment gateway down? (no metrics for 3rd party latency)
3. Is the NIC saturated? (no network monitoring)
4. Did we deploy something 5 minutes ago? (no event correlation)

Result:
  - First 20 mins: "Restart the server"
  - Next 10 mins: "Scale up compute" (doesn't help - DB is bottleneck)
  - Next 15 mins: Realizes it's DB, calls DBA
  - Total MTTD: 45 minutes
  - Revenue lost: 45 min × ₹4.2 crore/min = ₹189 crore
```

**The cascade could be stopped in 5 minutes with proper monitoring. Without it: 45 minutes.**

---

## Section 4: The Incident Timeline

| Time | Event | Impact |
|------|-------|--------|
| **T+0s** | 🚨 Push notification sent to 180M users | Initialization |
| **T+3s** | PostgreSQL connection pool exhausted (100/100) | DB rejecting new connections |
| **T+5s** | Node.js event loop backing up: response times 50ms → 500ms | Users perceive slowness |
| **T+8s** | All 100 DB connections held by payment calls | Menu queries completely blocked |
| **T+10s** | Promo race condition: budget over-redeemed by 5–10M requests | Financial loss begins |
| **T+12s** | Node.js event loop: 15,000 pending requests queued | OOM approaching |
| **T+15s** | Static assets saturate NIC (40TB/min transfer) | API completely unreachable |
| **T+18s** | Node.js process crashes: `FATAL: heap out of memory` | Service DOWN |
| **T+20s** | Load balancer marks instance unhealthy (no recovery) | No failover available |
| **T+22s** | Automated alerts fire: `5xx error rate > 99%` | MTTD clock starts |
| **T+5min** | On-call engineer context-switched from sleep | SRE begins diagnosis |
| **T+15min** | Engineer tries to SSH into database, but overwhelmed query queue blocks queries | Can't diagnose - DB is blocked |
| **T+25min** | Engineer manually restarts Node.js process | Failed: stack resumes at T+18s state (crash loop) |
| **T+35min** | Engineer realizes need to flush stale connections: `pg_terminate_backend()` | DB recovers |
| **T+40min** | Node.js restarts successfully, begins recovery | Slow traffic recovery |
| **T+45min** | System stable, all users can access platform | Incident closed |
| **T+60min** | Post-incident triage begins | Analysis starts |

**Total Impact:**
- **Downtime:** 27 seconds to complete failure, 45 minutes to full recovery
- **Users affected:** 10M users attempted orders
- **Revenue lost:** 45 minutes × ₹4.2 crore/minute = **₹189 crore**
- **App Store rating:** 4.3 → 1.8 in 45 minutes (600+ one-star reviews)
- **Customers lost:** ~5–8% churn in following weeks (based on 2018 Hotstar incident)

---

## Key Insight

**The cascade is not random. It follows physics.**

Each component has a hard limit. When that limit is hit, it fails in a specific, predictable way that overloads the next component. The entire failure from notification to OOM is **determined the moment the 500K RPS starts flowing**.

The monolith cannot survive this scenario. The only question is which component fails first (it's PostgreSQL at 394 RPS). Every other failure follows as physics, not chance.
