# archive

Manifests that are switched off but kept for later. **Flux never applies anything here**: no Flux
Kustomization in `clusters/my-cluster/apps.yaml` points at `./archive`.

To bring something back, `git mv` it into its `apps/<folder>/`, list it in that folder's
`kustomization.yaml`, and push.

- `compliance/kubescape-pv.yaml`, `compliance/kubescape-servicemonitor.yaml`: Kubescape (disabled 2026-11-14;
  the rest of the re-enable steps are in the note above the commented HelmRelease in `apps/compliance/helmrelease.yaml`).
- `gaming/project-zomboid.yaml` + `gaming/secret.pz.yaml` (SOPS): Project Zomboid server, image pinned to `v1.2.0`.
