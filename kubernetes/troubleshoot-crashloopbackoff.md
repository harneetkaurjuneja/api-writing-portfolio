# Troubleshoot a Kubernetes Pod in `CrashLoopBackOff`

> **Simulated portfolio exercise.** This article documents an illustrative Kubernetes troubleshooting scenario; no live production cluster was accessed.

## Identify the issue

```bash
kubectl get pods -n payments
```

The simulated `payment-api-8k2m` Pod shows `0/1` readiness and `CrashLoopBackOff`, with five recorded restarts. The `postgres-0` Pod is `Pending`.

## Investigate the application

```bash
kubectl logs payment-api-8k2m -n payments --previous
```

Simulated previous logs report a refused database connection, aborted startup, and exit code `1`.

## Check the database

```bash
kubectl get svc -n payments
kubectl get endpoints postgres -n payments
kubectl describe pod postgres-0 -n payments
kubectl describe nodes
```

The PostgreSQL Service exists, but the simulated endpoint listing has no backends. The PostgreSQL Pod has a `FailedScheduling` event reporting insufficient memory on both Nodes. The Pod requests `2Gi` of memory; one Node has `1Gi` and the other `0.5Gi` of unallocated schedulable memory. Neither can accommodate the request.

## Proposed remediation

Review resource requirements and change-management approvals. Add a Node with sufficient capacity or increase available capacity on an eligible existing Node. Do not reduce PostgreSQL's memory request without validating workload requirements.

## Simulated verification

After a hypothetical successful capacity change:

```bash
kubectl get pods -n payments
kubectl logs payment-api-8k2m -n payments
```

The simulated outputs show `postgres-0` and `payment-api-8k2m` both `Running` and `1/1` ready. The application logs report successful startup, a PostgreSQL connection, and listening on port `8080`. Readiness and these logs alone do not prove every application function works.

**Note:** Commands, resource names, outputs, and recovery results are illustrative, not evidence of a real production incident.
