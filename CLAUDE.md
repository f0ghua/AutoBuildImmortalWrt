# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Repository Purpose

AutoBuildImmortalWrt is a GitHub Actions-driven firmware build repository. It builds ImmortalWrt images via ImageBuilder Docker images, with per-target workflow inputs and per-target build scripts.

Primary references:
- `README.md`
- `.github/workflows/*.yml`
- `<target>/build*.sh` + `<target>/imm.config`

## Common Commands

## Build / Run

This repo is mainly operated through `workflow_dispatch` workflows (GitHub Actions).

- Trigger builds from Actions UI (recommended by project README).
- CLI trigger example (if using GitHub CLI):
  - `gh workflow run build-x86-64-24.10.x.yml -f luci_version=24.10.5 -f profile=1024 -f include_docker=yes -f enable_pppoe=no -f custom_router_ip=192.168.100.1`

Local x86-64 ImageBuilder run pattern (mirrors workflow behavior):

```bash
chmod +x x86-64/build24.sh
mkdir -p bin custom
printf '%s\n' '192.168.100.1' > custom/custom_router_ip.txt

docker run --rm -i \
  --user root \
  -v "$PWD/bin:/home/build/immortalwrt/bin" \
  -v "$PWD/files:/home/build/immortalwrt/files" \
  -v "$PWD/custom:/home/build/immortalwrt/files/etc/config" \
  -v "$PWD/x86-64/imm.config:/home/build/immortalwrt/.config" \
  -v "$PWD/shell:/home/build/immortalwrt/shell" \
  -v "$PWD/x86-64/build24.sh:/home/build/immortalwrt/build.sh" \
  -e PROFILE=1024 \
  -e INCLUDE_DOCKER=yes \
  -e ENABLE_PPPOE=no \
  immortalwrt/imagebuilder:x86-64-openwrt-24.10.5 \
  /bin/bash /home/build/immortalwrt/build.sh
```

Artifacts are typically written under `bin/targets/...` and uploaded by workflow release steps.

## Test / Lint

- No dedicated test framework is defined in this repository (no `make test`, `pytest`, `npm test`, etc.).
- No dedicated lint workflow is defined for repository scripts.
- There is no single-test command because there is no test suite in-repo.

## Architecture (Big Picture)

## 1) Workflow-Oriented Build Orchestration

GitHub Actions files in `.github/workflows/` are the entrypoints. They:
- define `workflow_dispatch` inputs (target profile, luci version, docker on/off, PPPoE credentials, etc.),
- optionally mutate package selection,
- run ImageBuilder containers with mounted config/scripts,
- publish built artifacts to releases.

Important examples:
- `build-x86-64-24.10.x.yml`
- `build-wireless-router.yml`
- `build-iso.yml`
- `build-dev-board-by-flippy.yml`
- `build-boxs-by-ophub.yml`

## 2) Per-Target Build Definitions

Each target directory contains target-specific build logic, typically:
- `build*.sh`: assembles package list and calls `make image ...`
- `imm.config`: ImageBuilder config mounted as `.config`

Examples:
- `x86-64/build24.sh`, `x86-64/imm.config`
- `mediatek-filogic/build24.sh`, `mediatek-filogic/imm.config`
- `armsr-armv8/build.sh`, `armsr-armv8/imm.config`

## 3) Shared Customization Layer

- `shell/custom-packages.sh`: optional package toggles (including third-party additions/removals).
- `shell/prepare-packages.sh`: unpacks `.run` bundles and collects `.ipk` files into `packages/`.
- `files/etc/uci-defaults/99-custom.sh`: first-boot runtime customization inside firmware, including:
  - single-NIC vs multi-NIC network behavior,
  - WAN/LAN assignment and optional PPPoE setup,
  - default management IP handling (`/etc/config/custom_router_ip.txt`),
  - firewall adjustments (including docker zone rules when docker exists),
  - ttyd/dropbear accessibility tweaks.

## 4) Platform/Model Routing Data

- `model/*.txt` is used by wireless-router workflow logic to map device profiles to platform/imagebuilder tags.
- `arch/arch.conf` is mounted in some workflows to control package architecture priority.

## 5) ISO and External Packaging Flows

Some workflows are multi-stage:
- Build ImmortalWrt image first,
- then package/deploy through external tools/repos (for example `img-installer`, flippy/ophub-based flows).

## Key Working Rules for Future Changes

- Treat `.github/workflows/*.yml` and target `build*.sh` as a coupled system; workflow env vars/volume mounts must match script expectations.
- When changing first-boot behavior, edit `files/etc/uci-defaults/99-custom.sh` and ensure required files are mounted/generated during build (`custom_router_ip.txt`, `pppoe-settings`).
- When changing package defaults, update both target build scripts and `shell/custom-packages.sh` carefully to avoid conflicting add/remove package flags.

## Additional Repository Guidance Sources

- No repository-level Cursor rules found (`.cursor/rules/`, `.cursorrules`).
- No Copilot instruction file found (`.github/copilot-instructions.md`).
