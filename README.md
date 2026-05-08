# waterminds-exercise

This repository contains my written strategy response for the WaterMinds DevOps scenario.

The scenario assumes a shared multi-tenant SaaS platform for water and wastewater utilities running in AWS GovCloud. The platform currently supports pilot customers and needs to scale toward 100+ customers over the next 18 months.

## Documents

- [Scaling Strategy](./scaling-strategy.md)  
  Strategy for scaling the data layer and application layer from 10 customers to 100+ customers.

- [Monitoring Strategy](./monitoring-strategy.md)  
  Monitoring, tenant-level visibility, capacity planning, tooling, and alerting approach.

- [Security Patching Strategy](./security-patching-strategy.md)  
  Routine and emergency patching strategy across containers, dependencies, EKS, and EC2-based systems.

## Guiding Principles

My approach is based on a few operating principles:

1. Keep shared infrastructure where it remains safe, cost-effective, and observable.
2. Add tenant-level visibility before making major scaling decisions.
3. Scale vertically first when it reduces complexity, then scale horizontally when workload growth, noisy-neighbor risk, or isolation needs justify it.
4. Tie monitoring to actionable thresholds, not vanity dashboards.
5. Automate routine operations while keeping production risk behind explicit approval gates.
6. Treat verification, rollback, and auditability as part of the deployment and patching process.
