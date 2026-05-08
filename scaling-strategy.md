# Scaling Strategy

## Overview

WaterMinds is starting with a shared multi-tenant platform and needs to grow from roughly 10 customers to 100+.

I would not immediately split every customer into their own dedicated infrastructure. That adds unnecessary cost and operational complexity too early. My default approach would be to keep the platform shared as long as it remains safe, observable, and cost-effective.

The main risk is not the customer count on its own. The bigger risk is the noisy neighbor problem. One large utility may generate far more telemetry, API traffic, database load, or Snowflake usage than the rest.

My approach:

- Scale vertically first when pressure is broad and predictable.
- Scale horizontally when throughput, concurrency, or hard limits require it.
- Isolate specific tenants when they create noisy neighbor risk, compliance risk, or unacceptable blast radius.
- Use tenant level metrics before making big architecture changes.

---

## 1. Data Layer Scaling

### Postgres

Current pattern: one shared Postgres database with schema-per-customer.

I do not think 100 schemas alone is the breaking point. I think the likely breaking points are connections, slow queries, schema migration overhead, noisy tenants, and backup/restore blast radius.

### What I'd Do First

- Add or tune PgBouncer/RDS Proxy for connection pooling.
- Set connection limits per service.
- Track query latency and connection usage by schema
- Tune indexes for high-volume schemas.
- Scale the database instance up if pressure is broad across most tenants.

### Thresholds I'd Watch

- Consumer lag grows for 15-30 minutes without recovery.
- Broker disk above 70%.
- Broker CPU or network above 70% sustained.
- One customer produces more than 25-30% of telemetry volume.
- Ingestion-to-availability latency misses the freshness target.

I would start with shared topics by domain/workload. If one customer becomes a repeat noisy neighbor, I would move them to dedicated topics/partitions. Dedicated clusters would be a later step for extreme scale, compliance, or blast-radius reasons.

---

## 2. Application Layer Scaling

### EKS

Current pattern: tenants in a shared EKS cluster with tenant context passed through APIs.

It's possible for a shared EKS cluster to still work at 100 customers if request/limits, HPA, cluster autoscaling, and tenant-aware monitoring are in a strong place.

#### Scaling Order

1. Use HPA to scale pods
2. Use Cluster Autoscaler or Karpenter to add nodes when pods cannot be scheduled.
3. Use namespaces, resource quotas, and node groups for workload isolation.
4. Consider multiple clusters only when compliance, blast radius, or operational complexity requires it.

HPA alone is not enough if the cluster has no room for more pods.

#### Thresholds I'd Watch

- API pods above 65-70% CPU or memory during normal traffic.
- HPA at max replicas for 10+ minutes
- Pending pods due to insufficient CPU/memory.
- Nodes above 75% CPU or memory sustained.
- Increasing pod restarts, OOM kills, or evictions.
- p95 latency exceeding the SLO while resources are saturated

I wouldn't create multiple clusters just because the platform reaches 100 customers. I would consider it once things like isolation, compliance, or blast-radius risk justify it.

### APIs

Current pattern: APIs filter by `customer_id`.

The API layer is the place where I'd be the most careful with tenant context. Every request should have tenant context validated at the auth boundary and carried safely through downstream calls.

#### What I'd Focus On

- Connection pooling to avoid exhausting Postgres.
- Per-tenant rate limits to control noisy customers.
- Tenant-safe caching for repeated reads.
- Metrics by route, customer/tier, latency, and error rate.

For Postgres, I would avoid unbounded connection pools per schema. I would use PgBouncer/RDS Proxy and define clear service-level limits.

#### Caching Triggers

I would add caching when:

- p95 API latency is above 500ms for common read endpoints. 
- Dashboards repeatedly hit the same expensive queries.
- Database CPU/query latency is rising from read-heavy traffic.
- semi-static aggregate data is being repeatedly pulled from Snowflake or Postgres.

---

## 3. Scaling Decision Framework

| Customer Count | Main Risk | Scaling Actions |
|---|---|---|
|25 customers | Need baselines before growth hides problems | Add tenant-level metrics, tune Postgres connection pooling, define API latency/error targets, monitor Kafka lag, add Snowflake query tagging, and validate HPA works as expected |
| 50 customers | Shared systems may show noisy neighbor patterns | Scale Postgres instance size vertically if pressure is broad, tune indexes, enforce per-tenant rate limits, increase Kafka partitions for high-volume topics, separate Snowflake warehouses by workload, and add cluster autoscaling if not already enabled |
| 100 customers | Noisy neighbor and blast radius become bigger concerns | Move high-volume tenants to dedicated Postgres databases if needed, dedicate Snowflake warhouses for heavy workloads, isolate noisy Kafka producers with dedicated topics/partitions, and use EKS namespaces/resource quotas/node groups to reduce noisy neighbor risk. I'd only evaluate additional clusters if compliance, customer requirements, or blast radius requires it |
| 100+ customers | Need formal tenant tiering and capacity forecasting | Create tenant tiers, define isolation criteria, automate capacity reporting, and standardize dedicated infrastructure patterns for large or regulated customers |

---

## Decision Rule

The decision rule I would use:

- If pressure is broad and predictable, scale vertically first.
- If pressure is caused by concurrency or throughput, scale horizontally.
- If pressure is caused by one tenant, isolate or throttle that tenant.
- If pressure creates customer-facing degradation, tie the response to the affected SLO.
- If isolation requirements are regulatory or contractual, treat them as architecture requirements, not optimization decisions.

The goal would be to keep the platform shared while it is safe and efficient, but have clear operational thresholds for when to add capacity or isolate workloads.
