# Version Matrix and Package Naming

## Table of Contents
- [Driver → CUDA → PyTorch compatibility](#compatibility)
- [Package naming (590+ changes)](#naming)
- [Allowed package sources](#sources)

---

## <a id="compatibility"></a>Driver → CUDA Toolkit → PyTorch

| Driver branch | CUDA Toolkit | Min driver | PyTorch index | sm_120 support |
|---|---|---|---|---|
| 570.x | 12.8 | ≥570.x | `cu128` | PTX JIT |
| 575.x | 12.9 | ≥575.x | `cu129` | Native cubins |
| 580.x | 13.0 | ≥580.x | `cu130` | Native cubins |
| 590.x | 13.1 | ≥590.x | check availability | Native cubins |
| 595.x | 13.2 | ≥595.x | check availability | Native cubins |

**Forward compatibility**: a newer driver supports all older CUDA Toolkits. Driver 595 works with CUDA 12.9, 12.8, etc.

**PyTorch cu-index must match the CUDA Toolkit version**, not the driver. If `cuda-toolkit-12-9` is installed, use `cu129`.

---

## <a id="naming"></a>Package Naming (590+ changes)

Starting with branch 590, NVIDIA changed package naming — branch numbers removed from default package names:

| Before 590 | 590+ | Purpose |
|---|---|---|
| `nvidia-driver-570-open` | `nvidia-open` | Open driver (latest) |
| `cuda-drivers-570` | `cuda-drivers` | Proprietary driver (NOT for Blackwell!) |
| branch in package name | `nvidia-driver-pinning-<ver>` | Pin to specific branch |
| `nvidia-open-570` | `nvidia-open-590` | Branch-specific open (still works) |

---

## <a id="sources"></a>Allowed Package Sources

| Source | What to install | What is FORBIDDEN |
|---|---|---|
| **NVIDIA apt repo** (cuda-keyring) | `nvidia-open`, `cuda-toolkit-12-x`, `nvidia-container-toolkit` | `cuda-drivers` / `cuda-drivers-5XX` (proprietary!) |
| **Ubuntu apt repo** | build-essential, python3-dev, docker.io | `nvidia-cuda-toolkit` (conflicts!) |
| **pip** (PyTorch index) | torch, torchvision, torchaudio | torch from standard PyPI (no CUDA) |
| **pip** (PyPI) | transformers, vllm, flash-attn, other ML libs | — |
| **git source** | vllm (bleeding-edge build) | — |
