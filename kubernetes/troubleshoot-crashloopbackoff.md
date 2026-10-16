# Troubleshoot a Kubernetes Pod in `CrashLoopBackOff`

> **Simulated portfolio case study.** All Kubernetes resources, commands' outputs, events, and recovery results in this guide are illustrative. No live cluster was accessed and no infrastructure changes were performed.

## Overview

A payment API Pod in the `payments` namespace repeatedly fails during startup. Its previous container logs report a refused PostgreSQL connection. This guide traces the failure through Kubernetes Services and Pod scheduling events, identifies insufficient schedulable memory for the PostgreSQL Pod, and explains a safe remediation and verification approach.

**Audience:** Kubernetes support engineers and technical writers learning infrastructure troubleshooting.

**Prerequisites:** Access to a Kubernetes cluster with `kubectl` configured and read permissions for Pods, Services, EndpointSlices, events, and Nodes. Replace the example resource names with names from your environment. Follow organizational change-control procedures before modifying workloads or cluster capacity.

## Symptoms

- `payment-api-8k2m` is `0/1` ready and in `CrashLoopBackOff`.
- Previous application logs show a refused database connection and startup failure.
- The `postgres-0` Pod is `Pending`.

## Troubleshooting

### 1. Identify affected Pods

**Action:** List the Pods in the application namespace.

```bash
kubectl get pods -n payments
```

**Simulated output:**

```text
NAME                READY   STATUS             RESTARTS
payment-api-7d9x    1/1     Running            0
payment-api-8k2m    0/1     CrashLoopBackOff   5
payment-api-9p4q    1/1     Running            0
postgres-0          0/1     Pending            0
```

**Interpretation:** The `payment-api-8k2m` container is not ready (`0/1`). Its `CrashLoopBackOff` status indicates repeated container failures with delayed restarts; five restarts have been recorded. The PostgreSQL Pod is also not ready and requires investigation.

### 2. Inspect logs from the failed container instance

**Action:** Retrieve logs from the previous payment API container instance.

```bash
kubectl logs payment-api-8k2m -n payments --previous
```

**Simulated output:**

```text
ERROR: Database connection refused
ERROR: Application startup aborted
Process exited with code 1
```

**Interpretation:** The application could not establish a database connection, aborted startup, and exited with code `1`. These logs do not establish the underlying cause of the connection refusal.

### 3. Check the Services

**Action:** Confirm that the expected Kubernetes Services exist.

```bash
kubectl get svc -n payments
```

**Simulated output:**

```text
NAME          TYPE        CLUSTER-IP      PORT(S)
payment-api   ClusterIP   10.96.12.40     8080/TCP
postgres      ClusterIP   10.96.18.25     5432/TCP
```

**Interpretation:** Both `payment-api` and `postgres` Services exist. A Service's presence does not prove that it has ready backends or that an application is healthy.

### 4. Inspect the PostgreSQL Service endpoints

**Action:** Check whether the `postgres` Service lists backend endpoints.

```bash
kubectl get endpoints postgres -n payments
```

**Simulated output:**

```text
NAME       ENDPOINTS   AGE
postgres   <none>      20m
```

**Interpretation:** The example Service has no listed backend endpoints and is approximately 20 minutes old. This alone does not identify the cause. For clusters using EndpointSlices, inspect the corresponding slices as well:

```bash
kubectl get endpointslices -n payments -l kubernetes.io/service-name=postgres
```

### 5. Check PostgreSQL Pod status

**Action:** Inspect the PostgreSQL Pod in the namespace.

```bash
kubectl get pods -n payments
```

**Simulated excerpt:**

```text
NAME         READY   STATUS    RESTARTS   AGE
postgres-0   0/1     Pending   0          20m
```

**Interpretation:** The `postgres-0` container is not ready (`0/1`), the Pod is `Pending`, and no container restarts have been recorded. A Pending Pod may not have been scheduled or started.

### 6. Examine scheduling events

**Action:** Describe the PostgreSQL Pod to inspect its scheduling events.

```bash
kubectl describe pod postgres-0 -n payments
```

**Simulated excerpt:**

```text
Name:       postgres-0
Namespace:  payments
Status:     Pending
Events:
  Type     Reason            Message
  ----     ------            -------
  Warning  FailedScheduling  0/2 nodes are available:
                             2 Insufficient memory.
```

**Interpretation:** The `FailedScheduling` event indicates that neither of the two Nodes has enough schedulable memory to accommodate the Pod's request.

### 7. Compare memory requests with Node allocations

**Action:** Review the Pod's memory request and each Node's allocatable memory and existing allocated requests.

```bash
kubectl describe pod postgres-0 -n payments
kubectl describe nodes
```

**Simulated, simplified output:**

```text
PostgreSQL Pod memory request: 2Gi

Node: worker-1
Allocatable memory: 4Gi
Allocated memory requests: 3Gi

Node: worker-2
Allocatable memory: 4Gi
Allocated memory requests: 3.5Gi
```

**Interpretation:** PostgreSQL requests `2Gi` of memory. The simplified figures leave `1Gi` of unallocated schedulable memory on `worker-1` and `0.5Gi` on `worker-2`. Neither can accommodate the request. Scheduling is based on resource requests and scheduling constraints, not merely observed free RAM.

## Proposed remediation

After reviewing workload requirements and obtaining any required change approvals, the engineer should add a Node with sufficient capacity or increase available capacity on an eligible existing Node. The team can also assess whether existing workload requests are appropriately sized. Do not arbitrarily reduce PostgreSQL's memory request: it may allow scheduling while leaving the database with insufficient runtime resources.

**No capacity change was actually performed in this exercise.** The following outputs are hypothetical examples of successful recovery after an approved capacity change.

## Verify recovery

### 8. Confirm that the Pods are ready

**Action:** Check Pod readiness after the hypothetical capacity change.

```bash
kubectl get pods -n payments
```

**Simulated output:**

```text
NAME                READY   STATUS    RESTARTS
payment-api-7d9x    1/1     Running   0
payment-api-8k2m    1/1     Running   5
payment-api-9p4q    1/1     Running   0
postgres-0          1/1     Running   0
```

**Interpretation:** Both `postgres-0` and `payment-api-8k2m` are running and ready (`1/1`). The payment API's restart count of five is historical; it does not necessarily mean failures are continuing. Pod readiness alone does not verify every application function.

### 9. Verify the application-to-database connection

**Action:** Inspect logs from the current payment API container instance.

```bash
kubectl logs payment-api-8k2m -n payments
```

**Simulated output:**

```text
INFO: Application started successfully
INFO: Connected to PostgreSQL database
INFO: Listening on port 8080
```

**Interpretation:** The logs report that the application started successfully, connected to PostgreSQL, and began listening on port `8080`. In a real environment, also test a representative application request or health check and confirm that errors do not recur.

## Summary

This simulated investigation traced a payment API `CrashLoopBackOff` symptom to a refused database connection, an unavailable PostgreSQL backend, and a PostgreSQL Pod that could not be scheduled because neither Node had sufficient unallocated schedulable memory. A capacity increase was proposed, and hypothetical recovery logs illustrated successful startup and database connectivity.

---

*Portfolio note: This is a guided technical-writing exercise. All outputs and recovery results are simulated, not evidence of a real incident or production remediation.*
