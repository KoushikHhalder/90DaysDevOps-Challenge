# Day 57 – Kubernetes Resource Requests, Limits, and Probes

## What I Learned Today

Up until today, my Pods had been running completely unsupervised.
No resource guardrails, no health checks, no way for Kubernetes
to know if an app was broken or overloading the node. Today I
fixed all of that — and it made the cluster feel genuinely
production-aware for the first time.

The first half was about resources. There are two numbers you
set per container: a **request** (what it needs) and a **limit**
(what it's allowed). The request is for the scheduler — it uses
this to decide which node has room for the Pod. The limit is for
the kubelet — it enforces this at runtime.

I learned the CPU/memory asymmetry the hard way by creating an
OOMKilled Pod on purpose. I gave a stress container a 100Mi
memory limit but told it to allocate 200M. It was killed in
under 4 seconds — exit code 137, which is `128 + SIGKILL`. CPU
limits throttle. Memory limits kill. That distinction matters.

I also created a Pod requesting 100 CPUs and 128Gi of RAM just
to watch it sit in `Pending` forever and read the scheduler's
message: "0/1 nodes are available: 1 Insufficient cpu." The
scheduler tells you exactly what's wrong if you bother to look
at `kubectl describe pod`.

The second half was about probes — and this is where Kubernetes
starts feeling like a real production platform.

**Liveness probes** restart stuck containers. I set one up on
a BusyBox pod that deleted its own health file after 30 seconds.
Three consecutive probe failures triggered a restart. The
`RESTARTS` counter went from 0 to 1 automatically. No manual
intervention.

**Readiness probes** were subtler and honestly more useful for
traffic management. I deleted the nginx index file, and within
15 seconds the pod went `0/1 READY` and disappeared from the
Service's endpoints. Traffic stopped reaching it. The container
itself was NOT restarted — it was just quietly taken out of
rotation until I fixed the file and readiness passed again.
That behavior is exactly what you want during a rolling update
or a DB reconnection.

**Startup probes** protect slow-starting containers. The key
insight: while the startup probe is active, liveness and
readiness are completely disabled. This means you can give a
slow app a 60-second budget to boot without Kubernetes killing
it for "not being alive yet." The math is simple:
`failureThreshold × periodSeconds = total startup budget`.

## Tools Used

- Kubernetes (kind cluster)
- kubectl CLI
- Nginx, BusyBox, polinux/stress
- YAML

## Key Concepts

- **Requests** — guaranteed minimum; used by scheduler for node placement
- **Limits** — enforced maximum; kubelet throttles CPU, kills on memory overuse
- **OOMKilled** — exit code 137; container exceeded memory limit
- **QoS Classes** — Guaranteed, Burstable, BestEffort (based on requests vs limits)
- **Liveness probe** — detects stuck containers; triggers restart on failure
- **Readiness probe** — detects unready containers; removes from endpoints, no restart
- **Startup probe** — protects slow-starting containers; disables liveness + readiness until startup passes
- **Probe types** — exec, httpGet, tcpSocket

## Commands Used

````bash
# Apply all manifests
kubectl apply -f resource-pod.yaml
kubectl apply -f oom-pod.yaml
kubectl apply -f impossible-pod.yaml
kubectl apply -f liveness-pod.yaml
kubectl apply -f readiness-pod.yaml
kubectl apply -f startup-pod.yaml

# Inspect resources and QoS
kubectl describe pod resource-pod

# Watch OOMKill happen
kubectl get pod oom-pod -w

# Scheduler error on impossible pod
kubectl describe pod impossible-pod

# Watch liveness restart
kubectl get pod liveness-pod -w

# Readiness probe test
kubectl expose pod readiness-pod --port=80 --name=readiness-svc
kubectl get endpoints readiness-svc
kubectl exec readiness-pod -- rm /usr/share/nginx/html/index.html
kubectl get pod readiness-pod
kubectl get endpoints readiness-svc

# Cleanup
kubectl delete pod resource-pod liveness-pod readiness-pod startup-pod
kubectl delete pod oom-pod impossible-pod
kubectl delete service readiness-svc
````

## Challenges Faced

The readiness vs liveness difference took a moment to fully
sink in. Both are health checks — but they do completely
different things on failure. Liveness says "this is broken,
restart it." Readiness says "this isn't ready, stop sending
it traffic." Getting the mental model right matters because
using liveness where you need readiness causes unnecessary
restarts; using readiness where you need liveness causessilent failures that never recover.

The startup probe failureThreshold math also tripped me up
conceptually at first. You have to think in terms of total
budget (`failureThreshold × periodSeconds`), not just the
number of failures. For a 20-second startup, a
`failureThreshold` of 2 with `periodSeconds` of 5 gives only
a 10-second budget — the pod would never start successfully.

## Final Outcome

- ✅ Resource requests and limits configured; QoS class verified
- ✅ OOMKilled demonstrated — exit code 137 confirmed
- ✅ Pending pod reproduced — scheduler's insufficient resource message read
- ✅ Liveness probe triggered restart after health file deletion
- ✅ Readiness probe removed pod from endpoints without restarting
- ✅ Pod restored to endpoints after file recovery
- ✅ Startup probe protected 20-second boot without liveness interference
- ✅ Understood failureThreshold × periodSeconds = startup budget


