---
layout: default
title: Handling Concurrency in Odeko's Ordering System
---

# Handling Concurrency in Odeko's Ordering System

## Context

Cafe owner customers order supplies through Odeko's **Supply Portal**. Orders placed before a daily cutoff are sent to **NetSuite** (our third-party ERP) for immediate fulfillment — getting supplies to customers overnight. Orders after the cutoff are queued for the next day.

NetSuite handles key business functions across multiple microservices:

- Sales order generation (Orders Service)
- Product information sync (Products Service)
- Customer creation (Customers Service)
- Delivery attachment (Delivery Service)

**Constraint:** NetSuite enforces a hard limit of **10 concurrent HTTP requests** at any time, with per-request latency ranging from **200ms to 3 seconds**.

---

## Problem

As order volume grows, N microservices compete for the same 10 concurrency slots in NetSuite. The existing batch process — triggered after the warehouse order deadline — creates a spike of simultaneous requests that exceeds the ERP's limit. There is no coordination between services, no prioritization of time-sensitive operations, and no protection against cascading failures if NetSuite degrades.

---

## Solution: NetSuite Gateway Service

Introduce a **centralized gateway** that all internal microservices route through. The gateway enforces the concurrency limit via a semaphore-gated worker pool and a priority queue, ensuring critical pre-deadline operations are always processed first.

### Design Principles

- No microservice communicates with NetSuite directly
- A single semaphore (count = 10) is the sole enforcement point for the concurrency limit
- Time-sensitive operations are promoted via explicit priority tiers
- The system is async-first to avoid blocking microservices during high load

---

## Architecture Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          INTERNAL MICROSERVICES                             │
│                                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │
│  │ Orders Svc   │  │ Products Svc │  │ Customers Svc│  │ Delivery Svc │  │
│  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  └──────┬───────┘  │
└─────────┼─────────────────┼─────────────────┼─────────────────┼───────────┘
          │                 │                 │                 │
          │         HTTP POST /netsuite/jobs  │                 │
          └─────────────────┴─────────────────┴─────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                        NETSUITE GATEWAY SERVICE                             │
│                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  REST API Layer                                                      │   │
│  │  POST /jobs  { priority, operation, payload, callback_url }         │   │
│  │  GET  /jobs/:id  (status polling)                                   │   │
│  └───────────────────────────┬─────────────────────────────────────────┘   │
│                              │                                              │
│  ┌───────────────────────────▼─────────────────────────────────────────┐   │
│  │  Priority Classification                                             │   │
│  │                                                                      │   │
│  │  CRITICAL (0)  → Pre-deadline sales order creation                  │   │
│  │  HIGH     (1)  → Post-deadline orders, customer creation            │   │
│  │  MEDIUM   (2)  → Delivery attachment, order updates                 │   │
│  │  LOW      (3)  → Product syncs, bulk reads                          │   │
│  └───────────────────────────┬─────────────────────────────────────────┘   │
│                              │                                              │
│  ┌───────────────────────────▼─────────────────────────────────────────┐   │
│  │  Priority Queue  (Redis Sorted Set)                                  │   │
│  │                                                                      │   │
│  │  score = priority_tier * 10^12 + unix_timestamp_ms                  │   │
│  │                                                                      │   │
│  │  [CRITICAL | t=09:59:50] ← pulled first                             │   │
│  │  [CRITICAL | t=09:59:55]                                             │   │
│  │  [HIGH     | t=10:01:00]                                             │   │
│  │  [MEDIUM   | t=09:58:00]  ← lower priority despite earlier arrival  │   │
│  │  [LOW      | t=09:45:00]                                             │   │
│  └───────────────────────────┬─────────────────────────────────────────┘   │
│                              │                                              │
│  ┌───────────────────────────▼─────────────────────────────────────────┐   │
│  │  Worker Pool + Semaphore (max_concurrent = 10)                       │   │
│  │                                                                      │   │
│  │  Worker 1  [ACTIVE → NetSuite]     Worker 6  [ACTIVE → NetSuite]    │   │
│  │  Worker 2  [ACTIVE → NetSuite]     Worker 7  [ACTIVE → NetSuite]    │   │
│  │  Worker 3  [ACTIVE → NetSuite]     Worker 8  [ACTIVE → NetSuite]    │   │
│  │  Worker 4  [ACTIVE → NetSuite]     Worker 9  [IDLE]                 │   │
│  │  Worker 5  [ACTIVE → NetSuite]     Worker 10 [IDLE]                 │   │
│  │                                                                      │   │
│  │  acquire(semaphore) → dequeue → HTTP call → store result → release  │   │
│  └───────────────────────────┬─────────────────────────────────────────┘   │
│                              │                                              │
│  ┌───────────────────────────▼─────────────────────────────────────────┐   │
│  │  Job Result Store  (Redis Hash)                                      │   │
│  │  job_id → { status, response, attempts, error, completed_at }       │   │
│  └───────────────────────────┬─────────────────────────────────────────┘   │
│                              │                                              │
│  ┌───────────────────────────▼─────────────────────────────────────────┐   │
│  │  Retry + Circuit Breaker                                             │   │
│  │  - Exponential backoff (max 3 attempts for transient errors)        │   │
│  │  - Circuit OPEN if NetSuite error rate exceeds threshold            │   │
│  │  - Dead Letter Queue for exhausted jobs → alert on-call             │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────────────────────┘
                              │
                              │  HTTPS (≤ 10 concurrent at all times)
                              ▼
                    ┌──────────────────────┐
                    │     NetSuite ERP     │
                    │  concurrency limit=10│
                    └──────────────────────┘
                              │
          ┌───────────────────┴────────────────────┐
          │  Webhook callback to originating svc    │
          │  OR polling via GET /jobs/:id           │
          └─────────────────────────────────────────┘
