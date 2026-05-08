# Security Patching Strategy

## Overview

The goal is to move patching away from manual, ticket-by-ticket work toward a repeatable process.

I would split patching into two tracks:

1. Routine patching: planned, scheduled, and mostly automated.
2. Emergency CVE response: faster path for critical vulnerabilities, especially if they're actively exploited.

The main rule I'd follow is to patch quickly, but still verify before calling it done.

---

## 1. What Needs Patching

### Container Base Images

Container images should be scanned in CI and in the image registry

Examples: Python/Node base images, Linux base layers, and OS packages inside images.

Process:

1. Scanner finds vulnerable image layer
2. Update the base image version in the Dockerfile
3. Rebuild the image through CI
4. Run tests and scan the rebuilt image
5. Push a new immutable image tag
6. Deploy to dev, then promote to staging/prod after validation

Critical vulnerabilities should block promotion unless there is an explicit exception.

### Application Dependencies

Application dependency scans should run before merge and before image promotion.

Examples: `requirements.txt`, `package.json`, `package-lock.json`, `pom.xml`, or `build.gradle`.

Process:

1. Identify the vulnerable dependency and the fixed version
2. Update the dependency file and lock file
3. Run unit/integration tests
4. Rebuild and rescan the image
5. Deploy through the normal promotion path

Minor patch updates can usually be handled through automated PRs. Major version changes should get human review because they are more likely to break the app.

### EKS/Kubernetes

For EKS, I'd separate control plane upgrades from worker node patching.

For the control plane, I would upgrade dev first, validate workloads and GitOps sync, then move to staging and production with approval.

Worker nodes, I'd prefer blue/green managed node groups:

1. Create a new patched node group with the updated AMI/version
2. Allow workloads to schedule onto the new nodes
3. Cordon and drain old nodes
4. Confirm pods rescheduled and pass readiness checks
5. Keep the old node group during a short stability window
6. Remove old nodes after validation

This gives a cleaner rollback path than patching nodes in place.

### EC2/Operating Systems

Any EC2-based systems need routine OS patching.

Examples: self-managed Kafka brokers, Jenkins/build servers, monitoring hosts, legacy/internal services, or bastion/admin hosts

For standard OS patching, I'd use AWS Systems Manager Patch Manager with patch baselines, maintenance windows, instance tags, and compliance reporting.

For systems that need application-aware steps, I would use Ansible or SSM Automation runbooks.

For example, if Kafka is self-managed, I'd patch one broker at a time, confirm replicas are healthy, apply OS updates, reboot if needed, confirm the broker rejoins the cluster, and verify under-replicated partitions return to zero.

Stateful systems should not be patched all at once

---

## 2. Patching Cadence

### Weekly/Continous

- Run dependency scans
- Run container image scans
- Review critical/high findings
- Open automated dependency PRs where safe
- Block promotion on critical findings unless an exception is approved

### Monthly 

- Refresh base images
- Rebuild and rescan service images
- Patch dev and staging EC2 instances
- Review patch compliance and exceptions

### Quarterly

- Review EKS version support
- Plan Kubernetes add-on updates
- Refresh worker node AMIs
- Review rollback and disaster recovery procedures

```
dev -> staging -> prod
```

Dev can be more automated. Staging should validate production-like behavior. Prod should require approval for high-risk patches, EKS upgrades, database-impacting changes, or maintenance windows.

---

## 3. Emergency CVE Process

For a critical CVE, especially one being actively exploited, I'd use a faster risk-based process.

### Detect

Inputs:

- CI vulnerability scans
- ECR/image scans
- Dependabot/Snyk alerts
- AWS Inspector
- vendor/security advisories
- AWS security bulletins

### Assess

Questions I'd ask:

- Do we actually use the affected package, image, OS, or Kubernetes version?
- Which services are affected?
- Is the affected service internet-facing?
- Is the vulnerability exploitable in our configuration?
- Is customer data at risk?
- Is a fixed version available?
- Is there a temporary mitigation?

### Patch and Validate

Minimum emergency path:

```
patch -> build -> scan -> test -> deploy dev -> smoke test -> deploy staging -> smoke test -> approve prod -> deploy prod -> verify
```

For an actively exploited critical vulnerability, I would compress the normal schedule, but I would still keep a clear approval and verification step before calling it complete.

### Communicate

Notify the platform/on-call engineer, service owner, security/compliance owner, and customer-facing teams if customer impact is possible

Communication should include the affected systems, severity, mitigation plan, target deployment window, rollback/roll-forward plan, and final verification status.

---

## 4. Verification and Rollback

Patching is not done when the deploy finishes. It is done when the fix is verified.

Verification checklist:

- vulnerable version is no longer present
- scanner passes or the finding is resolved
- new image tag, dependency version, or node version is running
- pods are healthy
- readiness/liveness checks pass
- API health checks pass
- Kafka/Postgres/Snowflake pipelines are healthy if affected
- latency and error rates remain normal
- logs do not show new failures
- dashboards and alerts are clean

Rollback depends on the layer:

- Containers: redeploy the previous known-good image tag or revert the GitOps image tag commit
- Dependencies: revert dependency update, rebuild image, and redeploy known-good version
- EKS worker nodes: keep the old node group during a stability window and move workloads back if needed
- EC2 systems: use AMI/snapshot backup where appropriate and runbooks for stateful systems

If the previous version is critically vulnerable and actively exploited, rollback may not be a safe option. In that case, I'd choose rolling forward with a corrected patch or temporary mitigation.

---

## 5. Auditability and Reporting

Because this runs in AWS GovCloud and supports utility customers, patching should leave an audit trail.

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

A patch report should show open critical/high vulnerabilities, patch systems, unpatched systems, exceptions, and upcoming EKS/OS/dependency deadlines.

---

## Summary

My goal would be to automate routine patching without eliminating production control.

Routine patches should move through CI/CD and a dev/staging/prod promotion. Emergency CVEs should move faster, but still include assessment, validation, communication, and verification.

The key principles are:

- scan continuously
- patch non-prod first when possible
- promote safely
- verify after deployment
- keep rollback or roll-forward options ready
- maintain audit evidence
