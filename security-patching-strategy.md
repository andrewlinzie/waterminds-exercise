# Security Patching Strategy

## Overview

The goal of this strategy is to move security patching from manual, ticket-driven work to a repeatable operating process.

I would separate patching into two tracks:

1. Routine patching: planned, scheduled, and automated as much as possible.
2. Emergency CVE response: faster process for critical vulnerabilities, especially if actively exploited.

The goal is not to patch blindly. The goal is to patch safely, verify the fix, preserve audit evidence, and have rollback or roll-forward options ready.

---

## 1. Layers That Need Patching

### Container Base Images

Container base images should be scanned and rebuilt regularly.

Examples:

- Python base images
- Node base images
- Alpine/Ubuntu/Debian base layers
- OS packages inside images

### Detection

Use image scanning in CI and registry scanning:

- Trivy, Snyk, Grype, or similar scanners in CI
- ECR image scanning after push
- Dependabot or vulnerability alerts where available

### Patch process

1. Update the base image version in the Dockerfile.
2. Rebuild the container image through CI.
3. Run tests.
4. Run vulnerability scan.
5. Push a new immutable image tag.
6. Deploy to dev.
7. Promote to staging and prod after validation.

### Production control

Critical vulnerabilities should block promotion until fixed or explicitly accepted as an exception.

---

## Application Dependencies

Application dependencies should be scanned before merge and before image promotion.

Examples:

- `requirements.txt`
- `package.json`
- `package-lock.json`
- `pom.xml`
- `build.gradle`

### Detection

Use dependency scanning in CI:

- Dependabot
- Snyk
- pip-audit
- npm audit
- OWASP Dependency-Check
- Trivy filesystem scan

### Patch process

1. Identify vulnerable dependency and fixed version.
2. Update dependency manifest and lock file.
3. Run unit tests and integration tests.
4. Rebuild container image.
5. Scan the rebuilt image.
6. Deploy through dev, staging, and prod.

### Production control

Minor and patch updates can be automated through PRs. Major version upgrades should require human review because they carry higher breakage risk.

---

## EKS / Kubernetes

EKS patching should separate control plane upgrades from worker node patching.

### EKS control plane

The EKS control plane should be upgraded through a controlled process:

1. Upgrade dev first.
2. Validate workloads, Helm charts, ingress, autoscaling, monitoring, and GitOps sync.
3. Upgrade staging.
4. Run smoke tests and workload checks.
5. Schedule prod upgrade window.
6. Upgrade prod with approval.
7. Monitor API latency, pod health, node health, HPA behavior, and errors.

### Worker nodes

Worker node patching should use blue/green managed node groups where possible.

Process:

1. Create a new patched node group using the updated AMI/version.
2. Allow workloads to schedule onto the new nodes.
3. Cordon old nodes so no new pods are scheduled there.
4. Drain old nodes safely.
5. Confirm pods reschedule and pass readiness checks.
6. Monitor errors, latency, pod restarts, and capacity.
7. Remove old node group after a stability window.

This avoids patching nodes in place and gives a cleaner rollback path if the new node group has issues.

---

## EC2 / Operating System Patching

Any EC2-based systems need routine OS patching.

Examples:

- self-managed Kafka brokers if applicable
- Jenkins or build servers
- monitoring hosts
- legacy/internal services
- bastion or admin hosts if used

### Standard EC2 patching

Use AWS Systems Manager Patch Manager for routine OS patching:

- patch baselines
- maintenance windows
- instance tags
- compliance reports
- approved package rules

Example tags:

```text
Environment=dev
Environment=staging
Environment=prod
Workload=kafka
PatchGroup=linux-prod
```

### Application-aware patching

For workloads that need special care, use Ansible or SSM Automation runbooks.

Example for self-managed Kafka:

1. Patch one broker at a time.
2. Confirm replicas are healthy before patching.
3. Apply OS updates.
4. Reboot if required.
5. Confirm broker rejoins the cluster.
6. Verify under-replicated partitions return to zero.
7. Move to the next broker.

Stateful systems should not be patched all at once.

---

## 2. Patching Cadence

Routine patching should follow a predictable schedule.

### Weekly / Continuous

