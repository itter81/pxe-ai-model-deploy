# pxe-ai-model-deploy

**PXE 全自动装机 + AI 推理底座一键部署**

*Fully automated bare-metal provisioning with GPU drivers and AI inference stack via PXE*

---

## What is this?

This project automates the full deployment of an AI inference node — from a bare-metal server to a fully operational GPU inference service — with **zero manual intervention**.

一台裸机，PXE 启动，重启后 GPU 驱动 + CUDA + 推理框架全部就绪，全程无需登录目标机器。

---

## Supported Inference Frameworks

| # | Framework | Status |
|---|-----------|--------|
| 01 | [Ollama](./01-ollama/) | ✅ Done |
| 02 | [vLLM](./02-vllm/) | 🚧 WIP |
| 03 | [TGI](./03-tgi/) | 🚧 WIP |
| 04 | [llama.cpp](./04-llamacpp/) | 🚧 WIP |
| 05 | [TensorRT-LLM](./05-trtllm/) | 🚧 WIP |

---

## Requirements

| Item | Version |
|------|---------|
| OS | Ubuntu 22.04.5 LTS |
| Kernel | 5.15.0-119-generic |
| NVIDIA Driver | 580.126.09 |
| CUDA | 13.2 |
| Disk | 700G ~ 1024G NVMe |
| Boot | iPXE + DHCP + HTTP |

---

## Key Achievement

Installing NVIDIA drivers during PXE (chroot environment) is notoriously difficult.

The root cause: the Live kernel has already loaded `nouveau`, which occupies the GPU. The nvidia-installer detects this and **rolls back all installed files** after `modprobe nvidia` fails.

**Solution:** `--skip-module-load`

Found by scanning the full `nvidia-installer --advanced-options` parameter list. This flag skips the `modprobe` step entirely — files are installed and compiled, but not loaded. On first real boot, `nouveau` is blacklisted and `nvidia` loads cleanly.

```bash
/tmp/nvidia.run --silent \
  --no-nouveau-check \
  --skip-module-load \
  --kernel-source-path=/usr/src/linux-headers-5.15.0-119-generic \
  --kernel-name=5.15.0-119-generic
```

---

## Author

**安栋梁 (An Dongliang)**
Infrastructure & AI Ops Engineer | RHCE · HCIE · KYCP

---

## License

MIT
