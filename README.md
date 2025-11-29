# TII UAE pKVM Manifest for Jetson

Google repo manifest for building pKVM-enabled NVIDIA Jetson BSP.

## Overview

This repository contains manifests that define all source repositories needed for:
- NVIDIA Jetson Linux BSP (L4T 36.4.4)
- Custom pKVM kernel with SMMUv2 support for Tegra234
- pKVM workspace (scripts, documentation, autopilot testing)

## Available Manifests

| Manifest | Description |
|----------|-------------|
| `pkvm-jetson-36.4.4.xml` | pKVM-enabled Jetson AGX Orin (includes custom kernel and drivers) |
| `jetson-36.4.4.xml` | Vanilla NVIDIA BSP (L4T 36.4.4) |
| `common.xml` | TII UAE shared infrastructure (remotes, workspace) |

### Manifest Structure

The manifests use an inheritance pattern similar to [tii_sel4_manifest](https://github.com/tiiuae/tii_sel4_manifest):

```
pkvm-jetson-36.4.4.xml
    │
    ├── <include name="jetson-36.4.4.xml" />   (vanilla NVIDIA BSP)
    ├── <include name="common.xml" />           (remotes + workspace)
    │
    ├── <remove-project> + <project> linux      (pKVM kernel replaces Ubuntu)
    ├── <extend-project> linux-nv-oot           (custom platform drivers)
    ├── <extend-project> linux-nvgpu            (GPU driver for 6.17)
    ├── <remove-project> + <project> nvdisplay  (display driver for 6.17)
    ├── <remove-project> + <project> atf        (pKVM-enabled ATF)
    └── <extend-project> hwpm, nvethernetrm...  (rel-38 branches)
```

## Quick Start

### Prerequisites

```bash
sudo apt update
sudo apt install repo python3-pycryptodome python3-pyelftools
```

### Fresh Setup

```bash
# Create workspace (can be any directory)
mkdir -p ~/pkvm
cd ~/pkvm
export WORKSPACE=$(pwd)

# Initialize and sync
repo init -u https://github.com/tiiuae/tiiuae-pkvm-manifest.git -b main -m pkvm-jetson-36.4.4.xml
repo sync -j4

# Setup BSP (downloads toolchain, unpacks rootfs, creates env.sh)
./scripts/jetson-bsp-setup.sh

# Source environment
. ${WORKSPACE}/env.sh

# Optional: add to ~/.bashrc for automatic environment setup
echo ". ${WORKSPACE}/env.sh" >> ~/.bashrc
```

### Directory Structure After Sync (pKVM)

```
~/pkvm/                              # workspace root (${WORKSPACE})
├── jetson-pkvm/                     # pKVM workspace repo (actual files)
├── Linux_for_Tegra/
│   └── source/                      # BSP sources
│       ├── kernel/
│       │   └── linux/               # Kernel (pKVM or Ubuntu depending on manifest)
│       ├── nvgpu/                   # NVIDIA GPU driver
│       ├── nvidia-oot/              # NVIDIA out-of-tree drivers (custom fork)
│       ├── nvdisplay/               # Display driver
│       ├── hwpm/                    # Hardware Performance Monitor
│       └── tegra/
│           └── optee-src/           # OP-TEE and ATF
│
│ (symlinks to jetson-pkvm/)
├── scripts -> jetson-pkvm/scripts
├── docs -> jetson-pkvm/docs
├── patches -> jetson-pkvm/patches
├── autopilot -> jetson-pkvm/autopilot
├── README.md -> jetson-pkvm/README.md
└── CLAUDE.md -> jetson-pkvm/CLAUDE.md
```

## Remotes

| Remote | URL | Description |
|--------|-----|-------------|
| `nv-tegra` | `git://nv-tegra.nvidia.com/` | NVIDIA public git server |
| `tiiuae` | `https://github.com/tiiuae/` | TII UAE organization |
| `hlyytine` | `https://github.com/hlyytine/` | Personal forks with custom modifications |

## Custom Repositories (pKVM manifest)

These repositories contain modifications for pKVM GPU virtualization:

### Personal Forks (hlyytine remote)

| Path | Repository | Branch | Description |
|------|------------|--------|-------------|
| `jetson-pkvm` | `hlyytine:jetson-pkvm` | `claude` | Workspace: scripts, docs, patches |
| `kernel/linux` | `hlyytine:linux` | `tegra/pkvm-mainline-6.17-smmu-backup` | pKVM kernel with SMMUv2 |
| `nvidia-oot` | `hlyytine:linux-nv-oot` | `rel-38-for-nvgpu-rel-36` | Platform drivers |
| `nvgpu` | `hlyytine:linux-nvgpu` | `v6.17` | GPU driver (kernel 6.17 compat) |
| `nvdisplay` | `hlyytine:nv-kernel-display-driver` | `v6.17` | Display driver (kernel 6.17 compat) |

### TII UAE Organization (tiiuae remote)

| Path | Repository | Branch | Description |
|------|------------|--------|-------------|
| `tegra/optee-src/atf` | `tiiuae:atf-nvidia-jetson` | `l4t/l4t-r36.4.4-pkvm2` | ATF with pKVM support |

### Updated NVIDIA Repos (nv-tegra remote, rel-38 branch)

| Path | Repository | Branch |
|------|------------|--------|
| `hwpm` | `linux-hwpm` | `rel-38` |
| `nvethernetrm` | `kernel/nvethernetrm` | `rel-38` |
| `hardware/nvidia/t23x/nv-public` | `device/hardware/nvidia/t23x-public-dts` | `rel-38` |
| `hardware/nvidia/tegra/nv-public` | `device/hardware/nvidia/tegra-public-dts` | `rel-38` |

## NVIDIA Stock Repositories

The base manifest includes 35 repositories from NVIDIA's `nv-tegra.nvidia.com` server, synced to tag `jetson_36.4.4`:

- **Kernel**: Ubuntu kernel, device trees, DTC
- **Drivers**: nvgpu, nvidia-oot, nvdisplay, hwpm
- **Secure World**: OP-TEE, ARM Trusted Firmware
- **Multimedia**: GStreamer plugins, V4L2 libraries
- **Other**: CUDA samples, NvSci, SPE FreeRTOS

## Extending the Base Manifest

To create your own derived manifest:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<manifest>
  <!-- Include base NVIDIA BSP -->
  <include name="jetson-36.4.4.xml" />

  <!-- Add your remote (org-level URL enables extend-project) -->
  <remote name="myremote" fetch="https://github.com/myorg/" />

  <!-- Add new projects -->
  <project name="my-kernel"
           path="Linux_for_Tegra/source/kernel/linux"
           remote="myremote"
           revision="my-branch" />

  <!-- Override existing projects (change remote/revision) -->
  <extend-project name="linux-nv-oot"
                  remote="myremote"
                  revision="my-custom-branch" />
</manifest>
```

**Note:** Using an org-level remote URL (e.g., `https://github.com/myorg/`) allows `extend-project` to redirect existing projects to your fork, as long as the repo name matches (e.g., `linux-nv-oot`).

## Symlinks

The `common.xml` manifest uses `<linkfile>` elements to create symlinks from `jetson-pkvm/` to the workspace root:

```xml
<project name="jetson-pkvm" path="jetson-pkvm" ...>
  <linkfile src="scripts" dest="scripts" />
  <linkfile src="docs" dest="docs" />
  ...
</project>
```

The Ethernet driver symlink is also created automatically:
```
Linux_for_Tegra/source/nvidia-oot/drivers/net/ethernet/nvidia/nvethernet/nvethernetrm
  -> ../../../../../../nvethernetrm
```

## Updating Repositories

```bash
cd ~/pkvm  # workspace root

# Sync all repos to latest
repo sync -j4

# Check status of all repos
repo status

# See which repos have local changes
repo diff
```

## Related Documentation

- [jetson-pkvm](https://github.com/hlyytine/jetson-pkvm) - Main workspace with build instructions
- [NVIDIA L4T Documentation](https://docs.nvidia.com/jetson/)
- [Google Repo Tool](https://gerrit.googlesource.com/git-repo/)

## License

Manifest files are provided under BSD-3-Clause license.
Individual repositories have their own licenses.
