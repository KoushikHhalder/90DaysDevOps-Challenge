# Day 58 – Kubernetes Metrics Server and Horizontal Pod Autoscaler

## What I Learned Today

Day 57 was about setting resource limits. Today was about
making the cluster *react* to those resources dynamically —
and honestly this felt like the day Kubernetes started
resembling a real production platform.

The first piece was **Metrics Server**. It's a lightweight
add-on that collects actual CPU and memory usage from every
node and pod every 15 seconds. Without it, `kubectl top`
returns nothing and HPA has no data to work with. On kind,
there's a small gotcha: kind nodes use self-signed TLS
certificates, so you have to patch the Metrics Server
manifest with `--kubelet-insecure-tls`. That flag is
fine locally — never in production.

Once Metrics Server was running, `kubectl top nodes` gave
me actual live CPU and memory numbers. The distinction
between `kubectl top` (real usage) and `kubectl describe pod`
(configured requests/limits) is worth remembering — they
measure completely different things and both are useful.

Then came the **Horizontal Pod Autoscaler**, and the load
test was the most satisfying demo I've done in this challenge.
I deployed a CPU-intensive PHP app with `requests.cpu: 200m`,
set an HPA to scale when average CPU exceeded 50% of that
request, then ran a BusyBox loop hammering the service with
`wget` requests.

Watching the HPA output live: CPU climbed to 112%, replicas
jumped from 1 to 4 automatically over about 90 seconds. The
formula is elegant — `desiredReplicas = ceil(currentReplicas
× currentUsage / targetUsage)` — pure arithmetic driving
infrastructure decisions in real time.

When I stopped the load generator, the scale-down was
deliberately slow — a 5-minute stabilization window stops
Kubernetes from thrashing replicas up and down on bursty
traffic. That's sensible behavior by default, but the
`autoscaling/v2` YAML API lets you tune it precisely.

Writing the v2 HPA manifest was the other highlight. The
imperative `kubectl autoscale` command only does CPU. The
`autoscaling/v2` API adds memory metrics, custom metrics,
and a `behavior` block where you can define exactly how
fast to scale up (aggressive — add 2 pods per 15s) and
how cautious to be scaling down (conservative — drop 1
pod per minute after a 5-minute window). That kind of
control is what real production tuning looks like.

## Tools Used

- Kubernetes (kind cluster)
- Metrics Server
- kubectl CLI
- HPA (autoscaling/v1 imperative + autoscaling/v2 declarative)
- registry.k8s.io/hpa-example (CPU load demo app)
- BusyBox (load generator)

## Key Concepts

- **Metrics Server** — collects live CPU/memory from kubelets; required for HPA and `kubectl top`
- **kubectl top** — actual real-time usage (not requests/limits)
- **HPA** — adjusts replica count automatically based on metrics
- **CPU utilization %** — `actual_cpu / requested_cpu × 100` (why requests are mandatory)
- **Scale-up** — fast by default; triggered when usage exceeds target
- **Scale-down** — slow by default (5-min stabilization window); prevents thrash
- **autoscaling/v1** — CPU only, imperative
- **autoscaling/v2** — CPU + memory + custom metrics + behavior tuning, declarative
- **HPA formula** — `ceil(currentReplicas × (currentUsage / targetUsage))`

## Commands Used

````bash
# Metrics Server
kubectl apply -f metrics-server.yaml
kubectl top nodes
kubectl top pods -A
kubectl top pods -A --sort-by=cpu

# Deployment + Service
kubectl apply -f php-apache.yaml
kubectl expose deployment php-apache --port=80

# HPA - imperative
kubectl autoscale deployment php-apache --cpu-percent=50 --min=1 --max=10
kubectl get hpa
kubectl describe hpa php-apache
kubectl get hpa php-apache --watch

# Load generator
kubectl run load-generator --image=busybox:1.36 --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache; done"

# HPA - declarative v2
kubectl delete hpa php-apache
kubectl apply -f hpa-v2.yaml
kubectl describe hpa php-apache

# Cleanup
kubectl delete hpa php-apache
kubectl delete service php-apache
kubectl delete deployment php-apache
kubectl delete pod load-generator
````

## Challenges Faced

The `--kubelet-insecure-tls` patch was the first hurdle.
The Metrics Server would install fine but stay in a
`CrashLoopBackOff` without it on kind. Once I understood
*why* — kind nodes use self-signed certs that the Metrics
Server rejects by default — the fix made sense.

The `<unknown>` HPA target also briefly confused me before
I remembered: HPA computes utilization as a percentage of
`requests.cpu`. No requests = no denominator = no percentage.
Once I added `resources.requests.cpu: 200m` to the Deployment,
the HPA populated within 30 seconds.

## Final Outcome

- ✅ Metrics Server installed on kind with insecure TLS patch
- ✅ `kubectl top nodes` and `kubectl top pods` returning live data
- ✅ php-apache Deployment deployed with `requests.cpu: 200m`
- ✅ HPA created imperatively — targets populated, `1%/50%` at idle
- ✅ Load generator triggered scale-up — watched replicas increase live
- ✅ Understood 5-minute scale-down stabilization window
- ✅ autoscaling/v2 HPA written with custom scale-up and scale-down behavior
- ✅ Behavior section verified in `kubectl describe hpa`

