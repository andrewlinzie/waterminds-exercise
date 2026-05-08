# waterminds-exercise

This repo contains my written response for the WaterMinds DevOps take-home exercise.

The scenario consists of a shared multi-tenant SaaS platform for water and wastewater utilities running in AWS GovCloud. The platform currently supports a small group of pilot customers and needs to scale toward 100+ customers over the next 18 months.

## Documents

- [Scaling Strategy](./scaling-strategy.md)
  How I would approach scaling the data/application layers from 10 customers to 100+.

- [Monitoring Strategy](./monitoring-strategy.md)
  How I would approach monitoring, tenant-level visibility, capacity planning, tooling, and alerting

- [Security Patching Strategy](./security-patching-stategy.md)
  How I would approach emergency patching across containers, dependencies, EKS, and EC2 based systems.

## Guiding Principles

My approach is based on a few practical principles:

1. Keep infra shared where it is still safe, cost-effective, and observable
2. Add tenant-level visibility before making major scaling decisions.
3. Scale vertically first when it reduces complexity, then scale horizontally when workload growth or noisy neighbor risk justifies it.
4. Monitor for actionable capacity and reliability signals, not just dashboard noise.
5. Automate routing operations while guarding production risk behind approval gates.
6. Treat verification, rollback, and auditability as part of the operating process.
