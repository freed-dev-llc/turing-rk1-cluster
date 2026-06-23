# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

## [1.2.1] - 2026-06-23

### Fixed

- **`deploy-talos-cluster.sh` hardened for real RK1 / Talos v1.13 deploys** (found and validated end-to-end on the 4-node hardware):
  - Pin Kubernetes via `--kubernetes-version` (default `K8S_VERSION=v1.35.0`) — the script previously deployed Talos's default (v1.36.2), not the repo's intended version ([#47](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/47)).
  - Set `machine.install.image` to the Factory **schematic** installer (`factory.talos.dev/installer/<SCHEMATIC_ID>:<version>`) so the `sbc-rockchip` overlay + `rockchip-rknn` NPU extension survive `talosctl upgrade` (was the plain `ghcr.io/siderolabs/installer`) ([#45](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/45)).
  - Stage the flash image on the BMC microSD (`/mnt/sdcard`; override `BMC_IMAGE_PATH`) instead of `/tmp` — the Turing Pi 2 BMC's `/tmp` is a ~58 MB tmpfs that cannot hold the 2.2 GB image ([#44](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/44)).
  - Bootstrap now polls the **secure** API (a configured node stops serving the insecure API), guards on real etcd membership (`get etcdmembers`), and retries until etcd is ready ([#39](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/39)).
- **`cluster-config/*-patch.yaml`**: drop the legacy `machine.network.hostname` — on Talos v1.13 it conflicts with the `HostnameConfig` document and fails `talosctl validate` ([#46](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/46)).
- **`wipe-cluster.sh`**: reset each Talos node via its own endpoint (`--endpoints $ip`) — resets previously proxied through the control-plane endpoint, so worker resets failed once the CP was reset ([#49](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/49)).
- **`cluster-config/`**: renamed the tracked `controlplane.yaml` / `worker.yaml` to `*.example.yaml` (sanitized references) and gitignored the working names — `deploy-talos-cluster.sh generate` writes real cluster secrets to `controlplane.yaml` / `worker.yaml`, which were tracked and therefore stage-able by `git add -A` ([#40](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/40)).
- **Docs:** marked the Talos NPU `rocket` driver binding as **verified on real hardware** (was an unverified caveat — `/dev/accel/accel0` confirmed on all 4 nodes); corrected the stale "control plane has no NVMe by design" claim — all 4 nodes have a 500GB NVMe and the control plane's is used for Longhorn (`allowSchedulingOnControlPlanes`); NVMe storage total `1.5TB` → `2TB`.

## [1.2.0] - 2026-06-23

### Added

- **freed-dev-llc Turing Pi branding system** — horizontal banner logo, square favicon icon, cross-repo buttons, and a rebranded social-preview image (Configuration theme) (#34, #37).
- **`siderolabs/rockchip-rknn` NPU system extension** added to the Talos image schematic — ships the mainline open `rocket` NPU driver (Linux 6.18) (#36).
- `docs/HARDWARE-TEST-PLAN.md` — phased plan to validate the cluster on real RK1 hardware (#42).
- `.editorconfig` to keep formatting consistent across editors.
- `.github/CODEOWNERS` matching the pattern used by sister freed-dev-llc repos.
- This `CHANGELOG.md`.

### Changed

- Mermaid diagrams in `docs/` locked to `neutral` theme for cross-mode legibility (#21).
- Dependabot config now ignores the `repo/u-boot-rockchip` submodule (updater crashes upstream) (#22).
- README badges + INSTALLATION docs updated for the `jfreed-dev` → `freed-dev-llc` org migration (#23, #25).
- INSTALLATION docs: Talos version bumped from v1.11.6 to v1.13.2 (#25, completed in #27 + this PR which caught references in CLUSTER_PLAN.md / QUICKREF.md / scripts/deploy-talos-cluster.sh / cluster-config/*.yaml that #25 missed).
- **Cluster machineconfig** (`cluster-config/*.yaml`, 8 files): bumped `installer:v1.11.6` → `v1.13.2` and all Kubernetes component images (kubelet, kube-apiserver, kube-controller-manager, kube-proxy, kube-scheduler) from v1.34.1 → v1.35.0 to match the Kubernetes version Talos v1.13.2 actually ships. These configs are applied via `talosctl apply-config` on fresh deploys; upgrading an existing cluster instead requires `talosctl upgrade` (OS) + `talosctl upgrade-k8s` (Kubernetes), since `apply-config` alone does not reinstall the OS.
- **Deploy script** (`scripts/deploy-talos-cluster.sh`): `TALOS_VERSION` default bumped from v1.11.6 to v1.13.2 so fresh deploys pull the matching image from the Factory.
- README badges + INSTALLATION/COMPARISON Kubernetes version: v1.34.1 → v1.35.0 (matches what Talos v1.13.2 ships, per Talos Factory's `kubernetes_version` field in the manifest).
- MetalLB pool range reconciled to ground-truth `10.10.88.80-89` across all docs (was inconsistent: `80-89` in 4 places, `80-99` in 4) (#23).
- STORAGE.md hostnames switched from auto-generated `talos-0ow-v7t`-style IDs to `turing-w1/w2/w3` to match the documented hostname patch (#23).
- INSTALLATION storage table corrected — Node 1 control plane has no NVMe by design (#23).
- `docs/README.md` scripts table now lists `deploy-talos-cluster.sh` and `talos-cluster-status.sh` (#23).
- CI/Dependabot: PAT-based auto-approve/merge with a semver-major guard (#29), then switched to the org reusable auto-merge workflow (#30).
- Dependency bumps: `actions/checkout` → v7.0.0 (#31, #33), plus `actions/setup-python`, `markdownlint-cli2-action`, and submodules `repo/rknn-llm` (#32) / `repo/sbc-rockchip`.
- Talos NPU/GPU docs corrected from "not supported" to **Partial** — the open `rocket` (NPU) and `panthor` (GPU) drivers load via contrib extensions, while the proprietary RKNN/RKLLM SDK remains K3s/Armbian-only; dropped the inaccurate PCIe "passthrough" framing (#41).
- README: "Choose Your Distribution" NPU/GPU column → "Partial"; Linux kernel `6.12.62` → `6.18.36` (the actual Talos v1.13.5 kernel) (#41).

### Fixed

- **Talos `v1.13.2` → `v1.13.5`** (`cluster-config/*.yaml` 8 installer images, `scripts/deploy-talos-cluster.sh`, plus README / INSTALLATION / QUICKREF / CLUSTER_PLAN references): Talos v1.13.2 crash-loops `kube-scheduler` when running Kubernetes v1.35 — it renders the Kubernetes 1.36-only scheduler plugin extension points (`placementGenerate` / `placementScore`) into the generated scheduler config, which the v1.35 scheduler rejects with a strict-decoding error ([siderolabs/talos#13350](https://github.com/siderolabs/talos/issues/13350)). Fixed upstream in v1.13.3; pinned to the latest v1.13 patch, `v1.13.5`. Kubernetes stays at `v1.35.0` (within Talos v1.13's supported 1.31–1.36 range).
- `CLUSTER_PLAN.md` `kubectl get nodes` example output still showed Kubernetes `v1.34.x`; bumped to `v1.35.x` to match the `v1.35.0` bump applied across the other docs.
- License badge: MIT → Apache 2.0 to match the actual `LICENSE` file (#25).
- `docs/INSTALLATION.md` ingress-nginx URL was pinned to `v1.12.0-beta.0`; bumped to `v1.13.3` GA (#24).
- `~/Code/turing-rk1-cluster` hardcoded paths in `CLUSTER_PLAN.md` and `docs/QUICKREF.md` generalized to `$REPO_ROOT` (#24).
- `docs/INSTALLATION.md` Version Reference table cited ingress-nginx `v1.12.0-beta.0`; bumped to `v1.13.3` for parity with the install snippet at line 753 (#27).
- `docs/INSTALLATION-K3S.md` K3s Ref table NGINX Ingress `v1.12.x` → `v1.13.x` (#27).
- README License section body still said "MIT license" even after #25 fixed the badge; now says "Apache 2.0 license (see LICENSE)" (#27).
- MetalLB pool YAML snippets at `docs/INSTALLATION.md:732` and `docs/INSTALLATION-K3S.md:604` still used `10.10.88.80-10.10.88.99` (full-range form); reconciled to `80-89` to match the table form and the ground-truth `cluster-config/metallb-config.yaml` (#27).
- **Config audit (#36):** the `.gitignore` secret-guard for the control-plane node config was illusory (a tracked sanitized copy defeated it) — renamed to `controlplane-node1.example.yaml` and tightened the ignore glob; stale `images/latest_link.txt` pointing at Talos `v1.11.6` → `v1.13.5`; dead `BMC_IP` → `BMC_HOST` in `.env.example`.
- **Script hardening (#38):** `setup-k3s-node.sh` refuses to format a populated NVMe (data-loss guard) and adds fstab `nofail`; `wipe-cluster.sh` fixed `ssh`-under-`set -e` (no more half-wiped fleet), now wipes the NVMe on Talos resets, and uses `BMC_HOST`; `deploy-k3s-cluster.sh` polls for readiness, derives the TLS SAN, and detects failed remote installs; `talos-cluster-status.sh` no longer aborts on one failed probe; gitignored `cluster-config/*-patched.yaml`; narrowed the `reset` glob.