- Dependency vulnerability scans.
- Container image scans.
- CI quality gates.
- Dependabot or automated dependency PRs.
- Review new critical/high vulnerabilities.

### Monthly

- Base image refresh and rebuilds.
- Routine OS patching for dev and staging.
- Review patch compliance dashboard.
- Review exceptions and aging vulnerabilities.

### Quarterly

- EKS version planning.
- Kubernetes add-on review.
- Node AMI refresh.
- Disaster recovery and rollback review.
- Patch process review.

### Promotion Order

Patches should move through environments in order:

```text
dev -> staging -> prod
```
Dev can be more automated.

Staging should validate production-like behavior.

Prod should require explicit approval for high-risk patches, EKS upgrades, database-related changes, and anything requiring a maintenance window.

---

## 3. Emergency CVE Process

For a critical CVE, especially one being actively exploited, I would use a faster risk-based process.

### Step 1: Detect

Sources:

- CI vulnerability scans
- registry image scans
- Dependabot/Snyk alerts
- AWS Inspector
- vendor/security advisories
- AWS security bulletins

### Step 2: Assess impact

Questions:

- Do we use the affected package, image, OS, or Kubernetes version?
- Which services are affected?
- Is the service internet-facing?
- Is the vulnerability exploitable in our configuration?
- Is customer data at risk?
- Is a fixed version available?
- Is there a temporary mitigation?

### Step 3: Patch and validate

Minimum emergency path:

```text
patch -> build -> scan -> test -> deploy dev -> smoke test -> deploy staging -> smoke test -> approve prod -> deploy prod -> verify
```
The process should be faster than routine patching, but not reckless. For actively exploited critical vulnerabilities, staging validation may be shortened, but production should still have a clear approval and verification step.

### Step 4: Communicate

Notify:

- platform/on-call engineer
- service owner
- security/compliance owner
- engineering lead
- customer-facing teams if customer impact is possible

Communication should include:

- affected systems
- severity
- mitigation plan
- target deployment window
- rollback or roll-forward plan
- final verification status

---

## 4. Verification and Rollback

Patching is not complete when the deploy finishes. It is complete when the fix is verified.

### Verification checklist

After patching, confirm:

- vulnerable version is no longer present
- scanner passes or finding is resolved
- new image tag or node version is running
- pods are healthy
- readiness/liveness checks pass
- API health checks pass
- Kafka/Postgres/Snowflake pipelines are healthy if affected
- latency and error rates remain normal
- logs do not show new failures
- dashboards and alerts are clean

### Rollback strategy

Rollback depends on the layer.

#### Containers

- redeploy previous known-good image tag
- revert GitOps image tag commit
- confirm service health after rollback

#### Dependencies

- revert dependency update
- rebuild image
- redeploy previous known-good version

#### EKS worker nodes

- keep old node group during stability window
- move workloads back if new node group causes issues
- avoid deleting old node group until validation is complete

#### EC2 systems

- use AMI/snapshot backup where appropriate
- roll back package if safe
- restore previous image/configuration
- use runbooks for stateful systems

#### Rollback vs roll-forward

If the previous version is critically vulnerable and actively exploited, rollback may not be safe. In that case, the better option may be to roll forward quickly with a corrected patch or temporary mitigation.

---

## 5. Auditability and Reporting

Because the platform runs in AWS GovCloud and serves utility customers, patching should be auditable.

Track:

- vulnerability scan results
- affected services
- patch PRs
- approvals
- deployment timestamps
- image tags
- EKS/node versions
- EC2 patch compliance
- exceptions/risk acceptances
- rollback records

A weekly or monthly patch report should show:

- open critical/high vulnerabilities
- patched systems
- unpatched systems
- exceptions
- upcoming EKS/OS/dependency deadlines
- systems outside compliance

---

## Summary

The patching strategy should automate routine work without removing production control.

Routine patches should be detected through scans, applied through PRs and CI/CD, tested in dev and staging, then promoted to prod with approval when needed.

Emergency CVEs should follow a faster process: detect, assess, patch, test, deploy, verify, and communicate.

The key principles are:

- scan continuously
- patch non-prod first
- promote safely
- verify after deployment
- keep rollback or roll-forward options ready
- maintain audit evidence
