# Docker GPU Passthrough

Setup NVIDIA GPU access in Docker containers on Ubuntu 24.04 with RTX 5090.

**Prerequisite:** `nvidia-open` driver installed and working (`nvidia-smi` shows GPU).

## Install Container Toolkit

```bash
sudo apt install nvidia-container-toolkit
```

## Configure Docker Runtime

```bash
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```

This creates/updates `/etc/docker/daemon.json`:
```json
{
    "runtimes": {
        "nvidia": {
            "args": [],
            "path": "nvidia-container-runtime"
        }
    }
}
```

## Verify

```bash
docker run --rm --gpus all nvidia/cuda:12.8.0-base-ubuntu24.04 nvidia-smi
```

## docker-compose GPU Access

Option 1 — deploy resources:
```yaml
services:
  my-service:
    image: nvidia/cuda:12.8.0-runtime-ubuntu24.04
    deploy:
      resources:
        reservations:
          devices:
            - driver: nvidia
              count: all
              capabilities: [gpu]
```

Option 2 — runtime:
```yaml
services:
  my-service:
    image: nvidia/cuda:12.8.0-runtime-ubuntu24.04
    runtime: nvidia
    environment:
      - NVIDIA_VISIBLE_DEVICES=all
```

## Base Images

| Image | Size | Use case |
|---|---|---|
| `nvidia/cuda:12.8.0-base-ubuntu24.04` | ~130 MB | Minimal, runtime only |
| `nvidia/cuda:12.8.0-runtime-ubuntu24.04` | ~400 MB | Runtime + CUDA libs |
| `nvidia/cuda:12.8.0-devel-ubuntu24.04` | ~3 GB | Full, with nvcc |

Images with CUDA 12.8 work with any driver ≥570 (forward compatibility).

## Update

```bash
sudo apt install --only-upgrade nvidia-container-toolkit
sudo nvidia-ctk runtime configure --runtime=docker
sudo systemctl restart docker
```
