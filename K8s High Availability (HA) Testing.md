# K8s High Availability (HA) Testing Guide

This document provides a step-by-step guide to test the resilience, self-healing, and high availability of the `product-service` running on the Kubernetes cluster.

## Prerequisites
- `kubectl` configured to access the cluster.
- `product-service` deployed in the `dev` namespace.
- HPA (Horizontal Pod Autoscaler) configured.
- 2 Worker Nodes (`node4-vm-01`, `node4-vm-02`) available.

---

##  Test 1: Pod Self-Healing (Auto-Restart)

**Goal:** Verify that if a pod crashes or is deleted, Kubernetes automatically creates a new one to maintain the desired state (Replicas).

1. **Check current pods:**
   ```bash
   kubectl -n dev get pods
   ```
   *Note the name of the running `product-service` pod.*

2. **Delete the pod manually (simulate crash):**
   ```bash
   kubectl -n dev delete pod <product-service-pod-name>
   ```

3. **Watch the recovery:**
   ```bash
   kubectl -n dev get pods -w
   ```
   **Expected Result:** The old pod goes into `Terminating` state, and immediately a new pod is created with a new name and goes into `Running` state.

---

##  Test 2: Rolling Update (Zero Downtime)

**Goal:** Verify that the application remains available to users while a new version is being deployed.

1. **Start a continuous ping in a separate terminal:**
   ```bash
   # This will ping the health endpoint every 0.5 seconds
   while true; do curl -s -o /dev/null -w "%{http_code}\n" http://10.70.16.237:30080/actuator/health; sleep 0.5; done
   ```

2. **Trigger a rolling update (simulate new code deployment):**
   In another terminal, update an environment variable to force a rollout:
   ```bash
   kubectl -n dev set env deployment/product-service ROLLOUT_TEST="v2"
   ```

3. **Watch the pods update:**
   ```bash
   kubectl -n dev get pods -w
   ```
   **Expected Result:** 
   - A new pod is created and reaches `1/1 Running`.
   - Only *after* the new pod is ready, the old pod is terminated.
   - In the first terminal, you should see `200` responses continuously. **Zero downtime!**


**After test you must delete environment variable:**
   ```bash
   kubectl -n dev set env deployment/product-service ROLLOUT_TEST-
   ```


---

## Test 3: Horizontal Pod Autoscaling (HPA)

**Goal:** Verify that the application scales out (up to 4 pods) under heavy load and scales in when load drops.

1. **Verify HPA is ready:**
   ```bash
   kubectl -n dev get hpa
   # TARGETS should show something like "cpu: 1%/50%"
   ```

2. **Deploy a Load Generator:**
   ```bash
   kubectl -n dev run load-generator --image=busybox:1.28 --restart=Never -- /bin/sh -c "while true; do wget -q -O- http://product-service:8081/api/v1/products > /dev/null; done"
   ```

3. **Watch the CPU and Replicas increase:**
   ```bash
   kubectl -n dev get hpa -w
   # Wait 1-2 minutes. REPLICAS should go from 1 -> 2 -> 3 -> 4 (Max)
   kubectl -n dev get pods -w
   # You will see new pods being scheduled.
   ```

4. **Stop the load and watch scale-down (Cool-down):**
   ```bash
   kubectl -n dev delete pod load-generator
   ```
   **Expected Result:** After ~5 minutes (default cool-down period), the CPU drops, and HPA scales the replicas back down to 1.

---

## Test 4: Node Failure / Maintenance (Pod Rescheduling)

**Goal:** Verify that if an entire Worker Node goes down, Kubernetes reschedules the pods to another healthy node.

1. **Check which node your pod is running on:**
   ```bash
   kubectl -n dev get pods -o wide
   # Look at the NODE column (e.g., node4-vm-01)
   ```

2. **Simulate Node Maintenance (Drain the node):**
   ```bash
   kubectl drain <node-name> --ignore-daemonsets --delete-emptydir-data
   ```
   *This safely evicts all pods from that node.*

3. **Watch the pod reschedule to another node:**
   ```bash
   kubectl -n dev get pods -o wide -w
   ```
   **Expected Result:** The old pod is terminated, and a new pod is created on the *other* Worker Node (`node4-vm-02`).

4. **Bring the node back online:**
   ```bash
   kubectl uncordon <node-name>
   ```

---

## Test 5: Infrastructure High Availability (DB & Redis)

**Goal:** Verify the application can survive database or cache failures.

### 5.1 Redis Sentinel Failover
1. Go to VM `.236` (Redis Cluster).
2. Stop the current Redis Master: `docker stop <redis-master-container-name>`
3. Check logs in K8s: `kubectl -n dev logs -l app=product-service --tail=20`
4. **Expected Result:** Spring Boot should throw some Redis connection errors briefly, but Sentinel will promote a replica to master within 5-10 seconds. The app will recover automatically without crashing.

### 5.2 PostgreSQL HA
1. Stop the Primary PostgreSQL node.
2. **Expected Result:** Your app will fail to write to the DB temporarily. *(Note: If you have PG Pool or Patroni set up, it will failover automatically; otherwise, you will need to manually promote a replica).*

---
## Summary Checklist

| Test Scenario | Command / Action | Status |
| :--- | :--- | :---: |
| **Pod Healing** | `kubectl delete pod` | ⬜ |
| **Zero Downtime** | `kubectl set env` + continuous `curl` | ⬜ |
| **Auto-scaling** | Run `load-generator` pod | ⬜ |
| **Node Failure** | `kubectl drain` node | ⬜ |
| **Redis Failover** | Stop Redis master container | ⬜ |
