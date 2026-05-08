# Scaling Strategy

## Overview

WaterMinds is currently a shared multi-tenant platform supporting approximately 5-10 pilot customers. The target state is 100+ customers over the next 18 months.

I would not immediately split every customer into isolated infrastructure. That would increase cost and operational complexity too early. I would keep the platform shared where it remains safe, observable, and cost-effective, while adding clear thresholds for when to scale vertically, scale horizontally, or isolate high-usage tenants.

The main scaling risk is not customer count alone. The bigger risk is uneven customer load. One large utility may generate significantly more telemetry, API traffic, database activity, or Snowflake compute usage than smaller utilities. Scaling decisions should therefore be based on tenant-level usage patterns, not only the total number of customers.

Guiding approach:

1. Measure tenant-level usage before making major scaling decisions.
2. Scale vertically first when it buys time and avoids unnecessary complexity.
3. Scale horizontally when shared systems approach hard limits or noisy-neighbor risk becomes clear.
4. Isolate large or high-risk tenants only when their workload, compliance needs, or blast radius justify it.

---

## 1. Data Layer Scaling

## Postgres

Current pattern: one shared Postgres database using schema-per-customer.

This pattern can work at 100 customers if usage is moderate and the platform has strong connection pooling, query visibility, and migration discipline. The number of schemas alone is not the main breaking point. The system is more likely to break because of connection exhaustion, slow queries, noisy tenants, schema migration overhead, index bloat, or backup/restore complexity.

### What breaks first

Likely pressure points:

- Too many active database connections from APIs and background jobs.
- Slow queries in one or more customer schemas.
- One large tenant consuming disproportionate database CPU, memory, or IOPS.
- Schema migrations becoming slower and riskier across many schemas.
- Backup and restore blast radius becoming too large.

### Scaling approach

I would start with vertical scaling and operational controls:

- Add or tune PgBouncer/RDS Proxy for connection pooling.
- Set per-service and per-tenant connection limits.
- Track query latency and connection usage by schema.
- Tune indexes for high-volume schemas.
- Increase instance size when CPU, memory, or IOPS pressure is broad across the database.

Thresholds that would trigger action:

- Active connections consistently above 70% of max connections.
- p95 query latency above 250-500ms for core app queries for 15+ minutes.
- CPU above 65-70% sustained during normal traffic.
- Read/write IOPS above 70% of provisioned capacity.
- One tenant consistently responsible for more than 25-30% of database load.

### When to shard

I would not shard automatically at 50 or 100 customers. I would shard when load distribution proves the shared model is creating risk.

Preferred sharding model:

- Keep standard tenants in the shared Postgres database.
- Move large/noisy tenants to dedicated databases or dedicated clusters.
- Use a tenant routing map: `customer_id -> database/schema`.
- Treat dedicated databases as a pressure-release valve or premium isolation tier, not the default for every customer.

I would avoid separate databases for every customer at 100 customers unless compliance or customer contract requirements demand it. That model improves isolation but increases operational overhead, migration complexity, and cost.

---

## Snowflake

Current pattern: one Bronze/Silver/Gold data lake with all customers in shared tables using row-level security.

Snowflake can scale well with this model if data is organized correctly and compute is isolated by workload. I would separate compute before separating data.

### What breaks first

Likely pressure points:

- Queries scanning too much shared data.
- Warehouse queueing during heavy analytics workloads.
- One customer or workload consuming disproportionate compute credits.
- Poor cost attribution by tenant or workload.
- Governance risk if row-level security policies become difficult to validate.

### Scaling approach

I would keep the shared data model initially, but improve organization and compute isolation:

- Cluster or organize large tables around `customer_id` and time-based fields.
- Use query tags to track customer, workload, environment, and service.
- Separate warehouses by workload type:
  - ingestion/transformation
  - API-serving extracts
  - BI/reporting
  - data science/modeling
- Use larger or multi-cluster warehouses only when queueing or concurrency requires it.
- Consider dedicated warehouses for high-volume customers or heavy reporting workloads.

Thresholds that would trigger action:

- Warehouse queue time consistently above 1-2 minutes.
- Queries repeatedly scanning far more data than expected for customer-scoped requests.
- One customer or workload driving more than 25-30% of warehouse credits.
- Data freshness SLOs missed because transformation jobs cannot complete on time.
- Row-level security or governance rules becoming difficult to test and audit safely.

### When to rethink shared tables

I would rethink the shared-table model if customer-specific data volume, compliance requirements, or performance isolation needs make logical separation insufficient.

Before splitting tables by customer, I would first try:

- better clustering/partition strategy
- warehouse isolation
- query optimization
- materialized views or derived customer-specific serving tables
- stricter query tagging and cost attribution

---

## Kafka

Current pattern: shared Kafka cluster ingesting Sparkplug B MQTT telemetry from all customers.

Kafka should scale based on message volume, consumer lag, broker saturation, and tenant skew, not customer count alone.

### What breaks first

Likely pressure points:

- Consumer lag grows because messages arrive faster than downstream consumers process them.
- Broker disk or network throughput approaches limits.
- Too few partitions create hot partitions.
- One customer or facility produces much more telemetry than others.
- Retention settings cause storage growth faster than planned.

### Scaling approach

I would scale Kafka horizontally before relying only on larger brokers:

