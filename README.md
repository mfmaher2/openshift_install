# openshift_install
Mission Control on OpenShift — Quick Install Guide
This guide walks through a clean install of Mission Control on OpenShift with fast, reliable defaults. It exposes the UI via an OpenShift Route using TLS passthrough, logs you in with a known admin user, and includes OpenShift-specific notes for creating your first Cassandra cluster (SCC on OCP). Observability components (Loki/Mimir/Grafana) are disabled initially for speed and can be enabled later.

Prerequisites
OpenShift cluster admin permissions and oc CLI
Helm v3.12+ installed
Access to the Mission Control Helm chart registry
Files in this directory:
values-defaults.yaml (or values-full.yaml)
dex-reset.yaml (with a bcrypt hash for the admin password)
disable-observability.yaml (optional for first-time install)
Tip: To generate a bcrypt hash for dex-reset.yaml, see “Appendix: Generate a bcrypt hash.”
