# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- `.editorconfig` to keep formatting consistent across editors.
- `.github/CODEOWNERS` matching the pattern used by sister freed-dev-llc repos.
- This `CHANGELOG.md`.

### Changed

- Mermaid diagrams in `docs/` locked to `neutral` theme for cross-mode legibility (#21).
- Dependabot config now ignores the `repo/u-boot-rockchip` submodule (updater crashes upstream) (#22).
- README badges + INSTALLATION docs updated for the `jfreed-dev` → `freed-dev-llc` org migration (#23, #25).
- INSTALLATION docs: Talos version bumped from v1.11.6 to v1.13.2 (#25, completed in #27 + this PR which caught references in CLUSTER_PLAN.md / QUICKREF.md / scripts/deploy-talos-cluster.sh / cluster-config/*.yaml that #25 missed).
- **Cluster machineconfig** (`cluster-config/*.yaml`, 8 files): bumped `installer:v1.11.6` → `v1.13.2` and all Kubernetes component images (kubelet, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler) from v1.34.1 → v1.35.0 to match the Kubernetes version Talos v1.13.2 actually ships. Applying these configs via `talosctl apply-config` will upgrade nodes to Talos v1.13.2 + K8s v1.35.0.
- **Deploy script** (`scripts/deploy-talos-cluster.sh`): `TALOS_VERSION` default bumped from v1.11.6 to v1.13.2 so fresh deploys pull the matching image from the Factory.
- README badges + INSTALLATION/COMPARISON Kubernetes version: v1.34.1 → v1.35.0 (matches what Talos v1.13.2 ships, per Talos Factory's `kubernetes_version` field in the manifest).
- MetalLB pool range reconciled to ground-truth `10.10.88.80-89` across all docs (was inconsistent: `80-89` in 4 places, `80-99` in 4) (#23).
- STORAGE.md hostnames switched from auto-generated `talos-0ow-v7t`-style IDs to `turing-w1/w2/w3` to match the documented hostname patch (#23).
- INSTALLATION storage table corrected — Node 1 control plane has no NVMe by design (#23).
- `docs/README.md` scripts table now lists `deploy-talos-cluster.sh` and `talos-cluster-status.sh` (#23).

### Fixed

- License badge: MIT → Apache 2.0 to match the actual `LICENSE` file (#25).
- `docs/INSTALLATION.md` ingress-nginx URL was pinned to `v1.12.0-beta.0`; bumped to `v1.13.3` GA (#24).
- `~/Code/turing-rk1-cluster` hardcoded paths in `CLUSTER_PLAN.md` and `docs/QUICKREF.md` generalized to `$REPO_ROOT` (#24).
- `docs/INSTALLATION.md` Version Reference table cited ingress-nginx `v1.12.0-beta.0`; bumped to `v1.13.3` for parity with the install snippet at line 753 (#27).
- `docs/INSTALLATION-K3S.md` K3s Ref table NGINX Ingress `v1.12.x` → `v1.13.x` (#27).
- README License section body still said "MIT license" even after #25 fixed the badge; now says "Apache 2.0 license (see LICENSE)" (#27).
- MetalLB pool YAML snippets at `docs/INSTALLATION.md:732` and `docs/INSTALLATION-K3S.md:604` still used `10.10.88.80-10.10.88.99` (full-range form); reconciled to `80-89` to match the table form and the ground-truth `cluster-config/metallb-config.yaml` (#27).
