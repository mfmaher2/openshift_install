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