# Hardware Test Plan — Turing RK1 Cluster

Validation of recent changes against the **real 4-node RK1 hardware**. Covers the merged work (#35–#41), the two deferred issues (#39, #40), and the unverified NPU/GPU driver binding noted in #41.

> **Run order:** Phase 1 (non-destructive) → Phase 2 (single-node destructive) → Phase 3 (full redeploy, maintenance window) → Phase 4 (optional functional). Do not skip ahead to destructive phases until Phase 1 passes.

## Cluster facts

| Item | Value |
|------|-------|
| BMC | `10.10.88.70` |
| Control plane | `10.10.88.73` (turing-cp1) |
| Workers | `10.10.88.74/75/76` (turing-w1/w2/w3) |
| Talos / K8s / kernel | v1.13.5 / v1.35.0 / 6.18.36 |
| System disk / data disk | eMMC `/dev/mmcblk0` / NVMe `/dev/nvme0n1` |
| NPU / GPU device nodes | `/dev/accel/accel0` (rocket) / `/dev/dri/renderD128` (panthor) |
| Schematic ID (NPU image) | `35d07e8c5e698ec18c7356eded0849b477585f45454d86a52b4e0b87e2e543ec` |

## Prerequisites

- `talosctl` (v1.13.5), `kubectl` (v1.35.x), `tpi`, `ssh`, `jq` on the workstation.
- `cp .env.example .env` and fill `TPI_USERNAME` / `TPI_PASSWORD` (and `BMC_HOST`/`SSH_USER` if non-default).
- `export TALOSCONFIG=$PWD/cluster-config/talosconfig` and `KUBECONFIG=$PWD/cluster-config/kubeconfig`.
- **Designated test node** for Phase 2: a worker (recommended `turing-w3` / `10.10.88.76`) that can be drained and wiped.
- **Maintenance window** for Phase 3 (full cluster redeploy).
- Longhorn volumes backed up before any destructive phase.

## Conventions

- 🟢 non-destructive · 🟡 single-node destructive (data loss on one node) · 🔴 full-cluster destructive.
- Each test: **Objective → Steps → Expected → Pass/Fail**. Record results in the sign-off table.

---

## Phase 1 — Non-destructive smoke tests 🟢

### T1.1 — Status script runs and reports health

- **Steps:** `./scripts/talos-cluster-status.sh`
- **Expected:** auto-detects cluster type; prints node reachability, K8s nodes, pod summary, LoadBalancer/Ingress, Longhorn, recent warnings. No abort partway.
- **Validates:** #38 (status script `set -u`, no `-e` abort).

### T1.2 — Status script survives a down control plane (#38 H1)

- **Steps:** temporarily block the CP (e.g. power off `turing-cp1` via `./scripts/wipe-cluster.sh status` then `tpi power off -n 1`), rerun `talos-cluster-status.sh`, then power back on.
- **Expected:** prints a **node reachability table** and exits cleanly — not a single error line with no output.
- **Validates:** #38 H1.

### T1.3 — Kubernetes version + kube-scheduler healthy (#35)

- **Steps:** `kubectl get nodes -o wide`; `kubectl -n kube-system get pods | grep scheduler`; `kubectl -n kube-system logs -l component=kube-scheduler --tail=20`
- **Expected:** nodes report **v1.35.0**; kube-scheduler pods **Running**, no `strict decoding`/`placementGenerate` crash-loop.
- **Validates:** #35 (Talos v1.13.2 scheduler bug fixed by v1.13.5).

### T1.4 — Talos + kernel versions (#35, #41)

- **Steps:** `talosctl -n 10.10.88.73 version`; `talosctl -n 10.10.88.73 read /proc/version`
- **Expected:** Talos **v1.13.5**, kernel **6.18.36**.
- **Validates:** #35 version bump, #41 README kernel correction.

### T1.5 — NPU driver binding (#41 unverified caveat) 🟢

- **Steps:** on each node: `talosctl -n <ip> ls /dev/accel/ 2>/dev/null`; `talosctl -n <ip> dmesg | grep -i rocket`
- **Expected:** `/dev/accel/accel0` present and `rocket` driver bound. **If absent:** the `sbc-rockchip` RK1 DTB does not enable the NPU node → file a finding; the `rockchip-rknn` extension ships the `.ko` but it isn't binding.
- **Validates:** #41 NPU "Partial" claim + the open question it flagged.

### T1.6 — GPU driver binding 🟢

- **Steps:** `talosctl -n <ip> ls /dev/dri/`; `talosctl -n <ip> dmesg | grep -iE 'panthor|mali'`
- **Expected:** `/dev/dri/renderD128` present, `panthor` bound. If absent, same DTB caveat applies.
- **Validates:** #41 GPU "Partial" claim.

### T1.7 — BMC connectivity + `BMC_HOST` var (#38)

- **Steps:** `./scripts/wipe-cluster.sh status`; then `BMC_HOST=10.10.88.70 ./scripts/wipe-cluster.sh status`
- **Expected:** BMC reachable; setting `BMC_HOST` is honored (no `BMC_IP`).
- **Validates:** #38 BMC_HOST standardization.

### T1.8 — Schematic image builds for the pinned version (#36)

- **Steps:** `curl -sI "https://factory.talos.dev/image/35d07e8c5e698ec18c7356eded0849b477585f45454d86a52b4e0b87e2e543ec/v1.13.5/metal-arm64.raw.xz" | head -1`; `curl -s -X POST --data-binary @talos-schematic.yaml https://factory.talos.dev/schematics | jq -r .id`
- **Expected:** HTTP 200; returned id == the schematic ID above.
- **Validates:** #36 NPU schematic + ID integrity.

### T1.9 — Longhorn mount + fstab `nofail` (K3s nodes only) 🟢

- **Steps (K3s/Armbian path):** `ssh root@<node> 'findmnt /var/lib/longhorn; grep longhorn /etc/fstab'`
- **Expected:** mounted; fstab entry contains `nofail,x-systemd.device-timeout`.
- **Validates:** #38 setup-k3s fstab hardening (on already-provisioned nodes).

---

## Phase 2 — Single-node destructive (designated worker) 🟡

> Drain first: `kubectl drain turing-w3 --ignore-daemonsets --delete-emptydir-data`. Restore after each test.

### T2.1 — setup-k3s-node refuses to format a populated NVMe (#38 C1) 🟡

- **Steps (on the drained K3s test node, with existing data on `/dev/nvme0n1` but unmounted):** `umount /var/lib/longhorn; ./setup-k3s-node.sh turing-w3`
- **Expected:** script **refuses** with "already contains a filesystem … FORCE_WIPE=1 to override" and exits non-zero — does **not** `mkfs`. Re-run with `FORCE_WIPE=1` proceeds.
- **Pass/Fail:** PASS only if existing data is preserved without the override.
- **Validates:** #38 C1 (the data-loss guard).

### T2.2 — fstab `nofail` survives a missing NVMe (#38 H1) 🟡

- **Steps:** on the test node, simulate NVMe absence (detach / wrong UUID in fstab) and reboot.
- **Expected:** node **still boots** and is SSH-reachable (no emergency-shell hang); `/var/lib/longhorn` simply unmounted.
- **Validates:** #38 H1 (headless boot resilience).

### T2.3 — wipe-cluster single-node NVMe wipe + reliability (#38 H1) 🟡

- **Steps:** `./scripts/wipe-cluster.sh node` → select node 4 → `nvme`. Observe with a deliberately flaky/closed SSH to one earlier node if testing multi-node.
- **Expected:** prompts for `yes`; wipes the NVMe; a mid-run ssh failure **warns and continues** rather than aborting the whole run.
- **Validates:** #38 H1 (`if ssh … then`, no half-wiped fleet).

### T2.4 — `wipe talos` actually wipes the NVMe (#38 H2) 🟡

- **Steps (Talos test node):** put data on its NVMe, `./scripts/wipe-cluster.sh talos` (single node scope), then re-provision and check NVMe is empty.
- **Expected:** the NVMe (Longhorn data) **is** wiped (via `--user-disks-to-wipe`), matching the message.
- **Validates:** #38 H2 (false "includes NVMe" fixed).

---

## Phase 3 — Full deploy / redeploy (maintenance window) 🔴

> Full cluster wipe + redeploy. Confirm backups and that downtime is acceptable.

### T3.1 — deploy-talos bootstrap readiness uses the secure API (#39 H2) 🔴

- **Steps:** clean deploy: `./scripts/deploy-talos-cluster.sh deploy` (or `apply` then `bootstrap`). Watch timing between `apply` and `bootstrap`.
- **Expected:** `bootstrap` waits for the **secure** API + node stage `running` (post-install reboot), not the maintenance API; no premature `talosctl bootstrap` against a not-yet-installed CP. No flaky failures on the slow eMMC install/reboot.
- **Validates:** #39 H2. **If it fails/flakes here, that is the data we need for issue #39.**

### T3.2 — etcd bootstrap run-once guard does not fail open (#39 H3) 🔴

- **Steps:** after a successful bootstrap, re-run `./scripts/deploy-talos-cluster.sh bootstrap`; also test with a brief CP API outage during the guard check.
- **Expected:** re-run is **refused** ("already bootstrapped"); a transient API outage does **not** cause a second bootstrap.
- **Validates:** #39 H3.

### T3.3 — controlplane/worker secret-file hygiene (#40) 🔴

- **Steps:** after `generate`/`deploy`: `git status --porcelain cluster-config/`; `git check-ignore cluster-config/controlplane.yaml cluster-config/worker.yaml cluster-config/*-patched.yaml`; masked scan: `grep -lE '(secret|token|key):[[:space:]]+[A-Za-z0-9+/]{20,}' cluster-config/*.yaml`
- **Expected:** real-secret files (`*-patched.yaml`) are git-ignored. Confirm the residual hazard from #40 (the deploy overwrites tracked `controlplane.yaml`/`worker.yaml` with secrets → they show as modified and are stage-able). **This test reproduces #40** and confirms whether the `.example` rename is needed.
- **Validates:** #40.

### T3.4 — deploy-k3s readiness poll + TLS SAN + remote pipefail (#38) 🔴

- **Steps (K3s path):** `./scripts/setup-k3s-node.sh <name>` on each node, then `./scripts/deploy-k3s-cluster.sh`.
- **Expected:** server step **polls** for node-token + `/readyz` (no fixed 30 s race on slow ARM); `kubectl` works through the rewritten kubeconfig (cert SAN = server IP); a simulated failed `curl | sh` (block get.k3s.io) **fails the step** instead of reporting success.
- **Validates:** #38 deploy-k3s fixes.

---

## Phase 4 — NPU / GPU functional (optional) 🟢/🟡

### T4.1 — NPU inference via Mesa Teflon on Talos

- **Steps:** privileged/CDI pod mounting `/dev/accel/accel0`; run a MobileNet TFLite model through the Teflon delegate.
- **Expected:** inference runs on the NPU (not CPU fallback). LLM/RKLLM is **expected to NOT work** here (open `rocket` stack).
- **Validates:** #41 NPU "Partial (Teflon, small CNNs)" claim.

### T4.2 — RKLLM on the K3s/Armbian path

- **Steps:** on a K3s node with the BSP `rknpu` driver, run an RKLLM example from `repo/rknn-llm`.
- **Expected:** RKLLM works on K3s (proprietary stack) — confirming the K3s-only claim in #41.

### T4.3 — GPU GL/Vulkan on Talos

- **Steps:** pod with `/dev/dri/renderD128`; run a Vulkan/GL ES probe (e.g. `vulkaninfo`).
- **Expected:** `panthor` device usable for GL ES/Vulkan; OpenCL limited/absent.

---

## Results sign-off

| Test | Date | Result (Pass/Fail/Skip) | Notes / issue link |
|------|------|-------------------------|--------------------|
| T1.1 | | | |
| T1.2 | | | |
| T1.3 | | | |
| T1.4 | | | |
| T1.5 | | | |
| T1.6 | | | |
| T1.7 | | | |
| T1.8 | | | |
| T1.9 | | | |
| T2.1 | | | |
| T2.2 | | | |
| T2.3 | | | |
| T2.4 | | | |
| T3.1 | | | #39 |
| T3.2 | | | #39 |
| T3.3 | | | #40 |
| T3.4 | | | |
| T4.1 | | | |
| T4.2 | | | |
| T4.3 | | | |

> Failures in T3.1–T3.3 are the evidence to drive issues #39/#40 to a fix. Update those issues with results.
