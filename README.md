# skill-nvidia-setup-5090

Claude Code skill for installing, configuring, diagnosing, and updating the full NVIDIA + AI/ML stack on Ubuntu 24.04 with **RTX 5090 (Blackwell, sm_120)**.

## Why this exists

RTX 5090 (Blackwell architecture) **only works with open-source GPU kernel modules**. Installing the proprietary driver (`cuda-drivers`) results in a **black screen on boot** — a critical and non-obvious pitfall. This skill encodes that knowledge and many other hard-won lessons into an automated, safety-first workflow for Claude Code.

## What it does

| Action | Description |
|---|---|
| `audit` | Full system diagnostics — detect issues without changing anything |
| `install` | Fresh install of the entire stack from scratch |
| `update` | Update specific components to latest stable versions |
| `fix` | Diagnose and fix broken components |
| `plan` | Generate an execution plan without applying changes |

### Supported components (scope)

| Scope | Covers |
|---|---|
| `drivers` | NVIDIA open kernel module (`nvidia-open`) |
| `cuda` | CUDA Toolkit (from NVIDIA apt repo) |
| `docker` | Docker GPU passthrough via nvidia-container-toolkit |
| `pytorch` | PyTorch with correct cu-index in venv |
| `vllm` | vLLM inference server with Blackwell support |
| `flash-attn` | flash-attention (Blackwell compatibility notes) |
| `full` | All of the above (default) |

## Usage in Claude Code

```
/nvidia-setup audit full
/nvidia-setup install drivers cuda
/nvidia-setup update pytorch
/nvidia-setup fix docker
```

## Safety-critical rules

These rules are enforced by the skill to prevent boot failures:

```
CORRECT:   sudo apt install nvidia-open
WRONG:     sudo apt install cuda-drivers         # proprietary -> BLACK SCREEN
WRONG:     sudo apt install cuda-drivers-5XX     # proprietary -> BLACK SCREEN
WRONG:     sudo apt install nvidia-driver-5XX    # proprietary -> BLACK SCREEN
```

**Open vs proprietary packages:**

| Package | Module type | Blackwell safe? |
|---|---|---|
| `nvidia-open`, `nvidia-open-5XX` | Open (MIT/GPL) | Yes |
| `nvidia-dkms-open`, `nvidia-dkms-5XX-open` | Open DKMS | Yes |
| `cuda-drivers`, `cuda-drivers-5XX` | **Proprietary** | **No** |
| `nvidia-driver-5XX` (no `-open`) | **Proprietary** | **No** |

### Pre-reboot checklist

The skill runs this before every reboot after driver changes. If any check fails, reboot is blocked:

```bash
cat /proc/driver/nvidia/version 2>/dev/null | grep -i open && echo "OK" || echo "DANGER"
dkms status | grep "$(uname -r)" | grep installed && echo "OK" || echo "DANGER"
ls /lib/modules/$(uname -r)/updates/dkms/nvidia*.ko* 2>/dev/null && echo "OK" || echo "DANGER"
dpkg -l | grep -E 'nvidia-dkms.*open|nvidia-open' | grep ^ii && echo "OK" || echo "DANGER"
```

## Architectural principles

1. **Single package source** — NVIDIA apt repo only (`cuda-keyring`). No runfiles. No `nvidia-cuda-toolkit` from Ubuntu repo.
2. **Open kernel module only** — `nvidia-open` for Blackwell. Never `cuda-drivers`.
3. **Layer separation** — driver (apt) -> CUDA Toolkit (apt) -> Python ML packages (pip in venv). Layers don't mix.
4. **Python isolation** — each project in its own venv. PyTorch via `--index-url` matching the installed CUDA Toolkit.
5. **Targeted updates** — `apt install --only-upgrade <package>`, never `apt upgrade`.
6. **No snap** — always .deb/apt.

## Workflow phases

```
Phase 0: Parse arguments (action + scope)
Phase 1: System information gathering (parallel diagnostics)
Phase 2: Check latest stable versions online
Phase 3: Analysis — status for each component (OK / OUTDATED / MISSING / CONFLICT / ...)
Phase 4: Generate execution plan with verification and rollback for each step
Phase 5: Plan validation checklist
Phase 6: Step-by-step execution with user confirmation
Phase 7: Final validation suite
```

## Version compatibility matrix

| Driver | CUDA Toolkit | PyTorch index | sm_120 support |
|---|---|---|---|
| 570.x | 12.8 | `cu128` | PTX JIT |
| 575.x | 12.9 | `cu129` | Native cubins |
| 580.x | 13.0 | `cu130` | Native cubins |
| 590.x | 13.1 | check | Native cubins |
| 595.x | 13.2 | check | Native cubins |

Forward compatibility: a newer driver supports all older CUDA Toolkits. PyTorch cu-index must match the **CUDA Toolkit**, not the driver.

## Troubleshooting

The skill handles these common issues automatically:

- **Black screen after reboot** — proprietary module installed instead of open; recovery via GRUB
- **DKMS build failures** — orphan kernels, missing headers, kernel too new
- **nvidia-smi errors** — driver/kernel mismatch after kernel update
- **PyTorch "CUDA not available"** — wrong cu-index or CPU-only torch
- **Docker GPU not working** — missing nvidia-container-toolkit or daemon.json config
- **APT conflicts** — mixed package sources, version conflicts
- **MSI POST code 50** — hardware warm reboot issue (use cold boot)

## Project structure

```
README.md
nvidia-setup/                            # <- copy this folder to ~/.claude/skills/
  SKILL.md                               # Main skill definition
  references/
    version-matrix.md                    # Driver/CUDA/PyTorch compatibility
    diagnostic-commands.md               # Phase 1 & 7 command sets
    install-guide.md                     # Fresh install procedure
    troubleshooting.md                   # Common problems and solutions
    docker-gpu.md                        # Docker GPU passthrough setup
    vllm-setup.md                        # vLLM installation guide
    versions-baseline.template.md        # Template for tracking system state
```

## Installation

Clone the repo and copy the `nvidia-setup` folder into your Claude Code skills directory:

```bash
git clone https://github.com/nikitaCodeSave/skill-nvidia-setup-5090.git
cp -r skill-nvidia-setup-5090/nvidia-setup ~/.claude/skills/
```

Or symlink if you prefer to pull updates via `git pull`:

```bash
ln -s "$(pwd)/skill-nvidia-setup-5090/nvidia-setup" ~/.claude/skills/nvidia-setup
```

After that, the `/nvidia-setup` command becomes available in Claude Code.

## License

MIT
