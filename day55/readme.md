# Day 55 – Kubernetes Persistent Volumes and PVCs

## What I Learned Today

Every Kubernetes day so far has been about *running* things.
Today was about *remembering* things — and it started by
proving how badly containers forget.

I created a simple Pod with an `emptyDir` volume, wrote a
timestamped message to a file, deleted the Pod, recreated it,
and checked the file again. New timestamp. Old data: gone.
That five-minute demo made the problem real in a way that no
amount of documentation could.

Then came Persistent Volumes and PersistentVolumeClaims, and
honestly the mental model clicked quickly once I stopped
thinking of them as one object and started thinking of them
as two sides of a contract.

The **PV** is the actual storage — a piece of disk, a cloud
volume, an NFS share. The **PVC** is a request: "I need
500Mi, read-write." Kubernetes plays matchmaker, finds a PV
that satisfies the request, and binds them together.

The proof that it works was the best part of the day. I wrote
data from Pod 1, deleted that pod entirely, spun up Pod 2
pointing at the same PVC, and read the file. Both entries
were there. Pod 1's data outlived Pod 1. That's the whole
point — and seeing it work directly is different from just
understanding it conceptually.

The static vs dynamic provisioning comparison was equally
useful. Static means an admin writes PV manifests by hand.
Dynamic means a StorageClass handles PV creation automatically
when a PVC asks for it. In kind, the default `standard`
StorageClass backed by `rancher.io/local-path` spun up a PV
the moment my Pod claimed one — I wrote zero PV YAML.

The cleanup task taught the reclaim policy difference better
than any explanation. Delete the PVC backed by `standard`
(ReclaimPolicy: Delete) → the PV vanishes automatically.
Delete the PVC backed by my manual PV (ReclaimPolicy: Retain)
→ the PV stays, moves to `Released`, and waits for an admin
to clean it up. One is fire-and-forget. The other keeps the
receipt.

## Tools Used

- Kubernetes (kind cluster)
- kubectl CLI
- BusyBox (for file read/write testing)
- YAML

## Key Concepts

- **emptyDir** — temporary volume; lives and dies with the Pod
- **PersistentVolume (PV)** — cluster-wide storage resource; provisioned by admin
- **PersistentVolumeClaim (PVC)** — namespaced storage request; made by developer
- **hostPath** — maps a node directory into a Pod; fine for learning, not production
- **Access modes** — RWO (one node, read-write), ROX (many nodes, read-only), RWX (many nodes, read-write)
- **Reclaim policies** — Retain (keep data after PVC delete) vs Delete (auto-remove)
- **StorageClass** — defines a provisioner; enables dynamic PV creation
- **Dynamic provisioning** — PVC triggers automatic PV creation via StorageClass

## Commands Used
````bash
# Apply manifests
kubectl apply -f ephemeral-pod.yaml
kubectl apply -f persistent-volume.yaml
kubectl apply -f persistent-volume-claim.yaml
kubectl apply -f pvc-pod.yaml
kubectl apply -f dynamic-pvc.yaml
kubectl apply -f dynamic-pod.yaml

# Inspect storage objects
kubectl get pv
kubectl get pvc
kubectl get storageclass
kubectl describe storageclass standard
kubectl describe pvc my-pvc

# Read/write data inside pods
kubectl exec pvc-pod -- cat /data/message.txt
kubectl exec pvc-pod-2 -- cat /data/message.txt
kubectl exec dynamic-pod -- cat /data/test.txt

# Cleanup
kubectl delete pod pvc-pod-2 dynamic-pod
kubectl delete pvc my-pvc dynamic-pvc
kubectl delete pv my-pv
````

## Challenges Faced

The `hostPath` directory issue caught me early. I created
`/tmp/k8s-pv-data` on my WSL host and the Pod kept getting
stuck in `ContainerCreating`. Turns out, kind nodes are
Docker containers — the path needs to exist *inside* that
container, not on the host machine. One `docker exec` command
fixed it.

The `WaitForFirstConsumer` binding mode on the dynamic PVC
also confused me briefly. The PVC sat in `Pending` even after
I applied it. It only bound once I created a Pod that
referenced it — that's by design, so Kubernetes can schedule
the Pod to the right node before provisioning the volume.

## Final Outcome

- ✅ Data loss demonstrated with `emptyDir` Pod
- ✅ PV created manually with `hostPath` and `Retain` policy
- ✅ PVC created and bound to PV — verified `Bound` status
- ✅ Data survived Pod deletion using PVC-backed volume
- ✅ File contained entries from both Pod 1 and Pod 2
- ✅ StorageClass inspected — default `standard` with `Delete` policy
- ✅ Dynamic PV provisioned automatically via `storageClassName: standard`
- ✅ Reclaim policies verified — Delete auto-removed PV, Retain kept it

Persistent storage is what makes Kubernetes viable for
real stateful applications — databases, message queues,
file stores. Tomorrow the focus moves to resource management:
CPU and memory requests and limits, so Pods don't starve
each other on shared nodes.

---
**#90DaysOfDevOps #Kubernetes #K8s #DevOps #CloudNative
#PersistentVolumes #Storage #kubectl #LearningInPublic**
````
````

---

### 📂 Suggested Commit
````bash
cd ~/github-actions-practice
git add 2026/day-55/
git commit -m "Day 55: Kubernetes PV and PVC - static provisioning, dynamic provisioning, reclaim policies"
git push origin main
````

---
