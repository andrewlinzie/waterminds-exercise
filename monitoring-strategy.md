# Monitoring Strategy

## Overview

The main monitoring problem is that WaterMinds is operating a shared multi-tenant platform without enough tenant-level visibility.

Basic CloudWatch metrics may show that the platform is healthy or unhealthy overall, but they do not answer the most important scaling questions:

- Which customers are consuming the most resources?
- Which shared systems are approaching capacity?
- Where are bottlenecks forming?
- Is one customer creating noisy-neighbor risk for others?

The monitoring strategy should make the platform visible at three levels:

1. Platform health: Is AWS/EKS/Kafka/Postgres/Snowflake healthy?
2. Service health: Are APIs, ingestion jobs, and customer-facing workflows meeting reliability targets?
3. Tenant health: Which customers are driving usage, cost, latency, or risk?

The goal is not just dashboards. The goal is proactive capacity planning.

---

## 1. Key Metrics to Predict Scaling Needs

## Metric 1: API request rate and p95 latency by customer

### What to monitor

- API request count by `customer_id` or customer tier.
- API p95 latency by route.
- API error rate by route and customer.
- Top customers by request volume.

### Thresholds

- p95 latency above 500ms for common read endpoints for 10+ minutes.
- 5xx error rate above 2% for 5+ minutes.
- One customer generating more than 25-30% of total API traffic.
- HPA at max replicas while latency is still increasing.

### Why this metric matters

The API is the user-facing layer. Customers feel API slowness directly through dashboards, reports, and application workflows. API metrics are also useful because they often reveal downstream problems in Postgres, Snowflake, Kafka, or internal services.

---

## Metric 2: Postgres connection usage and query latency by schema

### What to monitor

- Active connections as a percentage of max connections.
- Connections by service.
- Query count by schema.
- Slow queries by schema.
- p95 query latency for core application queries.
- Database CPU, memory, IOPS, and lock waits.

### Thresholds

- Active connections above 70% of max connections for 10+ minutes.
- Active connections above 85% should trigger urgent action.
- p95 query latency above 250-500ms for core queries.
- One schema/customer responsible for more than 25-30% of query load.
- Increasing lock waits or long-running transactions.

### Why this metric matters

With schema-per-customer, the system may not break because of the number of schemas alone. It is more likely to break because of too many connections, slow queries, or one tenant creating heavy load. Postgres is also a likely bottleneck because many API paths depend on it.

---

## Metric 3: Kafka message volume and consumer lag by customer/topic

### What to monitor

- Messages per second by topic.
- Bytes in/out by topic.
- Consumer lag by consumer group.
- Message volume by customer/facility when available.
- Broker disk usage, CPU, memory, and network throughput.
- Failed or delayed message processing.

### Thresholds

- Consumer lag grows for 15-30 minutes without recovering.
- Broker disk usage above 70%.
- Broker CPU or network above 70% sustained.
- One customer generating more than 25-30% of total telemetry volume.
- Ingestion-to-availability latency exceeds the data freshness target.

### Why this metric matters

Kafka is the live telemetry ingestion layer. If Kafka falls behind, customers may see stale or delayed facility data even if the API and UI are still up.

---

## Metric 4: Snowflake warehouse utilization, queueing, and cost by workload/customer

### What to monitor

- Warehouse queue time.
- Query duration.
- Bytes scanned per query.
- Credits consumed by warehouse.
- Query tags by customer, workload, and service.
- Data freshness for Bronze/Silver/Gold pipelines.

### Thresholds

- Warehouse queue time above 1-2 minutes.
- Data freshness SLO missed for ingestion or transformation jobs.
- One customer/workload driving more than 25-30% of warehouse credits.
- Repeated customer-scoped queries scanning far more data than expected.
- BI/reporting workloads slowing API-serving or transformation workloads.

### Why this metric matters

Snowflake may not fail in the same way as Postgres, but cost, queueing, and freshness can degrade as usage grows. Separating compute by workload only works if warehouse usage is visible.

