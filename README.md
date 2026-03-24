# pxe-ai-model-deploy

GPU 推理节点自动化交付体系，从裸机到推理服务就绪，零人工干预

Automated GPU inference node delivery — bare-metal to inference-ready, zero manual intervention
---

## What is this?

This project automates the full deployment of an AI inference node — from a bare-metal server to a fully operational GPU service — with zero manual intervention.
Starting with a single PXE boot, the target machine automatically installs GPU drivers, CUDA, and the inference framework upon reboot, requiring no login.
⚠️ Note: This is currently a Proof of Concept (PoC) to demonstrate feasibility. Certain steps (e.g., downloading the Qwen/Qwen2.5-0.5B model) are for demonstration purposes and should be evaluated/adjusted for formal production environments.
本项目实现了 AI 推理节点的全自动化部署——从裸金属服务器到完全可用的 GPU 服务，全程零人工干预。
只需一次 PXE 启动，目标机器重启后即可自动装好 GPU 驱动、CUDA 及推理框架，无需登录。
⚠️ 注意： 当前版本仅为验证可行性的概念演示 (PoC)。部分步骤（如下载 Qwen/Qwen2.5-0.5B 模型）仅供演示，在正式生产环境中执行时请自行评估和调整

---

## Supported Inference Frameworks

| # | Framework | Status |
|---|-----------|--------|
| 01 | [Ollama](./01-ollama/) | ✅ Done |
| 02 | [vLLM](./02-vllm/) | ✅ Done |
| 03 | [llama.cpp](./03-llamacpp/) | ✅ Done |
| 04 | [TGI](./04-tgi/) | 🚧 WIP |
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
