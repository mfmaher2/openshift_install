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
    
In‑cluster endpoints you’ll use later:
  - Docker registry: nexus-docker.nexus.svc.cluster.local:5000
  - Helm repo:  http://nexus.nexus.svc.cluster.local:8081/repository/helm-local

## Mirror the Mission Control chart and images to Nexus
Pull chart and render (observability disabled for templating)
```bash
# Pull Mission Control chart
helm pull oci://registry.replicated.com/mission-control/mission-control --version 1.15.0

# Render manifests to harvest images
helm template mission-control oci://registry.replicated.com/mission-control/mission-control \
  --version 1.15.0 \
  -n mission-control \
  -f disable-observability.yaml \
  > rendered.yaml
```
Extract all container image references
```bash
yq e -r '.. | .image? | select(.)' rendered.yaml | sort -u > images.txt
```
Port‑forward Nexus Docker service and mirror images into Nexus
```bash
# Port-forward the in-cluster Nexus registry (leave running)
oc -n nexus port-forward svc/nexus-docker 5000:5000 >/tmp/nexus-registry-pf.log 2>&1 &

# Mirror all images; strip the source registry from the path
# so final refs look like: nexus:5000/<repo>/<image>:<tag>
while read -r IMG; do
  REPO_TAG=$(echo "$IMG" | sed -E 's#^[^/]+/##')   # drop leading registry
  echo "Copying $IMG -> localhost:5000/$REPO_TAG"
  skopeo copy --all docker://"$IMG" docker://localhost:5000/"$REPO_TAG" --dest-tls-verify=false
done < images.txt
```

Upload the chart archive to helm‑local
```bash
NEXUS_HOST=$(oc get route nexus -n nexus -o jsonpath='{.spec.host}')
NEXUS_URL="https://${NEXUS_HOST}"

# Upload chart (replace password with your changed admin password)
curl -u admin:'<ADMIN_PASSWORD>' \
  --upload-file mission-control-1.15.0.tgz \
  "${NEXUS_URL}/repository/helm-local/"

# Add internal Helm repo
helm repo add mc-internal "${NEXUS_URL}/repository/helm-local/" \
  --username admin --password '<ADMIN_PASSWORD>'
helm repo update
helm search repo mc-internal/mission-control
```

### Install Cert-Manager
Mission Control uses cert-manager CRDs (Certificate, Issuer, ClusterIssuer). Install upstream cert-manager via Helm:

```bash
# Namespace
oc create namespace cert-manager || true

# Install CRDs
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
```
### Helm repo and install
```bash
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm upgrade --install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v1.14.5
```

### Verify CRDs and pods
```bash
oc get crd certificates.cert-manager.io issuers.cert-manager.io clusterissuers.cert-manager.io
oc get pods -n cert-manager
```
