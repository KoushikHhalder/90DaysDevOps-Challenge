# Day 59 – Helm: Kubernetes Package Manager

## What I Learned Today

By Day 59, I had written a lot of Kubernetes YAML. Deployments,
Services, ConfigMaps, Secrets, PVCs — individual files for
everything, all wired together by hand. It works, but it
doesn't scale. A real application might have twenty of these
files. And deploying to three environments (dev, staging, prod)
means managing sixty files with slightly different values.

Today I learned **Helm** — and it immediately changed how I
think about Kubernetes packaging.

Helm has three concepts that clicked quickly once I stopped
thinking of it as just "YAML management." A **Chart** is a
package of manifest templates. A **Release** is one running
installation of that chart. A **Repository** is a collection
of charts you can search and pull from.

The first `helm install my-nginx bitnami/nginx` was a genuine
"oh wow" moment. One command — and Kubernetes had a Deployment,
Service, and ConfigMap, all correctly configured, all running.
`helm get manifest my-nginx` showed me exactly what was
applied. I could inspect it, version it, and recreate it
exactly.

The values system is where Helm earns its keep. Every chart
exposes a set of configurable knobs. `helm show values
bitnami/nginx` prints all of them with their defaults. I
overrode values inline with `--set`, then moved to a proper
`custom-values.yaml` file for repeatable installs. The file
approach is clearly the right habit for anything beyond
quick experiments.

Upgrade and rollback felt familiar after Day 52's Deployment
rollouts — but at the *full stack* level. Upgrading a release
bumps the revision. Rolling back adds a *new* revision that
restores the old config. The history is immutable and
auditable, which matters in production.

Creating my own chart with `helm create` was the deepest
part of the day. The scaffolded structure exposed Go
templating for the first time: `{{ .Values.replicaCount }}`,
`{{ .Release.Name }}`, `{{ .Chart.Name }}`. The template
files are just Kubernetes YAML with these placeholders
injected at deploy time. `helm template` renders everything
to stdout without touching the cluster — an incredibly useful
debugging tool. `helm lint` catches structural errors before
you even try to install.

## Tools Used

- Helm v3
- Bitnami chart repository
- kubectl CLI
- kind cluster
- Go templates (in chart authoring)

## Key Concepts

- **Chart** — package of Kubernetes manifest templates
- **Release** — a deployed instance of a chart in a cluster
- **Repository** — hosted collection of charts
- **values.yaml** — default configurable values for a chart
- **`--set`** — inline value override for quick changes
- **`-f values.yaml`** — file-based override for repeatable installs
- **`helm upgrade`** — updates a release, bumps revision number
- **`helm rollback`** — creates new revision restoring a previous config
- **Go templates** — `{{ .Values.x }}`, `{{ .Chart.Name }}`, `{{ .Release.Name }}`
- **`helm lint`** — validates chart structure before installing
- **`helm template`** — renders YAML locally without cluster interaction

## Commands Used

````bash
# Setup
helm version
helm env
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

# Discovery
helm search repo nginx
helm show values bitnami/nginx

# Install and inspect
helm install my-nginx bitnami/nginx
helm list
helm status my-nginx
helm get manifest my-nginx
helm get values my-nginx

# Custom values
helm install my-nginx-custom bitnami/nginx \
  --set replicaCount=3 --set service.type=NodePort
helm install my-nginx-file bitnami/nginx -f custom-values.yaml

# Upgrade and rollback
helm upgrade my-nginx bitnami/nginx --set replicaCount=5
helm history my-nginx
helm rollback my-nginx 1

# Custom chart
helm create my-app
helm lint my-app
helm template my-release ./my-app
helm install my-release ./my-app
helm upgrade my-release ./my-app --set replicaCount=5

# Cleanup
helm uninstall my-nginx my-nginx-custom my-nginx-file my-release
helm list
````

## Challenges Faced

The `helm upgrade` values gotcha caught me once: if you
don't pass `-f values.yaml` on an upgrade, Helm only uses
the values you explicitly provide — it doesn't remember your
previous file. So `helm upgrade my-release ./my-app` without
`-f` will reset any file-based overrides back to chart
defaults. Always pass the values file on every upgrade.

The Go template syntax in `_helpers.tpl` also took a few
minutes to understand. Functions like `{{ include
"my-app.fullname" . }}` pull from helper definitions in a
separate file — it's a DRY mechanism so the release name
is computed consistently across all templates.

## Final Outcome

- ✅ Helm v3 installed and verified
- ✅ Bitnami repo added, nginx chart deployed in one command
- ✅ Customized release using `--set` and `-f custom-values.yaml`
- ✅ Upgrade triggered, history inspected, rollback confirmed
- ✅ Rollback creates new revision — audit trail preserved
- ✅ Custom chart scaffolded with `helm create`
- ✅ Go template syntax explored and understood
- ✅ `helm lint` and `helm template` used for validation
- ✅ Chart installed and upgraded — 3 replicas → 5 replicas
- ✅ All releases cleaned up
