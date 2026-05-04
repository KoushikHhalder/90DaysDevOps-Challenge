# Day 60 – Capstone: WordPress + MySQL on Kubernetes

## What I Built Today

Ten days ago I didn't know what a Pod was.

Today I deployed a production-architecture WordPress + MySQL
stack from scratch using twelve Kubernetes concepts — all
working together in a single `capstone` namespace.

No step-by-step tutorials. No copy-paste. Just the YAML files
I've been writing since Day 51, combined into something real.

## Architecture

````
Browser → NodePort :30080
       → WordPress Deployment (2 replicas)
            ├── ConfigMap (DB host, DB name)
            ├── Secret (DB credentials)
            ├── Resource limits (200m CPU / 256Mi RAM)
            ├── Liveness + Readiness probes (/wp-login.php)
            └── HPA (min 2, max 10 replicas at 50% CPU)
                         ↓
            Headless Service (mysql)
                         ↓
            MySQL StatefulSet (mysql-0)
                 ├── Secret (root password, DB, user)
                 ├── Resource limits (250m CPU / 512Mi RAM)
                 └── PVC: 1Gi at /var/lib/mysql
````

Every box in that diagram is a YAML file I wrote by hand.

## What Made It Real

Three moments made this capstone click as more than an
exercise.

The first was watching WordPress connect to MySQL using the
StatefulSet DNS name I'd written:
`mysql-0.mysql.capstone.svc.cluster.local:3306`. That string
is only meaningful because of Day 56's lesson on Headless
Services and stable pod identity. Everything built on
everything else.

The second was the self-healing test. I deleted `mysql-0`
mid-session, watched it come back with the exact same name
and reconnect to the same PVC, and then refreshed the
browser. My blog post was still there. Persistent storage
across pod deletion — that's Day 55's lesson, proven.

The third was the Helm comparison at the end. One
`helm install` created more resources than I'd written
manually, with less control. That comparison made clear
*why* you learn YAML before Helm, not the other way around.

## Concepts Used

| Resource | Concept | Day |
|---|---|---|
| Namespace | Isolation | 52 |
| Secret | Sensitive config | 54 |
| ConfigMap | Non-sensitive config | 54 |
| PersistentVolumeClaim | Stateful storage | 55 |
| StatefulSet | Stable identity + storage | 56 |
| Headless Service | Per-pod DNS | 56 |
| Deployment | Replicated workload | 52 |
| NodePort Service | External access | 53 |
| Resource requests/limits | Scheduler + enforcement | 57 |
| Liveness + Readiness probes | Self-healing | 57 |
| HPA | Autoscaling | 58 |
| Helm comparison | Package management | 59 |

## Key Commands

````bash
# Namespace
kubectl create namespace capstone
kubectl config set-context --current --namespace=capstone

# Deploy MySQL stack
kubectl apply -f mysql-secret.yaml
kubectl apply -f mysql-headless-svc.yaml
kubectl apply -f mysql-statefulset.yaml
kubectl exec -it mysql-0 -- mysql -u wpuser -pwppassword123 -e "SHOW DATABASES;"

# Deploy WordPress stack
kubectl apply -f wordpress-configmap.yaml
kubectl apply -f wordpress-deployment.yaml
kubectl apply -f wordpress-service.yaml

# Access
kubectl port-forward svc/wordpress 8080:80 -n capstone

# HPA
kubectl apply -f wordpress-hpa.yaml
kubectl get hpa

# Full picture
kubectl get all -n capstone
kubectl get pvc -n capstone

# Cleanup
kubectl delete namespace capstone
kubectl config set-context --current --namespace=default
````

## What Was Hardest

Getting the MySQL connection string exactly right took a
couple of attempts. `WORDPRESS_DB_HOST` must follow the
full StatefulSet DNS pattern including namespace:
`mysql-0.mysql.capstone.svc.cluster.local:3306`. Leaving
out the namespace or the port breaks the connection silently
— WordPress just shows a database error page.

The readiness probe timing also required patience. WordPress
takes 60-90 seconds to bootstrap the first time. The temptation
is to delete the stuck pod and try again — but that just
resets the clock. `initialDelaySeconds: 30` on readiness
and `60` on liveness gave it the breathing room it needed.

## What I'd Add for Production

- **Ingress controller** with TLS termination instead of NodePort
- **Separate PVC for WordPress uploads** (`/var/www/html/wp-content`)
  so content survives pod restarts
- **MySQL replica** with a second StatefulSet pod for read scaling
- **Network Policies** to restrict which pods can talk to MySQL
- **Horizontal Pod Autoscaler on MySQL** (or Vertical Pod Autoscaler)
- **External Secrets Operator** to pull credentials from AWS
  Secrets Manager instead of storing in cluster Secrets

## Final Outcome

- ✅ WordPress + MySQL deployed end-to-end from scratch
- ✅ WordPress setup wizard completed, blog post created
- ✅ WordPress pod self-healing verified — Deployment recreated pod instantly
- ✅ MySQL pod self-healing verified — StatefulSet preserved name + storage
- ✅ Blog post survived both pod deletions — persistence confirmed
- ✅ HPA configured — min 2, max 10 replicas at 50% CPU
- ✅ Helm comparison completed — trade-offs understood
- ✅ Entire stack torn down with one `kubectl delete namespace`
