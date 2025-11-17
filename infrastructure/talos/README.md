# Talos infrastructure overlays

This directory contains Talos-specific infrastructure overlays.

- `cert-manager/` (canonical): contains the cert-manager kustomize overlay which references `../../base/cert-manager/` as the upstream source.
- `clusterissuer/`: contains the ClusterIssuer and cloudflare secret. A Flux Kustomization under `clusters/talos` depends on `cert-manager` so the ClusterIssuer is applied only after cert-manager is ready.

Guidelines:

- Keep the cert-manager manifests in `infrastructure/base/cert-manager` and reference them from `infrastructure/talos/cert-manager`.
- Do not duplicate cert-manager manifests in other directories; create overlays that reference the base as needed.
