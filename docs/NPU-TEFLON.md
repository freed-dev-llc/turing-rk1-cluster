# NPU Inference via Mesa Teflon (Talos)

Validates real NPU-accelerated inference on this Talos cluster's RK3588 NPU
using the open-source `rocket` kernel driver and Mesa's Teflon TFLite
delegate. Run end-to-end on the 4-node hardware (2026-07-03).

## Why not RKNN?

This repo's own [COMPARISON.md](COMPARISON.md) and
[HARDWARE-TEST-PLAN.md](HARDWARE-TEST-PLAN.md) are explicit: the proprietary
RKNN/RKLLM SDK (`repo/rknn-llm`, `repo/rknn-toolkit2`) only works on the
**K3s/Armbian** path (vendor BSP kernel). On **Talos**, only the mainline
open `rocket` driver is available, exposed through Mesa's Teflon TFLite
delegate - small CNNs (MobileNet-class) only, no LLM/RKLLM support.

## What's confirmed working

- `/dev/accel/accel0` present on all 4 nodes (mainline kernel 6.18.36,
  `siderolabs/rockchip-rknn` extension, `rocket` driver bound).
- A self-built Mesa Teflon delegate (`libteflon.so`) genuinely offloads
  MobileNetV1 inference to the NPU: **all 27 CONV/DWCONV layers** in the
  network showed `supported` in Teflon's op-eligibility table (not a CPU
  fallback).
- Inference latency ~12-15ms per invocation (warm-cache re-runs ~1ms).
- Correct classification result on the canonical "Grace Hopper" test image:
  `0.901961: military uniform` - the standard expected output for
  MobileNetV1-quantized on this well-known test asset.
- Reproduced from a clean pod on two different nodes (`talos-jii-i41` and
  `talos-pz7-vcb`), confirming this isn't node-specific or a fluke.

## No prebuilt package exists yet

Mesa's `rocket` gallium driver + Teflon support merged ahead of the Mesa
25.3 release. As of this writing, no Debian/Ubuntu package ships it
([Debian bug #1070788](https://bugs.debian.org/cgi-bin/bugreport.cgi?id=1070788)
is still open), so `cluster-config/npu-teflon-test.yaml` builds Mesa from
source inside the pod - takes roughly 10-20 minutes on an 8-core RK1 node,
the bulk of it in `apt-get build-dep` and the `ninja` compile.

An unofficial prebuilt `libteflon.so` exists (via the
[Frigate NVR project](https://frigate.video)'s
[community build releases](https://github.com/jimmyhon/frigate-builds)),
which would be much faster to deploy. That path was deliberately not used
here - it's an unsigned third-party binary with no distro/upstream
provenance, run `privileged: true` with device access. Building from Mesa's
own upstream source, even though slower, keeps the supply chain to just
Mesa + Debian's own packages.

## Running the test

```bash
kubectl create namespace npu-teflon-test
kubectl label namespace npu-teflon-test pod-security.kubernetes.io/enforce=privileged
kubectl apply -f cluster-config/npu-teflon-test.yaml
kubectl -n npu-teflon-test logs -f npu-teflon-test
```

Expect the final lines to show the Teflon op-support table (CONV/DWCONV
rows marked `supported`), several `teflon: invoked graph, took ~12-15ms`
lines, and a top classification result of `military uniform` around 0.90
confidence. If NPU offload silently fell back to CPU, the op table would
show `not supported` for the conv layers and/or a
`INFO: Created TensorFlow Lite XNNPACK delegate for CPU` line without the
preceding `Loading external delegate from .../libteflon.so` - that combination
(external delegate loaded, ops all supported, correct result at NPU-typical
latency) is what confirms real acceleration.

### Cleanup

```bash
kubectl delete namespace npu-teflon-test
```

## Resource requirements

The build needs real CPU (Mesa + LLVM is a substantial C/C++ codebase) and
headroom for the apt package cache plus Mesa's build tree:

| Resource | Request | Limit |
|----------|---------|-------|
| CPU | 4 | 8 |
| Memory | 4Gi | 6Gi |

No `nodeSelector` or taint is required - `/dev/accel/accel0` is present on
every node in this cluster. The pod needs `privileged: true` for device
access (the `npu-teflon-test` namespace must be labeled
`pod-security.kubernetes.io/enforce=privileged`, same as `longhorn-system`
and `monitoring` - see [STORAGE.md](STORAGE.md) and
[MONITORING.md](MONITORING.md)).

## Known gotcha

Same PodSecurity issue documented elsewhere in this repo: a fresh namespace
defaults to the cluster's "baseline" PodSecurity admission, which rejects
the privileged pod outright. Label the namespace before applying the
manifest (the commands above already do this).