---

## Metric 5: EKS pod, node, and autoscaling saturation

### What to monitor

- Pod CPU and memory by service.
- Node CPU and memory.
- Pending pods.
- Pod restarts and OOM kills.
- HPA current vs desired replicas.
- HPA max replicas reached.
- Cluster autoscaler/Karpenter scaling events.
- Service availability and readiness.

### Thresholds

- Pods above 70% CPU or memory for 10+ minutes.
- Nodes above 75% CPU or memory sustained.
- HPA at max replicas for 10+ minutes.
- Pending pods due to insufficient CPU or memory.
- OOM kills or restart loops increasing.
- Available replicas below desired replicas.

### Why this metric matters

EKS is the shared runtime for the application layer. HPA can add pods, but only if the cluster has room to schedule them. Node and pod saturation show whether the platform can absorb more customer traffic.

---

## 2. Per-Customer Visibility

The most important monitoring improvement is adding customer-level attribution across shared systems.

Every request, message, job, and query should be attributable to a customer or customer tier where practical.

## APIs

Track:

- request count by customer
- error rate by customer
- latency by customer and route
- rate-limit events by customer
- top customers by traffic volume

Implementation notes:

- Add safe labels such as `customer_id_hash`, `customer_tier`, `route`, `status_code`, and `method`.
- Avoid labels with high-cardinality or sensitive data such as raw user IDs, request IDs, device IDs, or request bodies.
- Use logs for detailed investigation and metrics for aggregate trends.

## Postgres

Track:

- connections by application/service
- query count by schema
- slow queries by schema
- table size by schema
- index bloat or missing indexes for large schemas
- long-running transactions

Implementation notes:

- Use schema naming conventions that map clearly to customer IDs.
- Use Postgres exporter plus query analysis tools.
- Use application logs/query tags where direct schema attribution is not available from default metrics.

## Kafka

Track:

- messages per customer
- bytes ingested per customer
- consumer lag by topic/consumer group
- failed message counts by customer if available
- top customers by telemetry volume

Implementation notes:

- Include customer/facility context in message metadata where possible.
- Use topic/partition strategy that avoids hot partitions.
- Avoid per-device metric labels if device count is high.

## Snowflake

Track:

- credits by warehouse
- query cost by workload/customer using query tags
- data scanned by query
- queueing by warehouse
- data freshness by pipeline stage

Implementation notes:

- Use query tags to attribute usage to customer, workload, service, and environment.
- Separate warehouses by workload so cost and performance problems are easier to isolate.
- Build a cost/capacity dashboard showing top customers and top workloads.

## Cost Allocation

Track shared cost by:

- customer traffic volume
- telemetry message volume
- Snowflake credits/query tags
- storage growth by customer
- API usage by customer tier
- dedicated infrastructure usage where applicable

The goal is not perfect billing precision at first. The goal is enough attribution to identify the largest cost drivers and noisy tenants.

---

## 3. Tooling

## CloudWatch

Use CloudWatch for AWS-native metrics, logs, and alarms.

Good uses:

- EKS node metrics
- RDS/Postgres infrastructure metrics
- EC2 host metrics if applicable
- CloudWatch Logs
- AWS alarms
- basic infrastructure health

## Prometheus

Use Prometheus for Kubernetes and application metrics.

Good uses:

- pod CPU/memory
- pod restarts
- HPA behavior
- service request rate
- service latency
- service error rate
- Kafka exporter metrics
- Postgres exporter metrics

## Grafana

Use Grafana for dashboards across CloudWatch, Prometheus, and custom metrics.

Dashboards I would build first:

1. Executive/platform health dashboard
2. API health and latency dashboard
3. Tenant usage dashboard
4. Postgres capacity dashboard
5. Kafka ingestion dashboard
6. Snowflake warehouse/cost dashboard
7. EKS capacity/autoscaling dashboard

## What I would instrument first

