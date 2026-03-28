# vLLM Installation for RTX 5090

**Prerequisite:** Driver ≥575 (`nvidia-open`), CUDA Toolkit 12.9+, Python 3.12.

## Option 1: pip (recommended)

Since v0.18.0, vLLM ships CUDA 12.9 binaries by default and supports RTX 5090 (Blackwell sm_120).

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip wheel setuptools
pip install vllm --extra-index-url https://download.pytorch.org/whl/cu129
```

Note: on Blackwell, Flash Attention 3 backend doesn't work yet. Use `VLLM_FLASH_ATTN_VERSION=2` if issues arise. PyTorch SDPA (cuDNN) is the automatic fallback.

## Option 2: Build from source (bleeding-edge)

```bash
python3 -m venv .venv
source .venv/bin/activate
pip install --upgrade pip wheel setuptools

# Nightly torch (may be needed for source build compatibility)
pip install --pre torch torchvision torchaudio --index-url https://download.pytorch.org/whl/nightly/cu129

git clone https://github.com/vllm-project/vllm.git
cd vllm
python use_existing_torch.py
pip install -r requirements/build.txt

export MAX_JOBS=6  # prevent OOM during compilation (tune for your CPU)
pip install --no-build-isolation -e .
```

Build takes ~20 minutes on Ryzen 9 9950X.

## Update

pip: `pip install --upgrade vllm --extra-index-url https://download.pytorch.org/whl/cu129`

Source: `cd vllm && git pull && pip install --no-build-isolation -e .`

If update breaks — recreate venv from scratch.
