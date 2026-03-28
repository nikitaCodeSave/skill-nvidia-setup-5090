# Versions Baseline (Template)

This file is populated automatically during the first `audit` run. The skill writes the detected system state here as `versions-baseline.md` for reference in future sessions.

If `versions-baseline.md` does not exist, run `/nvidia-setup audit` to create it from this template.

## Current System

| Component | Version | Source | Status |
|---|---|---|---|
| Ubuntu | — | — | — |
| Kernel | — | — | — |
| GPU | — | — | — |
| Driver | — | — | — |
| CUDA Toolkit | — | — | — |
| Docker | — | — | — |
| nvidia-container-toolkit | — | — | — |
| Python | — | — | — |
| cuda-keyring | — | — | — |

## RTX 5090 Specifics

- Architecture: Blackwell
- Compute Capability: sm_120
- **Requires open kernel module** (`nvidia-open`). Proprietary (`cuda-drivers`) causes black screen.
- CUDA 12.8: sm_120 via PTX JIT
- CUDA 12.9+: native sm_120 cubins

## Incident Log

_No incidents recorded._

## User Documentation Paths

_Populate with paths to local guides if they exist._

## NVIDIA Documentation Links

- [NVIDIA Driver Installation Guide — Ubuntu](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/ubuntu.html)
- [NVIDIA Kernel Modules — Open vs Proprietary](https://docs.nvidia.com/datacenter/tesla/driver-installation-guide/kernel-modules.html)
- [CUDA Installation Guide for Linux](https://docs.nvidia.com/cuda/cuda-installation-guide-linux/)
- [NVIDIA Container Toolkit Install Guide](https://docs.nvidia.com/datacenter/cloud-native/container-toolkit/latest/install-guide.html)
