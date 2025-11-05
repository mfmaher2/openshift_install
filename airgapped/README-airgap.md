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

## 1) Deploy an in‑cluster Nexus (UI + Docker registry)

Apply Nexus resources (Namespace, SA, PVC, Deployment, Services, Route):

```bash
cat <<'YAML' | oc apply -f -
apiVersion: v1
kind: Namespace
metadata:
  name: nexus
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: nexus-sa
  namespace: nexus
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: nexus-data
  namespace: nexus
spec:
  accessModes: [ "ReadWriteOnce" ]
  resources:
    requests:
      storage: 20Gi
  # storageClassName: <OPTIONAL-YOUR-SC>   # set if your cluster requires an explicit SC
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nexus
  namespace: nexus
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nexus
  template:
    metadata:
      labels:
        app: nexus
    spec:
      serviceAccountName: nexus-sa
      securityContext:
        fsGroup: 2000
      containers:
      - name: nexus
        image: docker.io/sonatype/nexus3:latest
        imagePullPolicy: IfNotPresent
        ports:
        - containerPort: 8081   # Nexus UI / REST
        - containerPort: 5000   # Docker registry (you enable repo in UI)
        volumeMounts:
        - name: nexus-data
          mountPath: /nexus-data
        securityContext:
          runAsUser: 200
      volumes:
      - name: nexus-data
        persistentVolumeClaim:
          claimName: nexus-data
---
apiVersion: v1
kind: Service
metadata:
  name: nexus
  namespace: nexus
spec:
  selector:
    app: nexus
  ports:
  - name: ui
    port: 8081
    targetPort: 8081
---
apiVersion: v1
kind: Service
metadata:
  name: nexus-docker
  namespace: nexus
spec:
  selector:
    app: nexus
  ports:
  - name: registry
    port: 5000
    targetPort: 5000
---
apiVersion: route.openshift.io/v1
kind: Route
metadata:
  name: nexus
  namespace: nexus
spec:
  to:
    kind: Service
    name: nexus
  port:
    targetPort: ui
  tls:
    termination: edge
YAML

```