- Increase partitions for high-volume topics.
- Add brokers when CPU, disk, network, or partition density approaches limits.
- Use partition keys that distribute load across customer/facility/device instead of creating hot partitions.
- Track message volume, bytes in/out, and lag by customer or topic.
- Create dedicated topics or clusters only for very large or high-risk tenants.

Thresholds that would trigger action:

- Consumer lag grows for 15-30 minutes without recovering.
- Broker disk usage above 70%.
- Broker CPU or network above 70% sustained.
- One customer produces more than 25-30% of total message volume.
- Ingestion-to-availability latency exceeds the data freshness target.

### Tenant isolation

I would start with shared Kafka topics organized by domain/workload. As tenant skew appears, I would move high-volume customers to dedicated topics or partitions. Dedicated Kafka clusters should be reserved for extreme scale, compliance isolation, or major blast-radius concerns.

---

## 2. Application Layer Scaling

## EKS

Current pattern: shared EKS cluster with tenant context passed through APIs.

A shared EKS cluster can support 100 customers if workload resource requests/limits, HPA, cluster autoscaling, and tenant-aware monitoring are in place.

### Scaling approach

I would scale EKS in layers:

1. Pod scaling with HPA.
2. Node scaling with Cluster Autoscaler or Karpenter.
3. Workload isolation using namespaces, resource quotas, and node groups.
4. Multiple clusters only when isolation, compliance, or blast-radius concerns justify it.

HPA alone is not sufficient if the cluster does not have node capacity. HPA adds pods, but cluster autoscaling is needed when the existing nodes cannot schedule more pods.

Thresholds that would trigger action:

- API pods consistently above 65-70% CPU or memory during normal load.
- HPA at max replicas for 10+ minutes.
- Pending pods due to insufficient CPU or memory.
- Node CPU or memory above 75% sustained.
- Pod restarts, OOM kills, or eviction events increasing.
- p95 API latency exceeding SLO while infrastructure metrics are saturated.

### Multiple clusters

I would not move to multiple clusters simply because the platform reaches 100 customers. I would evaluate additional clusters when:

- customer contracts require stronger isolation
- GovCloud/account boundaries require it
- cluster upgrades become too risky for one shared cluster
- blast radius becomes unacceptable
- workloads have very different scaling/security profiles
- regional or customer-specific deployment requirements emerge

---

## APIs

Current pattern: APIs filter by `customer_id`.

The API layer is the customer-facing control point for tenant isolation. Every request should have tenant context validated at the auth boundary and carried safely through downstream calls.

### Scaling concerns

Likely pressure points:

- Too many API requests.
- Too many database connections.
- Repeated expensive queries.
- Slow customer dashboards.
- Tenant filtering mistakes.
- No per-tenant rate limiting.

### Connection pooling

For 100 customer schemas, the API should not open unbounded connections per request or per schema. I would use connection pooling with clear limits.

Approach:

- Use PgBouncer or RDS Proxy.
- Set max connections per service.
- Track active connections by schema/customer.
- Avoid one large uncontrolled pool per customer if it risks exhausting Postgres.
- Use short-lived request-scoped tenant context, not long-lived tenant-specific connections when possible.

### Caching

Caching becomes necessary when repeated reads or expensive dashboard queries start increasing latency or database load.

Good cache candidates:

- customer metadata/configuration
- dashboard summaries
- latest telemetry snapshots
- expensive aggregate results
- frequently accessed read-only reference data

Tenant-safe cache keys are mandatory.

Example:

```text
tenant:{customer_id}:dashboard:{dashboard_id}:summary
```

Thresholds that would trigger caching:

- p95 API latency above 500ms for common read endpoints.
- repeated database queries for the same dashboard data.
- database CPU or query latency rising because of read-heavy workloads.
- Snowflake/Postgres repeatedly serving semi-static aggregate data.

---

## 3. Scaling Decision Framework

| Customer Count | Main Risk | Scaling Actions |
|---|---|---|
| 25 customers | Need baselines before growth hides problems | Add tenant-level metrics, tune Postgres connection pooling, define API latency/error SLOs, monitor Kafka lag, enable Snowflake query tagging, confirm HPA works |
| 50 customers | Shared systems may show noisy-neighbor patterns | Increase Postgres instance size if broad pressure exists, tune indexes, enforce per-tenant rate limits, increase Kafka partitions for high-volume topics, separate Snowflake warehouses by workload, add cluster autoscaling/Karpenter if not already enabled |
| 100 customers | Tenant skew and blast-radius risk become more important | Move high-volume tenants to dedicated Postgres databases if needed, dedicate Snowflake warehouses for heavy workloads, isolate noisy Kafka producers with dedicated topics/partitions, use EKS namespaces/resource quotas/node groups, evaluate additional clusters only if compliance or blast radius requires it |
| 100+ customers | Need formal tenant tiering and capacity forecasting | Create tenant tiers, define isolation criteria, automate capacity reporting, standardize dedicated infrastructure patterns for large or regulated customers |

---

## Decision Rule

The decision rule I would use is:

- If pressure is broad and predictable, scale vertically first.
- If pressure is caused by concurrency or throughput, scale horizontally.
- If pressure is caused by one tenant, isolate or throttle that tenant.
- If pressure creates customer-facing degradation, tie the response to the affected SLO.
- If isolation requirements are regulatory or contractual, treat them as architecture requirements, not optimization decisions.

The goal is to keep the platform shared while it is safe and efficient, but have clear operational thresholds for when to add capacity or isolate workloads.
