# Monitoring Strategy

## Overview

The core monitoring gap is tenant-level visibility.

Basic CloudWatch metrics can show if the platform is healthy overall, but they do not clearly show:

- which customers are using the most resources
- which systems are getting close to capacity
- whether one customer is creating noisy neighbor risk
- where the next bottleneck could form

My approach would be to move from general platform monitoring to customer-aware monitoring. Since this is a multi-tenant platform, I would want the major signals tied back to customer, customer tier, service, or workload.

---

| Metric | Threshold/Trigger | Why it Matters |
|---|---|---|
|API request rate, p95 latency, and error rate by customer | p95 latency > 500ms for 10 minutes, 5xx error rate > 2% for 5 minutes, or one customer > 25-30% of traffic | The API is closest to customer experience and often exposes downstream issues first. | 
| Postgres connections and query latency by schema | Connections > 70% for 10+ minutes, >85% urgent, p95 query latency > 250-500ms, or one schema > 25-30% of query load | With schema-per-customer, connections, slow queries, and tenant skew are more likely to break than the raw number of schemas. |
| Kafka message volume and consumer lag by topic/customer | Lag grows for 15-30 minutes, broker disk > 70%, broker CPU/network > 70%, or one customer > 25-30% of telemetry volume | Kafka is the live telemetry path. If it falls behind, customers may see stale facility data. |
| Snowflake warehouse queueing, credits, and query tags | Queue time > 1-2 minutes, one workload/customer > 25-30% of credits, or freshness targets are missed | Snowflake scaling issues may show up as cost, queueing, or stale data instead of a hard outage. |
| EKS pod, node, and HPA saturation | Pods > 70% CPU/memory, nodes > 75%, HPA maxed for 10+ minutes, pending pods, OOM kills, or rising restarts | HPA only helps if the cluster has room. Pod and node saturation show when the runtime is running out of headroom. |

---

## 2. Per-Customer Visibility

The biggest improvement I'd prioritize is customer-level attribution.

In a shared platform, I'd want to know which customer, customer tier, service, or workload is causing the pressure.

### APIs

Track:

- request count by customer or tier
- latency by route and customer/tier
- error rate by route and customer/tier
- rate-limit events
- top customers by traffic volume

I'd use safe labels like `customer_id_has`, `customer_tier`, `route`, `method`, and `status_code`.

I'd avoid sensitive labels like raw user IDs, device IDs, request IDs, or request bodies.

### Postgres

Track:

- query count by schema
- slow queries by schema
- table growth by schema
- connections by service
- long-running transactions

Schema-level visibility is important since the current model is schema-per-customer.

### Kafka

Track:

- message by topic/customer/facility
- bytes ingested by topic/consumer
- consumer lag by topic and consumer group
- failed message counts
- top telemetry producers

### Snowflake

Track:

- credits by warehouse
- query tags by customer/workload/service
- bytes scanned
- queue time
- query duration
- freshness by Bronze/Silver/Gold stage

The goal here is to have enough visibility to identify noisy tenants and major cost drivers.

---

## 3. Tooling

I'd use a layered approach.

### CloudWatch

I'd use CloudWatch for AWS-native infrastructure visibility:

- EKS/node metrics
- RDS/Postgres infrastructure metrics
- EC2 host metrics if applicable
- CloudWatch Logs
- AWS alarms

### Prometheus

I'd use Prometheus for Kubernetes and application metrics:

- pod CPU/memory
- pod restarts
- HPA behavior
- request rate
- latency
- error rate
- Kafka exporter metrics
- Postgres exporter metrics

### Grafana

I'd use Grafana for dashboards across CloudWatch, Prometheus, and custom metrics.

Dashboards I'd create first:

- API health
- tenant usage
- Postgres capacity
- Kafka ingestion
- Snowflake warehouse/cost
- EKS capacity/autoscaling

First instrumentation priority:

1. API request rate, latency, and errors by route/customer
2. Postgres connections, query latency, and schema-level load.
3. Kafka lag and message volume by topic/customer
4. EKS pod/node saturation and HPA behavior
5. Snowflake queueing, query tags, and credit usage

---

## 4. Alerting Strategy

Alerts should be actionable. I'd avoid alerting on raw CPU alone unless it is tied to saturation, latency, errors, or failed scaling.

### Alert 1: API Degradation

Trigger:

- p95 API latency > 500 ms for 10 minutes
- 5xx error rate > 2% for 5 minutes
- HPA at max replicas while latency keeps rising

Who gets alerted:

- platform/on-call engineer first
- API owner if application specific
- data/platform owner if downstream systems are involved

Actions:

- check recent deployments
- identify top customer traffic during the window
- check API pods, HPA, and node saturation
- check downstream Postgres/Snowflake/Kafka latency
- scale if capacity is the issue
- rate limit or isolate a tenant if one customer is driving the issue
- roll back if the issue lines up with a deployment

### Alert 2: Kafka Ingestion Backlog

Trigger:

- consumer lag grows for 15-30 minutes without recovery
- broker disk > 70%
- ingestion-to-availability latency misses the freshness target
- one customer spikes above 25-30% of message volume and lag rises

Who gets alerted:

- data platform/on-call engineer
- platform engineer if broker capacity is involved

Actions:

- identify topic, consumer group, and customer/facility causing lag
- check consumer health and recent changes
- scale consumers if processing is the bottleneck
- add partitions if topic throughput is constrained
- check broker disk/network saturation
- isolate high-volume tenant traffic if this repeats

### Alert 3: Postgres capacity risk

Trigger: 

- active connections > 85% for 5+ minutes
- active connections > 70% sustained for 30+ minutes
- p95 core query latency > 500ms for 10 minutes
- one schema/customer drives more than 25-30% of load during degradation

Who gets alerted:

- plafgorm/on-call engineer
- database owner if available
- API owner if connection growth is caused by service behavior

Actions:

- check connection pool usage
- identify top schemas/customers by query load
- investigate long-running queries
- tune connection pool settings
- add indexes or tune queries after stabilizing
- scale Postgres vertically if pressure is broad
- isolate tenant if noisy-neighbor behavior repeats

---

## 5. Capacity Planning Cadence

During the scale-up period, I'd review capacity on a weekly cadence.

The weekly review would include:

- top customers by API traffic
- top customers by Kafka telemetry volume
- top schemas by Postgres load
- Snowflake credits by warehouse/workload/customer
- EKS scaling events
- capacity headroom by major system
- alerts/incidents from the previous week

The output should be a simple capacity report:

- what is close to its limit
- which customer or workload is driving it
- what action is recommended
- what action is required
- whether the fix is vertical scaling, horizontal scaling, throttling, optimization, or tenant isolation

---

## Summary

The monitoring strategy should move WaterMinds from a platform with general visibility to tenant-aware visibility.

For a shared SaaS platform, the team will need to know which customers are driving load, which shared systems are approaching limits, and when customer-facing reliability is at risk - basic infrastructure metrics are not enough.

The first monitoring priorities would be API, Postgres, Kafka, EKS, and Snowflake visibility with customer attribution.
