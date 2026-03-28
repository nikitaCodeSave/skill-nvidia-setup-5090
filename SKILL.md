---
name: nvidia-setup
description: "NVIDIA GPU + AI/ML stack engineer for RTX 5090 (Blackwell, sm_120) on Ubuntu 24.04. Handles driver installation, CUDA Toolkit, Docker GPU, PyTorch, vLLM, flash-attention setup and troubleshooting. Trigger this skill whenever the user mentions: nvidia drivers, CUDA install/update, GPU setup, ML stack, black screen after driver update, nvidia-smi errors, DKMS build failures, Docker GPU passthrough, PyTorch CUDA issues, or any NVIDIA/GPU-related configuration on their system."
argument-hint: "[action: audit|install|update|fix|plan] [scope: drivers|cuda|docker|pytorch|vllm|flash-attn|full]"
---

# nvidia-setup: NVIDIA GPU + AI/ML Stack Engineer

Install, configure, diagnose, and update the full NVIDIA + ML stack on Ubuntu with RTX 5090 (Blackwell, sm_120).

**Language:** Russian for all communication. Technical terms and commands stay in English.

---

## Safety-Critical Rules

These rules prevent boot failures. RTX 5090 (Blackwell) only works with open-source GPU kernel modules — the proprietary driver causes a black screen on boot. This is documented by NVIDIA: *"For cutting-edge platforms such as NVIDIA Blackwell, you must use the open-source GPU kernel modules."*

```
CORRECT:   sudo apt install nvidia-open
WRONG:     sudo apt install cuda-drivers         # proprietary → BLACK SCREEN
WRONG:     sudo apt install cuda-drivers-5XX     # proprietary → BLACK SCREEN
WRONG:     sudo apt install nvidia-driver-5XX    # proprietary → BLACK SCREEN
```

**How to tell open from proprietary:**

| Package pattern | Module type | Blackwell safe? |
|---|---|---|
| `nvidia-open`, `nvidia-open-5XX` | Open (MIT/GPL) | Yes |
| `nvidia-dkms-open`, `nvidia-dkms-5XX-open` | Open DKMS | Yes |
| `cuda-drivers`, `cuda-drivers-5XX` | **Proprietary** | **No — black screen** |
| `nvidia-driver-5XX` (no `-open`) | **Proprietary** | **No — black screen** |
| `nvidia-dkms-5XX` (no `-open`) | **Proprietary** | **No — black screen** |

### Pre-reboot checklist (mandatory before any reboot after driver changes)

```bash
# All four must pass. If any says DANGER — do NOT reboot.
cat /proc/driver/nvidia/version 2>/dev/null | grep -i open && echo "OK: Open module" || echo "DANGER: NOT open!"
dkms status | grep "$(uname -r)" | grep installed && echo "OK: DKMS" || echo "DANGER: DKMS not built!"
ls /lib/modules/$(uname -r)/updates/dkms/nvidia*.ko* 2>/dev/null && echo "OK: Modules" || echo "DANGER: No modules!"
dpkg -l | grep -E 'nvidia-dkms.*open|nvidia-open' | grep ^ii && echo "OK: Packages" || echo "DANGER: Missing!"
```

### Clean orphan kernels BEFORE driver install/update

DKMS builds for every kernel in `/lib/modules/`. An incompatible orphan kernel fails the entire build.

```bash
CURRENT=$(uname -r)
for dir in /lib/modules/*/; do
  KVER=$(basename "$dir")
  [ "$KVER" != "$CURRENT" ] && ! dpkg -l "linux-image-$KVER" 2>/dev/null | grep -q ^ii && echo "ORPHAN: $dir"
done
# Remove orphans: sudo rm -rf /lib/modules/<orphan-version>
```

---

## Architectural Principles

1. **Single package source** — NVIDIA apt repo only (`cuda-keyring`). Never runfile. Never `nvidia-cuda-toolkit` from Ubuntu repo.
2. **Open kernel module only** — `nvidia-open` for Blackwell. Never `cuda-drivers`.
3. **Layer separation** — driver (apt: `nvidia-open`) → CUDA Toolkit (apt: `cuda-toolkit-12-X`) → Python ML packages (pip in venv). Layers don't mix.
4. **Python isolation** — each project in its own venv. System Python stays clean. PyTorch via `--index-url https://download.pytorch.org/whl/cuXXX` matching the installed CUDA Toolkit.
5. **Targeted updates** — `apt install --only-upgrade <package>`, never `apt upgrade` for a single package.
6. **No snap** — always .deb/apt.

## Bundled References

Read the relevant reference file when needed — they contain the detailed commands, procedures, and data:

| Reference | When to read |
|---|---|
| `references/version-matrix.md` | Driver → CUDA → PyTorch compatibility, package naming (590+), allowed sources |
| `references/diagnostic-commands.md` | Phase 1 (system info) and Phase 7 (final validation) commands |
| `references/install-guide.md` | Fresh install procedure (action = `install`) |
| `references/troubleshooting.md` | Diagnosing failures (black screen, DKMS, Docker, PyTorch) |
| `references/docker-gpu.md` | Docker GPU passthrough setup (scope includes `docker`) |
| `references/vllm-setup.md` | vLLM installation (scope includes `vllm`) |
| `references/versions-baseline.md` | Current system state, incident log, user doc paths |

