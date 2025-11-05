# Mission Control Air‑Gapped Install on OpenShift

This guide uses an in‑cluster Nexus (“mock repository”) to host both container images (Docker hosted) and Helm charts (Helm hosted), then installs Mission Control entirely from that Nexus. Observability (Loki/Mimir/Grafana) is disabled initially to avoid object‑store prerequisites; you can enable it later.

## Prerequisites

- OpenShift cluster admin (oc CLI configured)
- Helm v3.12+ on your workstation
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
## Install cert‑manager (air‑gapped)
Mission Control creates cert-manager resources; install cert-manager from Nexus (after mirroring its chart and images).

Download CRDs once (to include in your bundle)
```bash
curl -L -o cert-manager.crds.yaml \
  https://github.com/cert-manager/cert-manager/releases/download/v1.14.5/cert-manager.crds.yaml
```
Mirror cert‑manager chart and images
```bash
# Pull and upload chart
helm repo add jetstack https://charts.jetstack.io
helm repo update
helm pull jetstack/cert-manager --version v1.14.5

curl -u admin:'<ADMIN_PASSWORD>' \
  --upload-file cert-manager-v1.14.5.tgz \
  "${NEXUS_URL}/repository/helm-local/"

# Render to find images
helm template cert-manager jetstack/cert-manager --version v1.14.5 -n cert-manager > cm.yaml
yq e -r '.. | .image? | select(.)' cm.yaml | sort -u > cert-images.txt

# Mirror to Nexus (strip source registry prefix)
while read -r IMG; do
  REPO_TAG=$(echo "$IMG" | sed -E 's#^[^/]+/##')
  echo "Copying $IMG -> localhost:5000/$REPO_TAG"
  skopeo copy --all docker://"$IMG" docker://localhost:5000/"$REPO_TAG" --dest-tls-verify=false
done < cert-images.txt
```

Install cert‑manager in the air‑gapped cluster
```bash
# Namespace and CRDs
oc create namespace cert-manager || true
kubectl apply -f cert-manager.crds.yaml

# Pull secret for Nexus (cert-manager ns)
oc create secret docker-registry mc-regcred \
  --docker-server=nexus-docker.nexus.svc.cluster.local:5000 \
  --docker-username=admin \
  --docker-password='<ADMIN_PASSWORD>' \
  -n cert-manager

# Install from internal Helm repo
helm repo add mc-internal "${NEXUS_URL}/repository/helm-local/" \
  --username admin --password '<ADMIN_PASSWORD>'
helm repo update

helm upgrade --install cert-manager mc-internal/cert-manager \
  -n cert-manager \
  -f certmgr-images.yaml \
  --version v1.14.5

# Verify
oc get crd certificates.cert-manager.io issuers.cert-manager.io clusterissuers.cert-manager.io
oc get pods -n cert-manager
```
## Prepare Mission Control values (air‑gapped)
Force all images to pull from Nexus (include required globals)
```bash
# Pull secret for mission-control namespace
oc new-project mission-control || true
oc create secret docker-registry mc-regcred \
  --docker-server=nexus-docker.nexus.svc.cluster.local:5000 \
  --docker-username=admin \
  --docker-password='<ADMIN_PASSWORD>' \
  -n mission-control
```

## Install Mission Control from Nexus
```bash
CHART_VERSION=1.15.0
helm upgrade --install mission-control mc-internal/mission-control \
  -n mission-control \
  -f values-defaults.yaml \
  -f dex-reset.yaml \
  -f disable-observability.yaml \
  -f airgap-images.yaml \
  --version ${CHART_VERSION}

oc get pods -n mission-control
```
Expose UI with TLS passthrough (UI listens HTTPS on 8080):
```bash
# Ensure service port "https" -> 8080
oc patch svc mission-control-ui -n mission-control \
  -p '{"spec":{"ports":[{"name":"https","port":8080,"protocol":"TCP","targetPort":8080}]}}'

# Create route if missing
oc get route mission-control-ui -n mission-control >/dev/null 2>&1 || \
  oc expose service mission-control-ui -n mission-control

# TLS passthrough to the pod
oc patch route mission-control-ui -n mission-control \
  -p '{"spec":{"tls":{"termination":"passthrough","insecureEdgeTerminationPolicy":"Redirect"},"port":{"targetPort":"https"}}}'

# Print URL
HOST=$(oc get route mission-control-ui -n mission-control -o jsonpath='{.spec.host}')
echo "Mission Control UI: https://${HOST}"
```
Login:
  -  Username: admin@example.com
  -  Password: (the clear‑text password you hashed into dex‑reset.yaml)

## Create clusters on OpenShift (SCC)
If the first Cassandra pod is rejected due to SCC (UID/fsGroup), grant anyuid to the ServiceAccount used by the StatefulSet in the cluster’s namespace, then recreate the pod:
```bash
# Identify the SA used by the StatefulSet
oc get sts -n <cluster-ns> <sts-name> -o jsonpath='{.spec.template.spec.serviceAccountName}{"\n"}'

# Grant SCC (start with anyuid)
oc adm policy add-scc-to-user anyuid -z <sa-name> -n <cluster-ns>

# Recreate rack-0 pod or scale
oc delete pod -n <cluster-ns> <sts-name>-0
# or:
oc scale sts -n <cluster-ns> <sts-name> --replicas=0
oc scale sts -n <cluster-ns> <sts-name> --replicas=1
```
## Troubleshooting
- Chart rendering fails due to Loki/Mimir validations:
  - Template/install with observability disabled (loki.enabled=false, mimir.enabled=false, grafana.enabled=false).
- Pods still pull from the internet:
  - Ensure images.txt covered everything and mirroring succeeded.
  - Confirm pod specs show nexus-docker.nexus.svc.cluster.local:5000/... and mc-regcred is attached.
- Route shows “Application is not available”:
  - Service port name “https” → targetPort 8080, Route termination “passthrough”.
  - Check NetworkPolicies; allow from namespaceSelector label network.openshift.io/policy-group=ingress to port 8080.
- Dex login fails after password change:
  - Ensure bcrypt is quoted in dex-reset.yaml; apply via Helm; restart Dex:
    ```bash
    oc rollout restart deploy/mission-control-dex -n mission-control
    ```
## Appendix: Generate a bcrypt hash (for Dex)
```bash
python3 - <<'PY'
import bcrypt
print(bcrypt.hashpw(b'NewStrongPassword!', bcrypt.gensalt(rounds=10)).decode())
PY
```