```

---

## Component Breakdown

### 1. REST API Layer

The gateway exposes a simple HTTP API that all internal services use instead of calling NetSuite directly.

```
POST /jobs
{
  "operation": "create_sales_order",
  "priority": "critical",
  "payload": { ... },
  "callback_url": "https://orders-service/webhooks/netsuite"
}

→ 202 Accepted
{
  "job_id": "job_abc123"
}
```

```
GET /jobs/job_abc123

→ 200 OK
{
  "job_id": "job_abc123",
  "status": "completed",
  "response": { ... },
  "completed_at": "2025-01-15T10:00:03Z"
}
```

An optional `?wait=true` flag allows synchronous behavior for services that require an immediate response (e.g., real-time customer creation during checkout), subject to a configurable timeout.

---

### 2. Priority Queue (Redis Sorted Set)

Jobs are stored in a Redis Sorted Set. The score encodes both priority tier and arrival time, guaranteeing that higher-priority jobs always preempt lower-priority ones, while preserving FIFO order within the same tier.

```
score = priority_tier × 10¹² + unix_timestamp_ms
```

| Tier | Priority | Example Operations |
|---|---|---|
| 0 | CRITICAL | Pre-deadline sales order creation |
| 1 | HIGH | Post-deadline orders, customer creation |
| 2 | MEDIUM | Delivery attachments, order updates |
| 3 | LOW | Product syncs, bulk reads |

**Example scores:**

```
CRITICAL order  (t=09:59:50) → 0_000001750000000000  ← dequeued first
HIGH     order  (t=10:01:00) → 1_000001750000100000
LOW      sync   (t=09:45:00) → 3_000001749990000000  ← dequeued last
```

A lightweight **deadline monitor** watches for jobs approaching the order cutoff and promotes them to CRITICAL, preventing low-priority flood from starving time-sensitive work.

---

### 3. Worker Pool + Semaphore

A pool of workers continuously pulls from the priority queue. A semaphore with a count of 10 ensures that no more than 10 requests are in-flight to NetSuite at any moment.

```
loop:
  acquire semaphore        ← blocks if 10 workers already active
  job = ZPOPMIN queue      ← dequeue highest-priority job
  response = HTTPS POST to NetSuite
  store result in Redis
  send webhook callback to originating service
  release semaphore        ← allows next worker to proceed
```

This is the single enforcement point for the concurrency constraint. Because all microservices route through the gateway, there is no way for a service to bypass it.

---

### 4. Job Result Store (Redis Hash)

Each job's result is stored by `job_id` for a configurable TTL (e.g., 24 hours). This supports both callback delivery and polling.

```
job_abc123 → {
  status:       "completed",
  response:     { netsuite_id: "SO-00123", ... },
  attempts:     1,
  error:        null,
  completed_at: "2025-01-15T10:00:03Z"
}
```

---

### 5. Retry + Circuit Breaker

**Retry policy:** On transient failures (5xx, timeouts), the job is re-enqueued with exponential backoff up to 3 attempts before being routed to the Dead Letter Queue (DLQ) and triggering an alert.

**Circuit breaker states:**

```
CLOSED (normal)
  → error_rate < 50% over a 60-second rolling window
  → all jobs proceed normally

OPEN (tripped)
  → fast-fail all new jobs with 503
  → prevents thundering herd against a degraded NetSuite

HALF-OPEN (recovery probe)
  → allow 1 request through every 30 seconds
  → if successful, transition back to CLOSED
```

---

## Throughput Estimate

With 10 concurrent slots and an average latency of ~1.1 seconds (midpoint of 200ms–3s range):

```
Throughput ≈ 10 / 1.1s
           ≈ ~9 requests/second
           ≈ ~540 requests/minute
           ≈ ~32,400 requests/hour
```

With a 2-hour pre-deadline processing window, the system can handle approximately **65,000 NetSuite operations** — all CRITICAL-tier sales order creation completing before any lower-priority work begins.

---

## Trade-offs and Considerations

| Concern | Trade-off |
|---|---|
| Added network hop | All requests go through the gateway, adding ~1–5ms latency. Acceptable given NetSuite's 200ms–3s baseline. |
| Gateway as single point of failure | Run multiple gateway instances behind a load balancer; Redis provides shared queue state. |
| Async by default | Services must handle async responses via callbacks or polling. Synchronous callers can use `?wait=true` with a timeout. |
| Queue depth under sustained load | If order volume outpaces throughput, queue grows. Monitor queue depth and alert when backlog risks missing the deadline window. |
| Redis durability | Use Redis with AOF persistence or a managed Redis service (e.g., ElastiCache) to avoid losing queued jobs on restart. |

---

## Summary

| Problem | Solution |
|---|---|
| N services racing for 10 NetSuite slots | Single gateway with semaphore — one enforcement point |
| Pre-deadline orders getting starved | CRITICAL tier always dequeued before lower tiers |
| NetSuite outage cascading to all services | Circuit breaker fast-fails, DLQ retries on recovery |
| Observability into NetSuite operations | Job result store provides full per-operation audit trail |
| Scaling to more microservices | Transparent — services call the gateway API, not NetSuite |