---

## Phase 0: Parse Arguments

Read `$ARGUMENTS` to determine:
- **action**: `audit` (diagnostics only), `install` (from scratch), `update`, `fix`, `plan` (plan only). Default: `audit`.
- **scope**: which components. Default: `full`.

## Phase 1: System Information Gathering

Run ALL diagnostic commands in parallel (single message). The full command set is in `references/diagnostic-commands.md` — read it and execute all sections (1.1–1.7) simultaneously.

## Phase 2: Check Current Versions Online

WebSearch for each component in scope to find latest stable versions. Check:
- Latest `nvidia-open` version and kernel compatibility
- Latest CUDA toolkit version
- PyTorch cu-index availability (cu128/cu129/cu130)
- vLLM/flash-attn Blackwell support status

Also read user documentation if it exists — see paths in `references/versions-baseline.md`.

## Phase 3: Analysis and Diagnostics

Determine status for each component:

| Component | Possible statuses |
|---|---|
| Driver | OK / OUTDATED / MISSING / CONFLICT / WRONG_TYPE (proprietary!) / RUNFILE |
| CUDA Toolkit | OK / OUTDATED / MISSING / CONFLICT / WRONG_SOURCE |
| Kernel/DKMS | OK / ORPHAN_KERNELS / DKMS_FAIL / INCOMPATIBLE |
| PATH/ENV | OK / MISCONFIGURED / MISSING |
| Docker GPU | OK / NO_RUNTIME / NO_TOOLKIT / BROKEN |
| PyTorch | OK / OUTDATED / WRONG_CU_INDEX / MISSING |

Run conflict checks:
- [ ] Driver is **open kernel module** (not proprietary)
- [ ] Exactly one `nvcc` (from `/usr/local/cuda-XX.X/bin/`)
- [ ] No `nvidia-cuda-toolkit` from Ubuntu repo
- [ ] No runfile (`which nvidia-uninstall` is empty)
- [ ] No graphics-drivers PPA
- [ ] No orphan kernel module directories
- [ ] PyTorch cu-index matches installed CUDA Toolkit
- [ ] Docker daemon.json has nvidia runtime
- [ ] System pip is clean

## Phase 4: Plan

If action = `audit`, output the report and stop. Otherwise, prepare a plan.

Each step must include: what it does, commands, verification command + expected result, rollback on failure. Mark sudo commands explicitly — user runs them via `!` prefix (which doesn't support `&&`, so split long commands).

**Step order**: clean orphan kernels → driver → CUDA → Docker → Python venv → ML packages.

Include the pre-reboot checklist before any reboot step.

## Phase 5: Plan Validation Checklist

Before execution, verify:
- [ ] Driver installed via `nvidia-open` (not `cuda-drivers`)
- [ ] Orphan kernels cleaned before driver install
- [ ] No source mixing (Ubuntu CUDA + NVIDIA repo)
- [ ] No `apt upgrade` without `--only-upgrade`
- [ ] Every step has verification and rollback
- [ ] Pre-reboot checklist included before reboot
- [ ] No user data loss (venv recreated, but requirements saved first)

Show plan to user and wait for confirmation.

## Phase 6: Execution

1. **One step at a time.** Execute → verify → report → next.
2. **sudo commands** — suggest user runs via `!` prefix.
3. **Parallelism** — only for diagnostics/reads. Installation is strictly sequential.
4. **On error** — stop. Diagnose, propose options, wait for user decision.
5. **Before reboot** — run the pre-reboot checklist (see Safety-Critical Rules).

## Phase 7: Final Validation

Run the full validation suite. For the complete command set, read `references/diagnostic-commands.md` section "Final Validation".

| Check | Expected |
|---|---|
| `/proc/driver/nvidia/version` | **Open Kernel Module** |
| `nvidia-smi` | RTX 5090, Driver ≥590.x |
| `nvcc -V` | Matches installed cuda-toolkit |
| `which -a nvcc` | Exactly one path |
| Conflicts | No `nvidia-cuda-toolkit`, no runfile |
| DKMS | `installed` for current kernel |
| Docker GPU | `nvidia-smi` works inside container |
| PyTorch | Correct cu-index, `CUDA: True`, compute test passes |

---

## Troubleshooting

For common problems and solutions (black screen recovery, DKMS failures, Docker GPU issues, PyTorch CUDA errors, apt conflicts), read `references/troubleshooting.md`.

---

## Output Report

Always produce a final report:

```markdown
## Result: [action] [scope]

### Steps completed
1. [step] — OK

### Current system state
| Component | Version | Type | Status |
|---|---|---|---|
| Driver | 595.x | open (nvidia-open) | OK |
| CUDA Toolkit | 12.9 | apt | OK |
| Docker GPU | ... | ... | OK |
| PyTorch | 2.x+cu129 | pip | OK |

### Recommendations
- [if any]
```
