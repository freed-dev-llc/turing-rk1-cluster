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
- INSTALLATION docs: Talos version bumped from v1.11.6 to v1.13.2 (#25).
- MetalLB pool range reconciled to ground-truth `10.10.88.80-89` across all docs (was inconsistent: `80-89` in 4 places, `80-99` in 4) (#23).
- STORAGE.md hostnames switched from auto-generated `talos-0ow-v7t`-style IDs to `turing-w1/w2/w3` to match the documented hostname patch (#23).
- INSTALLATION storage table corrected — Node 1 control plane has no NVMe by design (#23).
- `docs/README.md` scripts table now lists `deploy-talos-cluster.sh` and `talos-cluster-status.sh` (#23).

### Fixed

- License badge: MIT → Apache 2.0 to match the actual `LICENSE` file (#25).
- `docs/INSTALLATION.md` ingress-nginx URL was pinned to `v1.12.0-beta.0`; bumped to `v1.13.3` GA (#24).
- `~/Code/turing-rk1-cluster` hardcoded paths in `CLUSTER_PLAN.md` and `docs/QUICKREF.md` generalized to `$REPO_ROOT` (#24).
