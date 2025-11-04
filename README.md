# Mission Control on OpenShift — Quick Install Guide

This guide walks through a clean install of Mission Control on OpenShift with fast, reliable defaults. It exposes the UI via an OpenShift Route using TLS passthrough, logs you in with a known admin user, and includes OpenShift-specific notes for creating your first Cassandra cluster (SCC on OCP). Observability components (Loki/Mimir/Grafana) are disabled initially for speed and can be enabled later.

## Prerequisites

- OpenShift cluster admin permissions and oc CLI
- Helm v3.12+ installed
- Access to the Mission Control Helm chart registry
- Files in this directory:
  - values-defaults.yaml (or values-full.yaml)
  - dex-reset.yaml (with a bcrypt hash for the admin password)
  - disable-observability.yaml (optional for first-time install)

Tip: To generate a bcrypt hash for dex-reset.yaml, see “Appendix: Generate a bcrypt hash.”

---

## 1) Install Mission Control (fresh cluster)

This installs Mission Control into the mission-control namespace using your values files.
# Create namespace
oc new-project mission-control

# Install Mission Control
CHART_VERSION=1.15.0
helm upgrade --install mission-control \
  oci://registry.replicated.com/mission-control/mission-control \
  -n mission-control \
  -f values-defaults.yaml \
  -f dex-reset.yaml \
  -f disable-observability.yaml \
  --version ${CHART_VERSION}

# Watch pods
oc get pods -n mission-control
