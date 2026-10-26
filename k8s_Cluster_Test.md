# 🩺 Kubernetes Cluster Complete Test — Internal & External

> **Scope:** Cluster health · Nodes · System pods · DNS · Pod-to-Pod · Service · Traefik Ingress · External access
> **Excluded:** Keycloak · Spring Cloud Gateway
> **Ingress Controller:** Traefik
> **Prerequisites:** `kubectl` configured, cluster access, `curl`, `busybox`/`netshoot` image pullable

---

## 📑 Table of Contents

1. [Pre-flight Checks](#1-pre-flight-checks)
2. [Cluster & Control Plane Health](#2-cluster--control-plane-health)
3. [Node Health](#3-node-health)
4. [System Pods](#4-system-pods)
5. [DNS Resolution](#5-dns-resolution)
6. [Internal Networking (Pod-to-Pod)](#6-internal-networking-pod-to-pod)
7. [Service & Endpoints](#7-service--endpoints)
8. [Traefik Ingress — Internal](#8-traefik-ingress--internal)
9. [External Access via Traefik](#9-external-access-via-traefik)
10. [Resource Usage & Events](#10-resource-usage--events)
11. [Cleanup](#11-cleanup)
12. [One-Shot Automated Script](#12-one-shot-automated-script)
13. [Result Checklist](#13-result-checklist)

---

## 1. Pre-flight Checks

```bash
# Confirm kubectl context points to the correct cluster
kubectl config current-context
kubectl config view --minify -o jsonpath='{.clusters[0].name}{"\n"}'

# Verify API server is reachable
kubectl version --short
```

**Expected:** Context name matches your cluster; client & server versions print without error.

---

## 2. Cluster & Control Plane Health

```bash
# 2.1 Cluster info (API server, CoreDNS, etc.)
kubectl cluster-info

# 2.2 Verbose control plane readiness
kubectl get --raw='/readyz?verbose'

# 2.3 Livez (liveness of API server)
kubectl get --raw='/livez?verbose'
```

**Expected:**
- `cluster-info` → `Kubernetes control plane is running at ...`
- `readyz?verbose` → every component (`etcd`, `kube-scheduler`, `kube-controller-manager`, `kube-apiserver`) shows `ok`
- `livez?verbose` → all `ok`

**If failed:** `kubectl -n kube-system logs kube-apiserver-<node>` and `kubectl -n kube-system logs etcd-<node>`

---

## 3. Node Health

```bash
# 3.1 List nodes
kubectl get nodes -o wide

# 3.2 Node conditions (MemoryPressure, DiskPressure, PIDPressure, Ready)
kubectl get nodes -o jsonpath='{range .items[*]}{.metadata.name}{"\t"}{range .status.conditions[*]}{.type}={.status} {end}{"\n"}{end}'

# 3.3 Node resource summary (requires metrics-server)
kubectl top nodes
```

**Expected:**
- All nodes `STATUS=Ready`
- Conditions: `Ready=True`, `MemoryPressure=False`, `DiskPressure=False`, `PIDPressure=False`
- `top` returns CPU/mem (if metrics-server absent, this step is optional)

**If NotReady:**
```bash
kubectl describe node <node-name> | tail -40
```

---

## 4. System Pods

```bash
# 4.1 All kube-system pods
kubectl get pods -n kube-system -o wide

# 4.2 Any non-Running pods across all namespaces
kubectl get pods -A --field-selector=status.phase!=Running,status.phase!=Succeeded

# 4.3 Restart counts > 5 (potential instability)
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.status.containerStatuses[0].restartCount}{"\n"}{end}' | awk '$2>5'
```

**Expected:**
- `coredns`, `kube-proxy`, `kube-apiserver`, `kube-scheduler`, `kube-controller-manager`, `etcd`, and CNI pods → `Running`
- 4.2 → empty
- 4.3 → empty (or investigate named pods)

**If a pod is unhealthy:**
```bash
kubectl describe pod <pod> -n kube-system
kubectl logs <pod> -n kube-system --tail=100
```

---

## 5. DNS Resolution

```bash
# 5.1 CoreDNS service exists
kubectl -n kube-system get svc -l k8s-app=kube-dns

# 5.2 Deploy a short-lived DNS test pod
kubectl run dnstest --image=busybox:1.28 --restart=Never --command -- sleep 600
kubectl wait --for=condition=Ready pod/dnstest --timeout=60s

# 5.3 Resolve internal service names
kubectl exec dnstest -- nslookup kubernetes.default
kubectl exec dnstest -- nslookup kube-dns.kube-system.svc.cluster.local

# 5.4 External DNS resolution from inside the cluster
kubectl exec dnstest -- nslookup google.com
```

**Expected:**
- `kubernetes.default` resolves to `10.96.0.1` (or your service CIDR)
- `kube-dns.kube-system` resolves to a ClusterIP
- `google.com` resolves (validates upstream DNS)

**If failed:** check `kubectl -n kube-system logs -l k8s-app=kube-dns` and CoreDNS ConfigMap.

---

## 6. Internal Networking (Pod-to-Pod)

### 6.1 Setup test pods

```bash
# Pod A — target (nginx)
kubectl run pod-a --image=nginx:alpine --port=80
kubectl wait --for=condition=Ready pod/pod-a --timeout=90s

# Pod B — client (netshoot: includes curl, ping, dig, tcpdump)
kubectl run pod-b --image=nicolaka/netshoot --command -- sleep 900
kubectl wait --for=condition=Ready pod/pod-b --timeout=90s
```

### 6.2 Same-node pod-to-pod ping

```bash
POD_A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}')
POD_A_NODE=$(kubectl get pod pod-a -o jsonpath='{.spec.nodeName}')
echo "Pod A → IP: $POD_A_IP, Node: $POD_A_NODE"

# Force pod-b onto the same node as pod-a
kubectl delete pod pod-b --grace-period=0 --force
kubectl run pod-b --image=nicolaka/netshoot --restart=Never --overrides="{\"spec\":{\"nodeName\":\"$POD_A_NODE\"}}" --command -- sleep 900
kubectl wait --for=condition=Ready pod/pod-b --timeout=90s

kubectl exec pod-b -- ping -c 3 $POD_A_IP
```

**Expected:** `3 packets transmitted, 3 received, 0% packet loss`

### 6.3 Cross-node pod-to-pod ping

```bash
# Get a different worker node
OTHER_NODE=$(kubectl get nodes -o jsonpath='{.items[*].metadata.name}' | tr ' ' '\n' | grep -v "$POD_A_NODE" | head -1)
echo "Other node: $OTHER_NODE"

kubectl run pod-c --image=nginx:alpine --overrides="{\"spec\":{\"nodeName\":\"$OTHER_NODE\"}}"
kubectl wait --for=condition=Ready pod/pod-c --timeout=90s
POD_C_IP=$(kubectl get pod pod-c -o jsonpath='{.status.podIP}')

kubectl exec pod-b -- ping -c 3 $POD_C_IP
```

**Expected:** `0% packet loss` (validates CNI cross-node routing)

### 6.4 Pod-to-pod HTTP (sanity)

```bash
kubectl exec pod-b -- curl -sS -o /dev/null -w "HTTP %{http_code} in %{time_total}s\n" http://$POD_A_IP
```

**Expected:** `HTTP 200 in ...`

---

## 7. Service & Endpoints

```bash
# 7.1 Create a ClusterIP Service for pod-a
kubectl expose pod pod-a --port=80 --target-port=80 --name=pod-a-svc

# 7.2 Verify Service gets a ClusterIP
kubectl get svc pod-a-svc -o wide

# 7.3 Verify Endpoints populated
kubectl get endpoints pod-a-svc

# 7.4 DNS resolve the service
kubectl exec pod-b -- nslookup pod-a-svc
kubectl exec pod-b -- nslookup pod-a-svc.default.svc.cluster.local

# 7.5 HTTP through the Service (kube-proxy path)
kubectl exec pod-b -- curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://pod-a-svc
kubectl exec pod-b -- curl -sS -o /dev/null -w "HTTP %{http_code}\n" http://pod-a-svc.default.svc.cluster.local
```

**Expected:**
- `EXTERNAL-IP=<none>`, `CLUSTER-IP` assigned
- `ENDPOINTS` → `10.244.x.y:80`
- Both `curl` return `HTTP 200`

**If endpoints empty:** selector mismatch — `kubectl describe svc pod-a-svc`

---

## 8. Traefik Ingress — Internal

> Assumes Traefik installed in namespace `traefik`. Adjust if different.

```bash
# 8.1 Traefik pods are Running and Ready
kubectl get pods -n traefik -o wide

# 8.2 Traefik services
kubectl get svc -n traefik

# 8.3 Traefik CRDs are registered
kubectl get crd | grep traefik

# 8.4 Configured IngressRoutes & Middlewares
kubectl get ingressroute -A
kubectl get middleware -A
kubectl get tlsstore -A

# 8.5 Traefik health endpoint (requires ping enabled in config)
kubectl port-forward -n traefik svc/traefik 8080:8080 &
sleep 2
curl -sf http://localhost:8080/ping && echo " Traefik healthy"
curl -s http://localhost:8080/metrics | head -5   # if metrics enabled
kill %1 2>/dev/null

# 8.6 Traefik access logs (last 50 lines)
kubectl logs -n traefik -l app.kubernetes.io/name=traefik --tail=50
```

**Expected:**
- Traefik pods `1/1 Running`
- Services: `web` (80) and `websecure` (443) exposed (LoadBalancer/NodePort/ClusterIP per setup)
- CRDs include `ingressroutes.traefik.io`, `middlewares.traefik.io`
- `/ping` returns `OK`
- Access log shows recent requests (or no errors)

**If `/ping` 404:** add `--ping=true` to Traefik args (or in Helm values `ping.enabled: true`).

---

## 9. External Access via Traefik

### 9.1 Discover the external entry point

```bash
# 9.1a If LoadBalancer
TRAEFIK_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}')
echo "Traefik external IP: $TRAEFIK_IP"

# 9.1b If NodePort (fallback)
NODE_PORT=$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="web")].nodePort}')
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="ExternalIP")].address}')
echo "Node IP: $NODE_IP, NodePort: $NODE_PORT"
```

### 9.2 Confirm a test IngressRoute exists

```bash
kubectl get ingressroute -A -o wide
# Identify the host, e.g. test.example.com or api.example.com
```

### 9.3 Internal curl via the IngressRoute Host header

```bash
# Using the Traefik external IP with the correct Host header
curl -vI -H "Host: <your-test-host>" http://$TRAEFIK_IP/

# Expected: HTTP/1.1 200 OK (or your backend's response)
```

### 9.4 External curl (if DNS is configured)

```bash
curl -vI https://<your-test-host>/
# Expected: HTTP/2 200 and valid TLS certificate
```

### 9.5 Common status codes

| Code | Meaning | Next step |
|---|---|---|
| `200/3xx` | Working | — |
| `404` | Route not matched | Check `IngressRoute` match rule & host |
| `502` | Backend unreachable | Check Service port & endpoints |
| `503` | No healthy backends | Check pod readiness |
| `504` | Backend timeout | Check NetworkPolicy / firewall |

### 9.6 Traefik router & service debug

```bash
# Inspect the Traefik API (requires --api.insecure or dashboard enabled)
kubectl port-forward -n traefik svc/traefik 9000:9000 &
sleep 2
curl -s http://localhost:9000/api/http/routers | jq .
curl -s http://localhost:9000/api/http/services | jq .
kill %1 2>/dev/null
```

**Expected:** your test route appears with `status=enabled`.

---

## 10. Resource Usage & Events

```bash
# 10.1 Top consumers
kubectl top nodes
kubectl top pods -A --sort-by=cpu | head -20
kubectl top pods -A --sort-by=memory | head -20

# 10.2 Recent Warning/Error events
kubectl get events -A --sort-by='.lastTimestamp' | grep -Ei 'warning|error|failed' | tail -30

# 10.3 Pods with image pull issues
kubectl get pods -A -o jsonpath='{range .items[*]}{.metadata.namespace}/{.metadata.name}{"\t"}{.status.containerStatuses[0].state.waiting.reason}{"\n"}{end}' | grep -Ei 'ErrImagePull|ImagePullBackOff'
```

**Expected:** no `Warning`/`Error` events; CPU/memory within node capacity.

---

## 11. Cleanup

```bash
kubectl delete pod dnstest pod-a pod-b pod-c --ignore-not-found --grace-period=0 --force
kubectl delete svc pod-a-svc --ignore-not-found
```

Confirm cleanup:

```bash
kubectl get pods | grep -E 'pod-a|pod-b|pod-c|dnstest' || echo "Cleanup complete"
```

---

## 12. One-Shot Automated Script

> Save this block as `run-cluster-test.sh` and execute. Produces a colour-coded PASS/FAIL summary.

```bash
#!/usr/bin/env bash
# run-cluster-test.sh — complete internal + external cluster test
set -uo pipefail

PASS=0; FAIL=0
ok()   { echo -e "  \033[32m✔ PASS\033[0m  $1"; PASS=$((PASS+1)); }
bad()  { echo -e "  \033[31m✘ FAIL\033[0m  $1"; FAIL=$((FAIL+1)); }
hdr()  { echo -e "\n\033[1;36m▶ $1\033[0m"; }

# 1. Cluster info
hdr "1. Cluster & Control Plane"
if kubectl cluster-info >/dev/null 2>&1; then ok "cluster-info"; else bad "cluster-info"; fi
if kubectl get --raw='/readyz' 2>/dev/null | grep -q ok; then ok "readyz"; else bad "readyz"; fi

# 2. Nodes
hdr "2. Nodes"
NOT_READY=$(kubectl get nodes --no-headers | awk '$2!="Ready"' | wc -l | tr -d ' ')
[[ "$NOT_READY" == "0" ]] && ok "all nodes Ready" || bad "$NOT_READY node(s) NotReady"

# 3. System pods
hdr "3. System Pods"
BAD_SYS=$(kubectl -n kube-system get pods --no-headers | awk '$3!="Running" && $3!="Completed"' | wc -l | tr -d ' ')
[[ "$BAD_SYS" == "0" ]] && ok "kube-system healthy" || bad "$BAD_SYS unhealthy pod(s) in kube-system"

# 4. DNS
hdr "4. DNS Resolution"
kubectl run dnstest --image=busybox:1.28 --restart=Never --command -- sleep 300 >/dev/null 2>&1
kubectl wait --for=condition=Ready pod/dnstest --timeout=60s >/dev/null 2>&1
if kubectl exec dnstest -- nslookup kubernetes.default 2>/dev/null | grep -q Address; then ok "kubernetes.default resolves"; else bad "DNS resolution"; fi
if kubectl exec dnstest -- nslookup google.com 2>/dev/null | grep -q Address; then ok "external DNS"; else bad "external DNS"; fi

# 5. Pod-to-Pod
hdr "5. Pod-to-Pod Networking"
kubectl run pod-a --image=nginx:alpine --port=80 >/dev/null 2>&1
kubectl run pod-b --image=nicolaka/netshoot --command -- sleep 300 >/dev/null 2>&1
kubectl wait --for=condition=Ready pod/pod-a --timeout=90s >/dev/null 2>&1
kubectl wait --for=condition=Ready pod/pod-b --timeout=90s >/dev/null 2>&1
A_IP=$(kubectl get pod pod-a -o jsonpath='{.status.podIP}' 2>/dev/null)
if kubectl exec pod-b -- ping -c 3 "$A_IP" 2>/dev/null | grep -q '0% packet loss'; then ok "pod-to-pod ping"; else bad "pod-to-pod ping"; fi

# 6. Service
hdr "6. Service & Endpoints"
kubectl expose pod pod-a --port=80 --name=pod-a-svc >/dev/null 2>&1
EP=$(kubectl get endpoints pod-a-svc -o jsonpath='{.subsets[0].addresses[0].ip}' 2>/dev/null)
[[ -n "$EP" ]] && ok "endpoints populated ($EP)" || bad "endpoints empty"
if kubectl exec pod-b -- curl -s -o /dev/null -w "%{http_code}" http://pod-a-svc 2>/dev/null | grep -q 200; then ok "service HTTP 200"; else bad "service HTTP"; fi

# 7. Traefik
hdr "7. Traefik Ingress"
TR_PODS=$(kubectl -n traefik get pods --no-headers 2>/dev/null | awk '$3=="Running"' | wc -l | tr -d ' ')
[[ "$TR_PODS" -gt 0 ]] && ok "traefik pod running" || bad "traefik pod not running"
if kubectl get crd 2>/dev/null | grep -q ingressroutes.traefik.io; then ok "traefik CRDs present"; else bad "traefik CRDs missing"; fi
if kubectl get ingressroute -A --no-headers 2>/dev/null | grep -q .; then ok "ingressroutes configured"; else bad "no ingressroutes"; fi

# 8. External
hdr "8. External Access"
TR_IP=$(kubectl -n traefik get svc traefik -o jsonpath='{.status.loadBalancer.ingress[0].ip}' 2>/dev/null)
if [[ -n "$TR_IP" ]]; then
  ok "traefik LB IP: $TR_IP"
else
  NP=$(kubectl -n traefik get svc traefik -o jsonpath='{.spec.ports[?(@.name=="web")].nodePort}' 2>/dev/null)
  [[ -n "$NP" ]] && ok "traefik NodePort: $NP" || bad "no external entry point"
fi

# 9. Cleanup
hdr "9. Cleanup"
kubectl delete pod dnstest pod-a pod-b --ignore-not-found --grace-period=0 --force >/dev/null 2>&1
kubectl delete svc pod-a-svc --ignore-not-found >/dev/null 2>&1
ok "resources cleaned up"

# Summary
echo -e "\n\033[1;37m═══════════════════════════════════════\033[0m"
echo -e "  \033[32mPASS: $PASS\033[0m   \033[31mFAIL: $FAIL\033[0m"
echo -e "\033[1;37m═══════════════════════════════════════\033[0m"
exit $FAIL
```

Run it:

```bash
chmod +x run-cluster-test.sh
./run-cluster-test.sh
```

---

## 13. Result Checklist

| # | Test | Command | Expected | Status |
|---|---|---|---|---|
| 1 | Context | `kubectl config current-context` | correct cluster | ☐ |
| 2 | Cluster info | `kubectl cluster-info` | control plane running | ☐ |
| 3 | readyz | `kubectl get --raw='/readyz?verbose'` | all `ok` | ☐ |
| 4 | Nodes | `kubectl get nodes` | all `Ready` | ☐ |
| 5 | System pods | `kubectl get pods -n kube-system` | all `Running` | ☐ |
| 6 | Internal DNS | `nslookup kubernetes.default` | Address returned | ☐ |
| 7 | External DNS | `nslookup google.com` | Address returned | ☐ |
| 8 | Pod-to-pod | `ping $POD_IP` | 0% loss | ☐ |
| 9 | Service endpoints | `kubectl get endpoints` | pod IP listed | ☐ |
| 10 | Service HTTP | `curl http://<svc>` | 200 | ☐ |
| 11 | Traefik pods | `kubectl get pods -n traefik` | 1/1 Running | ☐ |
| 12 | Traefik CRDs | `kubectl get crd \| grep traefik` | present | ☐ |
| 13 | IngressRoutes | `kubectl get ingressroute -A` | configured | ☐ |
| 14 | Traefik `/ping` | `curl :8080/ping` | `OK` | ☐ |
| 15 | External 200 | `curl -H "Host: ..." $TRAEFIK_IP` | 200 | ☐ |
| 16 | Events | `kubectl get events -A` | no Warning/Error | ☐ |

---

## Troubleshooting Quick Reference

| Symptom | First diagnostic command |
|---|---|
| Node NotReady | `kubectl describe node <n>` |
| Pod CrashLoopBackOff | `kubectl logs <pod> --previous` |
| DNS failure | `kubectl -n kube-system logs -l k8s-app=kube-dns` |
| Service unreachable | `kubectl get endpoints <svc>` |
| Traefik 404 | `curl :9000/api/http/routers \| jq` |
| Traefik 502 | `kubectl get endpoints <backend-svc>` |
| External NXDOMAIN | `dig +short <host> @8.8.8.8` |

---

**End of runbook.**
