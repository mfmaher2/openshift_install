# Mission Control Air‑Gapped Install on OpenShift

This guide uses an in‑cluster Nexus (“mock repository”) to host both container images (Docker hosted) and Helm charts (Helm hosted), then installs Mission Control entirely from that Nexus. Observability (Loki/Mimir/Grafana) is disabled initially to avoid object‑store prerequisites; you can enable it later.

## Prerequisites

- OpenShift cluster admin (oc CLI configured)
- Helm v3.12+ on your workstation
- A workstation with temporary internet access for mirroring (to pull once, then push to Nexus)
- Files you will create in this guide:
  - values-defaults.yaml (or values-full.yaml)
  - dex-reset.yaml (contains bcrypt hash; see Appendix)
  - disable-observability.yaml
  - airgap-images.yaml
  - cert-manager.crds.yaml (downloaded once)
  - certmgr-images.yaml (optional override for cert-manager)

---

## Deploy an in‑cluster Nexus (UI + Docker registry)

Apply Nexus resources (Namespace, SA, PVC, Deployment, Services, Route):

```bash
oc apply -f nexus.yaml

# grant SCC so Nexus can write to /nexus-data:
oc adm policy add-scc-to-user anyuid -z nexus-sa -n nexus

```
What each piece does

- Namespace nexus: isolates the Nexus deployment.
- ServiceAccount nexus-sa: the pod runs under this SA.
- PVC nexus-data: persistent storage for Nexus (/nexus-data).
- Deployment nexus: runs the Nexus 3 container, exposes 8081 (UI) and 5000 (Docker registry).
- Service nexus: exposes UI inside cluster.
- Service nexus-docker: exposes registry inside cluster on port 5000.
- Route nexus: exposes the UI externally via OpenShift router (HTTPS edge‑terminated).

Verify and get admin password
```bash
oc get pods -n nexus -w
NEXUS_HOST=$(oc get route nexus -n nexus -o jsonpath='{.spec.host}')
echo "Nexus UI: https://${NEXUS_HOST}"
POD=$(oc get pod -n nexus -l app=nexus -o jsonpath='{.items[0].metadata.name}')
oc exec -n nexus "$POD" -- cat /nexus-data/admin.password
# Username: admin
# Password: (value printed above)
```

Allow write access to /nexus-data (OCP SCC)
```bash
oc adm policy add-scc-to-user anyuid -z nexus-sa -n nexus
```

Wait for pod to come up
```bash
oc get pods -n nexus -w
```
## Configure Nexus repositories (UI)
Log in to the Nexus UI with the admin password you retrieved:

URL:
```bash
NEXUS_HOST=$(oc get route nexus -n nexus -o jsonpath='{.spec.host}')
echo "https://${NEXUS_HOST}"
```
Username: admin
Password: (value printed from /nexus-data/admin.password)

Create repositories
- Repository → Repositories → Create repository:
  - docker (hosted): Name docker-local, HTTP port 5000 → Save
  - helm (hosted): Name helm-local → Save
  - 
In‑cluster endpoints you’ll use later:
  - Docker registry: nexus-docker.nexus.svc.cluster.local:5000
  - Helm repo:  http://nexus.nexus.svc.cluster.local:8081/repository/helm-local