Priority order:

1. API request rate, latency, and errors by route/customer.
2. Postgres connections, query latency, and schema-level load.
3. Kafka lag and message volume by topic/customer.
4. EKS pod/node saturation and HPA behavior.
5. Snowflake warehouse queueing, query tags, and credit usage.

Reasoning:

The API is closest to user experience. Postgres and Kafka are likely early bottlenecks. EKS shows runtime capacity. Snowflake shows cost, analytics pressure, and data freshness.

---

## 4. Alerting Strategy

Alerts should be actionable and tied to customer impact or scaling risk. I would avoid paging on raw CPU alone unless it is paired with saturation, latency, errors, or failed scaling.

## Alert 1: API degradation

### Trigger

- API p95 latency above 500ms for 10 minutes, or
- 5xx error rate above 2% for 5 minutes, or
- HPA at max replicas while latency continues rising.

### Who gets paged

- Platform/on-call engineer first.
- API service owner if issue appears application-specific.
- Data/platform owner if downstream Postgres/Snowflake/Kafka is involved.

### Action

- Check whether issue follows a deployment.
- Identify top customer traffic during the window.
- Check API pods, HPA, and node saturation.
- Check downstream dependency latency.
- Scale pods/nodes if capacity is the issue.
- Apply tenant rate limiting if one customer is driving the incident.
- Roll back recent deployment if degradation correlates with release.

---

## Alert 2: Kafka ingestion backlog

### Trigger

- Consumer lag grows for 15-30 minutes without recovery.
- Ingestion-to-availability latency exceeds the freshness target.
- Broker disk usage above 70%.
- One customer spikes above 25-30% of message volume and lag rises.

### Who gets paged

- Data platform/on-call engineer.
- Platform engineer if broker capacity is involved.
- Customer support/CS only if customer-facing freshness is impacted.

### Action

- Identify topic, consumer group, and customer/facility driving lag.
- Check consumer health and recent deployments.
- Scale consumers if processing is the bottleneck.
- Add partitions if topic throughput is constrained.
- Check broker disk/network saturation.
- Consider isolating high-volume tenant traffic if repeated.

---

## Alert 3: Postgres capacity risk

### Trigger

- Active connections above 85% of max connections for 5+ minutes.
- Active connections above 70% sustained for 30+ minutes.
- p95 core query latency above 500ms for 10 minutes.
- One schema/customer drives more than 25-30% of load during degradation.

### Who gets paged

- Platform/on-call engineer.
- Database owner if available.
- API owner if connection growth is caused by service behavior.

### Action

- Check connection pool usage.
- Identify top schemas/customers by query load.
- Kill or investigate long-running queries if safe.
- Tune connection pool settings.
- Add indexes or tune queries after incident stabilization.
- Scale Postgres vertically if pressure is broad.
- Isolate tenant if repeated noisy-neighbor behavior is proven.

---

## Capacity Planning Cadence

I would review capacity weekly during the scale-up period and monthly once the platform is stable.

Weekly review should include:

- top customers by API traffic
- top customers by Kafka telemetry volume
- top schemas by Postgres query/load
- Snowflake credits by warehouse/workload/customer
- EKS HPA and node scaling events
- capacity headroom by major system
- incidents or alerts from the previous week

The key output should be a simple capacity report:

- what is near limit
- what customer or workload is driving it
- what action is recommended
- when action is needed
- whether the fix is vertical scaling, horizontal scaling, throttling, optimization, or tenant isolation

---

## Summary

The monitoring strategy should move WaterMinds from platform-level visibility to tenant-aware visibility.

Basic infrastructure metrics are not enough for a shared SaaS platform. The team needs to know which customers are driving load, which shared systems are approaching limits, and when user-facing reliability is at risk.

The first monitoring priority is API, Postgres, Kafka, EKS, and Snowflake visibility with customer attribution where practical. Alerts should be based on sustained degradation or capacity risk, not noisy one-off spikes.
