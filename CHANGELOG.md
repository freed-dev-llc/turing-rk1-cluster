# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- **`cluster-config/exo-web.yaml`: cluster-wide endpoint for the exo web UI + OpenAI-compatible API** - a ClusterIP Service load-balancing the exo `hostNetwork` DaemonSet across all four nodes, fronted by a Traefik Ingress (`http://exo.local/`), so the per-node `:52415` UIs share one stable URL ([#73](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/73)).
- **`cluster-config/exo-preload-cronjob.yaml`: self-healing NPU model preloader** - a CronJob in the `exo-rk` namespace that keeps a target model loaded on one exo instance per NPU node for N-way parallel request handling. exo instance state is ephemeral (a reboot, pod restart, or image roll drops loaded instances and exo does not auto-restore them, chat requests 404 rather than lazy-load), so the job reconciles the desired count after any restart; idempotent when healthy. Validated live: no-op at 4/4, and recovery relaunched a deleted instance back to 4/4. README documents the behavior and a worked example ([#74](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/74)).

### Changed

- **exo-rkllama reference bumped to `rk-v0.2.1`** in the README deployment note, matching the image deployed on the live cluster's DaemonSet (`rk-v0.2.0`: NPU-gated model catalog/search; `rk-v0.2.1`: the NPU onboarding pin) ([#75](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/75), [#77](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/77)).
- **`CLUSTER_PLAN.md`: reworked the "Next Steps After Deployment" list into a `Roadmap` section** - marks the metrics-server / monitoring / storage / NPU-Teflon items as shipped in v1.3.0, sets the v1.4.0 focus (K3s tooling reconciliation to match the re-platformed cluster, and RKLLM/exo-rkllama productionization), and moves Cilium (#57) and Longhorn backup (#62) to a deferred backlog.
- **Dependency bumps**: `actions/checkout` → v7.0.1 ([#82](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/82)); `actions/setup-python` → v7.0.0 ([#80](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/80), semver-major; the removed `pip-install` input was unused here); `DavidAnson/markdownlint-cli2-action` → v24.1.0 ([#79](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/79), semver-major); `github/codeql-action` init/analyze pins moved from the `v4.31.9` annotated-tag object SHA to the release commit SHA it dereferences to ([#78](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/78), [#81](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/81)).

### Fixed

- **README deployment note conflated the re-platform date with the current image** - it read as "the 2026-07-03 deployment runs `rk-v0.2.1`", but that date shipped `rk-v0.1.0` (see [1.3.0]); the note now dates the re-platform event and ties its validation figures to the recorded bring-up run. The README `Contents` list also gained the missing `Distributed NPU Inference (exo)` entry, and the exo section now summarizes the deployed `rk-v0.2.x` behavior from the exo-rkllama release notes ([#83](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/83)).
- **Changelog reference and link repairs** - `#77` was the file's only unlinked PR reference (GitHub does not autolink `#NN` in rendered Markdown files); the `rk-v0.2.1` bullet now attributes deltas per upstream release; Keep a Changelog link definitions added for `[Unreleased]`/`[1.3.0]`/`[1.2.1]`/`[1.2.0]`, the last as a tag link because its predecessor `v1.1.1` sits on the orphaned pre-re-root history ([#83](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/83)).
- **`CLUSTER_PLAN.md` v1.4.0 roadmap listed shipped work as pending** - exo ingress/routing ([#73](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/73)) and the model preloader ([#74](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/74)) already landed; the item now scopes the remaining DaemonSet hardening ([#83](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/83)).
- **`.markdownlint.json`'s `ignores` key was inert** - `ignores` is a markdownlint-cli2 option, not a lint rule, so the intended `repo/**` submodule exclusion never applied (masked in CI because `actions/checkout` leaves submodules empty). The rule config plus `ignores` moved to `.markdownlint-cli2.jsonc`, with `lint.yml` pointing at it; verified by canary (a violation under `repo/` is excluded, the same file under `docs/` is reported) ([#83](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/83)).
- **Live-cluster reconciliation** - the README deployment note named `rk-v0.2.1` while the running DaemonSet image is `rk-v0.2.3` (verified with kubectl against the cluster, 2026-08-07; the roll happened via the exo-rkllama deploy pin, per its rk-v0.2.2/rk-v0.2.3 releases of 2026-07-06). The note now names `rk-v0.2.3`, the exo section covers the `rk-v0.2.3` hub-add guidance, and the preloader docs record the observed `concurrencyPolicy: Forbid` wedge: a preload pod stuck `Terminating` on the dead rk1-2 node has blocked every scheduled run since 2026-07-13 ([#83](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/83)).
- **Preloader CronJob died on a flaky exo API** - once rk1-2 returned and the CronJob unwedged, reconcile jobs failed back-to-back: a timed-out `/state` fetch made `loaded_nodes()` emit an empty operand (busybox ash: `sh: out of range`, reproduced in alpine:3.20), and under `set -eu` a failed `placement=$(curl ...)` assignment killed a job with a one-line log. The script now falls back to empty state on fetch failures, guards the placement/launch calls (log and defer to the next run), defers when no node identities are visible, and its `MODEL` default matches the live CronJob target (`deepseek-r1-distill-qwen-14b-rkllm`, changed out-of-band since #74), so `kubectl apply` no longer flips the deployed model ([#84](https://github.com/freed-dev-llc/turing-rk1-cluster/pull/84)).

## [1.3.0] - 2026-07-03

### Changed

- **Physical cluster re-platformed from Talos to K3s on Armbian (2026-07-03)** to run
  RKLLM NPU inference, which requires the vendor-kernel rknpu driver that Talos cannot
  ship (see `docs/NPU-TEFLON.md` for the Talos-side analysis). All four nodes reflashed
  to Armbian Trixie (vendor kernel `6.1.115-vendor-rk35xx`, `RKNPU driver: v0.9.8`),
  K3s v1.36.2, [exo-rkllama](https://github.com/freed-dev-llc/exo-rkllama) `rk-v0.1.0`
  deployed as a privileged DaemonSet. Validated: streamed chat completions from the
  NPU (~90% load on all three cores, 3.46 tok/s generate on a w8a8_g128 3B), two
  replicas on distinct nodes serving concurrent requests 3/3. Bring-up procedure:
  exo-rkllama `docs/rk-hardware/RUNBOOK.md`. The Talos docs and scripts remain valid
  for redeploying Talos; they no longer describe the running cluster
  ([#69](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/69)).

### Added

- **`deploy-talos-cluster.sh metrics-server`** (Phase 10): installs the `metrics-server` Helm chart with `--kubelet-insecure-tls`, needed because Talos kubelet serving certs carry only a DNS SAN and fail the default IP-based TLS verification. Wired into the `deploy` flow alongside the Longhorn prompt. Validated end-to-end on the 4-node hardware - `kubectl top nodes`/`kubectl top pods` return data from all 4 nodes ([#58](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/58)).
- **`deploy-talos-cluster.sh monitoring`** (Phase 11): installs `kube-prometheus-stack` from `cluster-config/prometheus-values.yaml` and the Longhorn `ServiceMonitor`. Disabled the `kubeProxy` scrape target (metrics bind to `127.0.0.1` by default, unreachable from the ServiceMonitor). Validated end-to-end on the 4-node hardware - all 30 Prometheus scrape targets up, Grafana healthy with 15 pre-installed Kubernetes dashboards ([#59](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/59)).

- **`docs/STORAGE.md`: PVC creation and replication verification procedure** - covers creating a test PVC/pod, confirming replica count and per-node placement, rebooting a replica's node and watching the volume cycle `detached` → `attached` / `degraded` → `healthy` with data intact, and a gotcha about bare Pods getting permanently wedged after a brief node-not-ready blip (use a Deployment/StatefulSet instead). Validated end-to-end on the 4-node hardware ([#60](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/60)).
- **`docs/NPU-TEFLON.md` and `cluster-config/npu-teflon-test.yaml`**: NPU inference workload via Mesa's Teflon TFLite delegate - the RKNN/RKLLM SDK this repo also documents is K3s/Armbian-only, so on this Talos cluster the achievable path is the open `rocket` driver + Teflon. Since no distro packages Mesa's rocket/teflon support yet, the manifest builds Mesa from source (not the unofficial prebuilt binary some projects use) and runs a MobileNetV1 classification through the delegate. Validated end-to-end on the 4-node hardware - all 27 CONV/DWCONV layers `supported` (genuine NPU offload), correct classification result, reproduced from a clean pod on two different nodes ([#61](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/61)).

### Fixed

- **`docs/INSTALLATION-K3S.md`: wrong Armbian image for NPU work** - the guide
  downloaded `Bookworm_current` (mainline kernel, no rknpu driver), which reproduces
  the Talos NPU dead-end on Armbian. It now prescribes the `vendor` kernel image
  (`Trixie_vendor_minimal`) with the verification command, and stages the decompressed
  image on the BMC microSD for a BMC-local flash (the remote-flash-from-/tmp pitfall
  already documented for Talos in
  [#44](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/44))
  ([#69](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/69)).

- **`install_longhorn` / new `install_monitoring`: privileged pods rejected by the cluster's default "baseline" PodSecurity** on a freshly-created namespace - `longhorn-manager` (hostPath/privileged) and `node-exporter` (hostNetwork/hostPID) both stayed un-schedulable until their namespace was labeled `pod-security.kubernetes.io/enforce=privileged`. `docs/INSTALLATION.md`'s manual steps already did this; the scripted `install_longhorn()` path did not. Both phases now label their namespace before installing ([#65](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/65)).

- **`README.md`: incorrect K3s badge, RAM total, and submodule list** - the K3s badge claimed `v1.31` (the deploy script installs from the stable channel with no version pin, now labeled "stable channel"); the RAM total read `64-128GB`, contradicting the all-32GB topology (corrected to `128GB` for 4x 32GB); and the `u-boot-rockchip` submodule (defined in `.gitmodules`) was missing from the documented directory structure ([#54](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/54)).

- **`.gitignore`: generated kubeconfig variants left untracked-but-visible** - the rule matched only the exact filename `kubeconfig`, so variants like `kubeconfig-k3s` under `cluster-config/` were not ignored. Broadened to a glob so any kubeconfig credential there is ignored ([#53](https://github.com/freed-dev-llc/turing-rk1-cluster/issues/53)).

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

[Unreleased]: https://github.com/freed-dev-llc/turing-rk1-cluster/compare/v1.3.0...HEAD
[1.3.0]: https://github.com/freed-dev-llc/turing-rk1-cluster/compare/v1.2.1...v1.3.0
[1.2.1]: https://github.com/freed-dev-llc/turing-rk1-cluster/compare/v1.2.0...v1.2.1
[1.2.0]: https://github.com/freed-dev-llc/turing-rk1-cluster/releases/tag/v1.2.0
