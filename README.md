# pxe-ai-model-deploy

**PXE 鍏ㄨ嚜鍔ㄨ鏈?+ AI 鎺ㄧ悊搴曞骇涓€閿儴缃?*

*Fully automated bare-metal provisioning with GPU drivers and AI inference stack via PXE*

---

## What is this?

This project automates the full deployment of an AI inference node 鈥?from a bare-metal server to a fully operational GPU inference service 鈥?with **zero manual intervention**.

涓€鍙拌８鏈猴紝PXE 鍚姩锛岄噸鍚悗 GPU 椹卞姩 + CUDA + 鎺ㄧ悊妗嗘灦鍏ㄩ儴灏辩华锛屽叏绋嬫棤闇€鐧诲綍鐩爣鏈哄櫒銆?

---

## Supported Inference Frameworks

| # | Framework | Status |
|---|-----------|--------|
| 01 | [Ollama](./01-ollama/) | 鉁?Done |
| 02 | [vLLM](./02-vllm/) | 馃毀 WIP |
| 03 | [TGI](./03-tgi/) | 馃毀 WIP |
| 04 | [llama.cpp](./04-llamacpp/) | 馃毀 WIP |
| 05 | [TensorRT-LLM](./05-trtllm/) | 馃毀 WIP |

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

Found by scanning the full `nvidia-installer --advanced-options` parameter list. This flag skips the `modprobe` step entirely 鈥?files are installed and compiled, but not loaded. On first real boot, `nouveau` is blacklisted and `nvidia` loads cleanly.

```bash
/tmp/nvidia.run --silent \
  --no-nouveau-check \
  --skip-module-load \
  --kernel-source-path=/usr/src/linux-headers-5.15.0-119-generic \
  --kernel-name=5.15.0-119-generic
```

---

## Author

**瀹夋爧姊?(An Dongliang)**
Infrastructure & AI Ops Engineer | RHCE 路 HCIE 路 KYCP

---

## License

MIT
