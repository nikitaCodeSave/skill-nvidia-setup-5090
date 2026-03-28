# Fresh Install Procedure

Step-by-step commands for installing the full NVIDIA stack from scratch on Ubuntu 24.04 with RTX 5090.

## Prerequisites

```bash
sudo apt update
sudo apt install build-essential gcc python3-dev linux-headers-$(uname -r) -y
```

## Step 1. Clean conflicting packages

```bash
# Remove Ubuntu CUDA (if installed)
sudo apt remove --purge nvidia-cuda-toolkit nvidia-cuda-dev 2>/dev/null
sudo apt autoremove --purge

# Remove runfile driver (if installed)
sudo nvidia-uninstall 2>/dev/null
sudo rm -rf /usr/local/cuda*

# Remove graphics-drivers PPA (if present)
sudo rm /etc/apt/sources.list.d/graphics-drivers-*.sources 2>/dev/null

# Clean orphan kernel module directories
CURRENT=$(uname -r)
for dir in /lib/modules/*/; do
  KVER=$(basename "$dir")
  [ "$KVER" != "$CURRENT" ] && ! dpkg -l "linux-image-$KVER" 2>/dev/null | grep -q ^ii && sudo rm -rf "$dir"
done
```

## Step 2. Add NVIDIA apt repo

```bash
wget https://developer.download.nvidia.com/compute/cuda/repos/ubuntu2404/x86_64/cuda-keyring_1.1-1_all.deb
sudo dpkg -i cuda-keyring_1.1-1_all.deb
sudo apt update
```

## Step 3. Install open driver + CUDA Toolkit

```bash
# nvidia-open = open kernel module (required for Blackwell)
# NEVER use cuda-drivers (proprietary → black screen on RTX 5090)
sudo apt install nvidia-open cuda-toolkit-12-9
```

## Step 4. Pre-reboot checklist

```bash
modinfo /lib/modules/$(uname -r)/updates/dkms/nvidia.ko* 2>/dev/null | grep license
# Must show: Dual MIT/GPL

dkms status | grep "$(uname -r)"
# Must show: nvidia/XXX, <kernel>, x86_64: installed

ls /lib/modules/$(uname -r)/updates/dkms/nvidia*.ko*
# Must show 5 module files
```

If all pass → reboot. If any fails → diagnose, do NOT reboot.

```bash
sudo reboot  # or: sudo shutdown -h now (if MSI POST code 50 issues)
```

## Step 5. Configure environment

```bash
echo 'export PATH=/usr/local/cuda-12.9/bin:$PATH' >> ~/.bashrc
echo 'export LD_LIBRARY_PATH=/usr/local/cuda-12.9/lib64:$LD_LIBRARY_PATH' >> ~/.bashrc
source ~/.bashrc
```

## Step 6. Verify

```bash
cat /proc/driver/nvidia/version | grep -i open  # Must say "Open Kernel Module"
nvidia-smi                                       # RTX 5090, Driver version
nvcc -V                                          # CUDA 12.9
which -a nvcc                                    # Exactly one path
```

## Step 7. PyTorch in venv

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip wheel setuptools
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu129
```

Verify:
```python
import torch
print(torch.__version__)            # 2.11.0+cu129
print(torch.cuda.is_available())    # True
print(torch.cuda.get_device_name(0))# NVIDIA GeForce RTX 5090
```

## Updating

```bash
# Driver
sudo apt install --only-upgrade nvidia-open
# Run pre-reboot checklist before reboot!

# CUDA Toolkit (new major)
sudo apt install cuda-toolkit-12-10  # example
# Update PATH in ~/.bashrc

# PyTorch
pip install --upgrade torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu129
```
