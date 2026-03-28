# Troubleshooting Guide

## Table of Contents
- [Black screen after reboot](#black-screen)
- [DKMS build failures](#dkms)
- [nvidia-smi errors](#nvidia-smi)
- [PyTorch CUDA issues](#pytorch)
- [Docker GPU issues](#docker)
- [flash-attn issues](#flash-attn)
- [APT package conflicts](#apt-conflicts)
- [MSI POST code 50](#post-code-50)

---

## <a id="black-screen"></a>Black Screen After Reboot

**Root cause**: proprietary kernel module installed instead of open. Blackwell GPUs reject the proprietary module entirely.

**Recovery steps:**
1. At GRUB menu → Advanced options → Recovery mode
2. Select "root" shell
3. Enable networking:
   ```bash
   dhcpcd <interface>  # or: ip link set <interface> up && dhclient <interface>
   echo "nameserver 8.8.8.8" > /etc/resolv.conf
   ```
4. Remove all NVIDIA packages:
   ```bash
   apt remove --purge '^nvidia-.*'
   apt autoremove --purge
   ```
5. Install open driver:
   ```bash
   apt install nvidia-open
   ```
6. Verify before reboot:
   ```bash
   modinfo /lib/modules/$(uname -r)/updates/dkms/nvidia.ko* | grep license
   # Must show: Dual MIT/GPL
   ```
7. `reboot`

---

## <a id="dkms"></a>DKMS: "Bad return status for module build"

**Cause 1: Orphan kernel in /lib/modules/**
DKMS tries to build for ALL kernels. Old/incompatible kernel directories break the build.

```bash
# Find orphans
for dir in /lib/modules/*/; do
  KVER=$(basename "$dir")
  dpkg -l "linux-image-$KVER" 2>/dev/null | grep -q ^ii || echo "ORPHAN: $dir"
done

# Remove orphans
sudo rm -rf /lib/modules/<orphan-version>

# Retry
sudo dpkg --configure -a
```

**Cause 2: Kernel too new for driver**
Check `make.log` for the actual error:
```bash
tail -50 /var/lib/dkms/nvidia/*/build/make.log
```

If `-Werror` / warnings-as-errors:
- Check `apt-cache policy nvidia-open` for newer version
- Ubuntu repo (`noble-updates`) may have a patched version
- Use `apt install nvidia-open=<specific-version>` if needed

**Cause 3: Missing kernel headers**
```bash
sudo apt install linux-headers-$(uname -r)
```

---

## <a id="nvidia-smi"></a>nvidia-smi Errors

**"command not found"**
→ Driver not installed or PATH issue. Check `lsmod | grep nvidia`. Install `nvidia-open`.

**"NVIDIA-SMI has failed because it couldn't communicate with the NVIDIA driver"**
→ Driver/kernel mismatch after kernel update. Fix:
```bash
sudo apt install --only-upgrade nvidia-open
sudo reboot  # after pre-reboot checklist!
```

---

## <a id="pytorch"></a>PyTorch: "CUDA not available"

**Cause**: torch installed without CUDA or with wrong cu-index.

```bash
# Check what's installed
python -c "import torch; print(torch.__version__)"
# If it says +cpu or wrong +cuXXX:
pip install torch torchvision torchaudio --index-url https://download.pytorch.org/whl/cu129
```

The cu-index must match the installed CUDA Toolkit, not the driver version.

---

## <a id="docker"></a>Docker GPU Issues

**"could not select device driver nvidia"**
→ nvidia-container-toolkit not installed or Docker not configured:
```bash
sudo apt install nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

**nvidia-smi works on host but not in container**
→ Check `/etc/docker/daemon.json` — must contain nvidia runtime. Restart Docker.

**CDI refresh warning during container toolkit install**
→ Normal if driver was just installed/updated. Clears after reboot.

---

## <a id="flash-attn"></a>flash-attn Issues

**Compilation fails**
→ Check `nvcc -V` (need CUDA 12.8+), ensure `MAX_JOBS=6`, install `packaging ninja`.

**On Blackwell (sm_120)**: flash-attn v2 has no native sm_120 kernels. PyTorch SDPA (cuDNN backend) is used automatically as fallback. This is normal and supported by all major frameworks (transformers, vllm). flash-attn v4 (for Blackwell) is in development, waiting for CUTLASS v4.4+.

---

## <a id="apt-conflicts"></a>APT Package Conflicts During Driver Version Change

When switching driver versions (e.g., 570→590), packages may conflict on virtual package names. Solution:

```bash
# Force-remove conflicting packages first
sudo dpkg --purge --force-depends <conflicting-packages>
# Then install the new version
sudo apt install nvidia-open
```

Never have two driver versions installed simultaneously.

---

## <a id="post-code-50"></a>MSI POST Code 50 on Reboot

This is a **hardware** issue with warm reboot (DRAM initialization), unrelated to NVIDIA driver. Use cold boot instead:

```bash
sudo shutdown -h now
# Then press power button to start
```

Alternative: add `reboot=pci` or `reboot=acpi` to `GRUB_CMDLINE_LINUX_DEFAULT` in `/etc/default/grub`, then `sudo update-grub`.
