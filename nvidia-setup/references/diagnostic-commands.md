# Diagnostic Commands Reference

All commands below should be run in parallel (single message) during Phase 1.

## 1.1 Hardware and OS

```bash
nvidia-smi 2>&1 || echo "NVIDIA_SMI_MISSING"
lspci | grep -i nvidia
uname -r
lsb_release -ds
cat /proc/cpuinfo | grep "model name" | head -1
free -h | head -2
```

## 1.2 Driver and CUDA

```bash
# Driver type (open/proprietary)
cat /proc/driver/nvidia/version 2>/dev/null || echo "NO_DRIVER"
dpkg -l | grep -iE 'nvidia-driver|nvidia-open|nvidia-dkms'
modinfo nvidia 2>/dev/null | grep -E '^version|^license'

# Runfile detection
which nvidia-uninstall 2>/dev/null && echo "WARNING: runfile driver detected!"

# CUDA
nvcc -V 2>&1 || echo "NO_NVCC"
which -a nvcc 2>/dev/null
ls -la /usr/local/cuda* 2>/dev/null || echo "NO_CUDA_LOCAL"
dpkg -l | grep -iE 'cuda-toolkit|cuda-drivers|cuda-keyring|nvidia-open'

# Conflicts
dpkg -l | grep nvidia-cuda-toolkit 2>/dev/null && echo "CONFLICT: Ubuntu CUDA toolkit installed!"
```

## 1.3 Kernels and DKMS

```bash
echo "Current kernel: $(uname -r)"
dkms status 2>/dev/null

# Orphan kernel modules (no matching linux-image)
for dir in /lib/modules/*/; do
  KVER=$(basename "$dir")
  dpkg -l "linux-image-$KVER" 2>/dev/null | grep -q ^ii || echo "ORPHAN: $dir"
done

dpkg -l | grep linux-image | grep ^ii | awk '{print $2, $3}'
```

## 1.4 Environment Variables

```bash
echo "PATH=$PATH"
echo "LD_LIBRARY_PATH=$LD_LIBRARY_PATH"
grep -E 'cuda|nvidia' ~/.bashrc ~/.profile 2>/dev/null
```

## 1.5 Docker and Container Toolkit

```bash
docker --version 2>/dev/null || echo "NO_DOCKER"
dpkg -l | grep nvidia-container
cat /etc/docker/daemon.json 2>/dev/null || echo "NO_DAEMON_JSON"
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi 2>&1 | head -5 || echo "DOCKER_GPU_FAIL"
```

## 1.6 Python and ML Environments

```bash
python3 --version
which python3
pip list 2>/dev/null | grep -iE 'torch|cuda' || echo "SYSTEM_PIP_CLEAN"

# Find all venvs with torch
for dir in ~/PROJECTS/*/; do
  if [ -f "$dir/.venv/bin/python" ]; then
    torch=$("$dir/.venv/bin/python" -c "import torch; print(torch.__version__, '|', 'CUDA:', torch.cuda.is_available())" 2>/dev/null)
    [ -n "$torch" ] && echo "$(basename $dir): $torch"
  fi
done
```

## 1.7 APT Sources

```bash
grep -r nvidia /etc/apt/sources.list.d/ 2>/dev/null
dpkg -l | grep cuda-keyring
ls /etc/apt/sources.list.d/ | grep -iE 'nvidia|cuda|graphics'
```

---

## Final Validation (Phase 7)

Run after all steps are complete:

```bash
# 1. Driver — must be open module
cat /proc/driver/nvidia/version | grep -i open
nvidia-smi

# 2. CUDA
nvcc -V
which -a nvcc  # exactly one

# 3. No conflicts
dpkg -l | grep nvidia-cuda-toolkit  # must be empty
which nvidia-uninstall 2>/dev/null  # must be empty (no runfile)

# 4. DKMS
dkms status  # nvidia installed for current kernel

# 5. Docker GPU
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi

# 6. PyTorch test (substitute correct cu-index)
python3 -m venv /tmp/test-gpu-venv
/tmp/test-gpu-venv/bin/pip install -q torch --index-url https://download.pytorch.org/whl/cu129
/tmp/test-gpu-venv/bin/python -c "
import torch
assert torch.cuda.is_available(), 'CUDA not available!'
print(f'OK: {torch.__version__} | GPU: {torch.cuda.get_device_name(0)}')
print(f'Compute capability: {torch.cuda.get_device_capability(0)}')
t = torch.randn(1000, 1000, device='cuda')
r = t @ t.T
print(f'GPU compute test: OK ({r.shape})')
"
rm -rf /tmp/test-gpu-venv

# 7. Clean apt sources
grep -r nvidia /etc/apt/sources.list.d/
```
