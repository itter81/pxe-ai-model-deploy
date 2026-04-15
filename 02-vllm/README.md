# 02-vllm | PXE Automated Deploy - vLLM Inference Stack

基于 iPXE + Ubuntu Autoinstall，实现从裸机到 vLLM OpenAI 兼容推理服务就绪的零人工干预交付。

Automated bare-metal deployment of vLLM inference stack via iPXE + Ubuntu Autoinstall. Zero manual intervention required.

---

## Architecture Overview

```
Client (bare-metal server)
    |
    | 1. Power on, PXE boot
    v
DHCP + TFTP + Nginx (192.168.70.230)
    |
    | 2. iPXE two-stage boot -> Ubuntu Autoinstall
    v
Ubuntu Autoinstall late-commands:
    |
    | 3. NVIDIA Driver + CUDA  (same as 01-ollama)
    | 4. pip3 offline install
    | 5. python3-dev offline install (required for vLLM runtime CUDA compilation)
    | 6. vLLM offline pip install (180 packages, ~4.8GB)
    | 7. Test model download + systemd service
    v
Reboot -> vLLM serving OpenAI-compatible API on :8000
```

---

## Server Directory Structure

```
payload/
├── base/
│   └── gcc-pkgs/
│       └── gcc-pkgs.tar.gz         # Same as 01-ollama, 45 deb packages
└── gpu-inference/
    ├── nvidia-driver/               # Same as 01-ollama
    ├── cuda/                        # Same as 01-ollama
    ├── models/
    │   └── qwen2.5-0.5b.tar.gz     # Test model (experiment only, remove in production)
    └── vllm/
        ├── current -> vllm-0.17.1
        ├── pip3-pkgs/               # pip3 installer (not version-specific)
        │   ├── python3-pip_22.0.2+dfsg-1ubuntu0.7_all.deb
        │   └── python3-wheel_0.37.1-2ubuntu0.22.04.1_all.deb
        ├── python3-dev-pkgs/        # python3-dev and all dependencies (not version-specific)
        │   ├── javascript-common_*.deb
        │   ├── libexpat1_*.deb
        │   ├── libexpat1-dev_*.deb
        │   ├── libjs-jquery_*.deb
        │   ├── libjs-sphinxdoc_*.deb
        │   ├── libjs-underscore_*.deb
        │   ├── libpython3.10_*.deb
        │   ├── libpython3.10-dev_*.deb
        │   ├── libpython3.10-minimal_*.deb
        │   ├── libpython3.10-stdlib_*.deb
        │   ├── libpython3-dev_*.deb
        │   ├── python3.10_*.deb
        │   ├── python3.10-dev_*.deb
        │   ├── python3.10-minimal_*.deb
        │   ├── python3-dev_*.deb
        │   └── zlib1g-dev_*.deb
        └── vllm-0.17.1/            # Version-specific files
            ├── vllm-pkgs.tar.gz    # 180 pip packages, ~4.8GB
            └── vllm.service        # systemd service
```

**pip3-pkgs and python3-dev-pkgs are NOT version-specific** — placed at `vllm/` root, not inside version directory. Only `vllm-pkgs.tar.gz` and `vllm.service` change with version.

---

## Payload Preparation

### pip3-pkgs

**Why needed:** Ubuntu 22.04 minimal install does not include pip3.

**How to prepare:**

```bash
apt-get download python3-pip python3-wheel
# Result: 2 deb files
mkdir pip3-pkgs
mv python3-pip*.deb python3-wheel*.deb pip3-pkgs/
tar -zcf pip3-pkgs.tar.gz pip3-pkgs/
```

### python3-dev-pkgs

**Why needed:** vLLM dynamically compiles CUDA extensions at startup (even with `--attention-backend TRITON_ATTN`). Requires `Python.h` header file from python3-dev.

**How to prepare (on Ubuntu 22.04 with internet):**

```bash
mkdir /tmp/python3-dev-pkgs && cd /tmp/python3-dev-pkgs
apt-get install --download-only python3-dev
cp /var/cache/apt/archives/*.deb /tmp/python3-dev-pkgs/
# Result: 16 deb files
tar -zcf python3-dev-pkgs.tar.gz python3-dev-pkgs/
```

### vllm-pkgs.tar.gz

**Why needed:** vLLM and all its Python dependencies (torch, cuda libraries, etc.) for fully offline pip install.

**How to prepare (on Ubuntu 22.04, Python 3.10, with internet):**

```bash
# Set Chinese mirror for speed
pip3 config set global.index-url https://mirrors.aliyun.com/pypi/simple/
pip3 config set global.trusted-host mirrors.aliyun.com

# Fix DNS if needed
cat > /etc/systemd/resolved.conf << 'EOF'
[Resolve]
DNS=223.5.5.5 114.114.114.114
EOF
systemctl restart systemd-resolved

# Download all packages (do NOT use /tmp - cleared on reboot)
mkdir -p /data/vllm-pkgs
nohup pip download vllm -d /data/vllm-pkgs/ > /data/vllm-download.log 2>&1 &
tail -f /data/vllm-download.log

# Package
tar -zcf vllm-pkgs.tar.gz -C /data vllm-pkgs/
# Result: ~4.8GB, 180 packages
```

### vllm.service

```ini
[Unit]
Description=vLLM OpenAI Compatible API Server
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
Environment="PATH=/usr/local/cuda/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin"
Environment="LD_LIBRARY_PATH=/usr/local/cuda/lib64"

# NOTE: Parameters below are for GTX 1650 (compute capability 7.5) only
# For H200/A100 production: remove --attention-backend, --gpu-memory-utilization, --max-num-seqs
ExecStart=/usr/bin/python3 -m vllm.entrypoints.openai.api_server \
    --model /data/models/qwen2.5-0.5b \
    --host 0.0.0.0 \
    --port 8000 \
    --attention-backend TRITON_ATTN \
    --gpu-memory-utilization 0.7 \
    --max-num-seqs 4
Restart=always
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### Test model (experiment only)

**Why:** vLLM requires a model to start. Small model used for PXE flow validation only. Remove in production — models should be managed separately.

```bash
# Download Qwen2.5-0.5B (~750MB after compression)
pip3 install modelscope
modelscope download --model Qwen/Qwen2.5-0.5B-Instruct --local_dir /data/models/qwen2.5-0.5b

# Package
cd /data/models/qwen2.5-0.5b
tar -zcf ../qwen2.5-0.5b.tar.gz .
# Result: ~750MB
```

---

## user-data (Cloud-init Autoinstall)

File: `/data/pxe/iso/wubantu/ubuntu-22.04.5-custom/cloud-init/user-data-vllm`

```yaml
#cloud-config
# Live installation SSH access for log monitoring
# ssh installer@<ip>  password: ubuntu
chpasswd:
  list:
    - installer:ubuntu
  expire: false

autoinstall:
  version: 1
  refresh-installer:
    update: false

  identity:
    hostname: ubuntu
    username: ubuntu
    password: "$6$IpXEZfUOrF4R7lPi$VmM3g0NroxoKrjB3EgzamMwn9TQTZ6vlmz3Yn0m5uhQJcqTDwk/2rcayY2scQvYzK999afy91HbUlgT0j8rwN1"
    realname: ubuntu

  locale: en_US.UTF-8
  keyboard:
    layout: us
  timezone: Asia/Shanghai

  network:
    network:
      version: 2
      ethernets:
        main_iface:
          match:
            name: en*
          dhcp4: true

  storage:
    config:
      - type: disk
        id: system_disk
        match:
          size: "700G..1024G"
        ptable: gpt
        wipe: superblock-recursive
      - type: partition
        id: efi_part
        device: system_disk
        size: 1G
        flag: boot
        grub_device: true
      - type: format
        id: efi_fmt
        volume: efi_part
        fstype: vfat
      - type: mount
        id: efi_mount
        device: efi_fmt
        path: /boot/efi
      - type: partition
        id: boot_part
        device: system_disk
        size: 1G
      - type: format
        id: boot_fmt
        volume: boot_part
        fstype: ext4
      - type: mount
        id: boot_mount
        device: boot_fmt
        path: /boot
      - type: partition
        id: root_part
        device: system_disk
        size: -1
      - type: format
        id: root_fmt
        volume: root_part
        fstype: xfs
      - type: mount
        id: root_mount
        device: root_fmt
        path: /

  apt:
    preserve_sources_list: false
    primary: []
    fallback: offline-install
    conf: |
      Acquire::http::Timeout "2";
      Acquire::Retries "0";

  ssh:
    install-server: true
    allow-pw: true

  late-commands:
    # --- Base system config ---
    - curtin in-target -- sh -c "echo 'ubuntu ALL=(ALL) NOPASSWD:ALL' > /etc/sudoers.d/ubuntu"
    - curtin in-target -- sed -i 's/GRUB_TIMEOUT_STYLE=hidden/GRUB_TIMEOUT_STYLE=menu/' /etc/default/grub
    - curtin in-target -- sed -i 's/GRUB_TIMEOUT=0/GRUB_TIMEOUT=5/' /etc/default/grub
    - curtin in-target -- update-grub
    - curtin in-target -- sh -c 'echo "root:123456" | chpasswd'
    - curtin in-target -- passwd -u root
    - curtin in-target -- sh -c "sed -i '/PermitRootLogin/d' /etc/ssh/sshd_config && echo 'PermitRootLogin yes' >> /etc/ssh/sshd_config"
    - curtin in-target -- sh -c "sed -i '/PasswordAuthentication/d' /etc/ssh/sshd_config && echo 'PasswordAuthentication yes' >> /etc/ssh/sshd_config"
    - curtin in-target -- sh -c "sed -i '/UseDNS/d' /etc/ssh/sshd_config && echo 'UseDNS no' >> /etc/ssh/sshd_config"
    - curtin in-target -- sh -c "sed -i '/GSSAPIAuthentication/d' /etc/ssh/sshd_config && echo 'GSSAPIAuthentication no' >> /etc/ssh/sshd_config"

    # --- GPU driver preparation ---
    - mount --bind /proc /target/proc
    - mount --bind /sys /target/sys
    - mount --bind /dev /target/dev
    - curtin in-target -- sh -c "echo 'blacklist nouveau' > /etc/modprobe.d/blacklist-nouveau.conf"
    - curtin in-target -- sh -c "echo 'options nouveau modeset=0' >> /etc/modprobe.d/blacklist-nouveau.conf"
    - curtin in-target -- update-initramfs -u
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/base/gcc-pkgs.tar.gz -O /target/tmp/gcc-pkgs.tar.gz
    - curtin in-target -- tar xzf /tmp/gcc-pkgs.tar.gz -C /tmp/
    - curtin in-target -- sh -c "dpkg -i /tmp/gcc-pkgs/*.deb"

    # --- NVIDIA driver ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/nvidia-driver/current/nvidia.run -O /target/tmp/nvidia.run
    - curtin in-target -- chmod +x /tmp/nvidia.run
    - curtin in-target -- sh -c "/tmp/nvidia.run --silent --no-nouveau-check --skip-module-load --kernel-source-path=/usr/src/linux-headers-5.15.0-119-generic --kernel-name=5.15.0-119-generic 2>&1 | tee /var/log/nvidia-install.log || true"
    - curtin in-target -- sh -c "apt-mark hold linux-image-5.15.0-119-generic linux-headers-5.15.0-119-generic linux-image-generic linux-headers-generic linux-generic"

    # --- CUDA ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/cuda/current/cuda.run -O /target/tmp/cuda.run
    - curtin in-target -- chmod +x /tmp/cuda.run
    - curtin in-target -- sh -c "/tmp/cuda.run --silent --toolkit --no-drm --tmpdir=/tmp || true"
    - curtin in-target -- update-initramfs -u
    - curtin in-target -- sh -c "echo 'export PATH=/usr/local/cuda/bin:\$PATH' > /etc/profile.d/cuda.sh"
    - curtin in-target -- sh -c "echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:\$LD_LIBRARY_PATH' >> /etc/profile.d/cuda.sh"

    # --- pip3 (offline) ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/vllm/pip3-pkgs.tar.gz -O /target/tmp/pip3-pkgs.tar.gz
    - curtin in-target -- tar xzf /tmp/pip3-pkgs.tar.gz -C /tmp/
    - curtin in-target -- sh -c "dpkg -i /tmp/pip3-pkgs/*.deb || true"

    # --- python3-dev (required for vLLM CUDA compilation at runtime) ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/vllm/python3-dev-pkgs.tar.gz -O /target/tmp/python3-dev-pkgs.tar.gz
    - curtin in-target -- tar xzf /tmp/python3-dev-pkgs.tar.gz -C /tmp/
    - curtin in-target -- sh -c "dpkg -i /tmp/python3-dev-pkgs/*.deb || true"

    # --- vLLM (offline pip install) ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/vllm/current/vllm-pkgs.tar.gz -O /target/tmp/vllm-pkgs.tar.gz
    - curtin in-target -- tar xzf /tmp/vllm-pkgs.tar.gz -C /tmp/
    - curtin in-target -- pip3 install --no-index --find-links=/tmp/vllm-pkgs vllm

    # --- Test model (experiment only, remove in production) ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/models/qwen2.5-0.5b.tar.gz -O /target/tmp/qwen2.5-0.5b.tar.gz
    - curtin in-target -- mkdir -p /data/models/qwen2.5-0.5b
    - curtin in-target -- tar xzf /tmp/qwen2.5-0.5b.tar.gz -C /data/models/qwen2.5-0.5b/

    # --- vLLM systemd service ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/vllm/current/vllm.service -O /target/etc/systemd/system/vllm.service
    - curtin in-target -- chmod 644 /etc/systemd/system/vllm.service
    - curtin in-target -- systemctl enable vllm

  shutdown: reboot
```

---

## Verification

After installation completes and system reboots:

```bash
# NVIDIA driver
nvidia-smi

# CUDA
nvcc --version

# pip3
pip3 --version

# vLLM
python3 -c "import vllm; print(vllm.__version__)"

# Service status
systemctl status vllm

# API test
curl http://localhost:8000/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"/data/models/qwen2.5-0.5b","messages":[{"role":"user","content":"你好"}]}'
```

Expected:

```json
{"choices":[{"message":{"content":"你好！很高兴为您服务。有什么我能帮助您的吗？"}}]}
```

---

## GPU Compatibility Notes

| GPU | Compute Capability | FlashInfer | BF16 | Notes |
|-----|--------------------|-----------|------|-------|
| GTX 1650 | 7.5 | ✗ | ✗ | Use `--attention-backend TRITON_ATTN`, limit memory |
| RTX 3090 | 8.6 | ✓ | ✓ | No special params needed |
| A100 | 8.0 | ✓ | ✓ | No special params needed |
| H100/H200 | 9.0 | ✓ | ✓ | Also requires nvidia-fabricmanager |

**H200 production service (no restrictions):**

```bash
ExecStart=/usr/bin/python3 -m vllm.entrypoints.openai.api_server \
    --model /data/models/your-model \
    --host 0.0.0.0 \
    --port 8000
```

---

## Troubleshooting

**1. python3-dev required**
vLLM compiles CUDA utilities at startup even with TRITON_ATTN backend. Missing `Python.h` causes `EngineCore failed to start`. Solution: install python3-dev offline package.

**2. pip download interrupted**
Do NOT save to `/tmp` — cleared on reboot. Use `/data/` or other persistent path.

**3. Live SSH access during installation**
```bash
ssh installer@<target-ip>   # password: ubuntu
sudo -i
tail -f /var/log/installer/subiquity-server-debug.log
```

---

## Version Upgrade

```bash
# Upgrade vLLM only - user-data unchanged
# 1. Prepare new vllm-pkgs.tar.gz and vllm.service in new version dir
mkdir -p .../vllm/vllm-0.18.0/
# copy vllm-pkgs.tar.gz and vllm.service

# 2. Update symlink
ln -sfn vllm-0.18.0 .../vllm/current
```

---

## Author

**安栋梁 (An Dongliang)**
Infrastructure & AI Ops Engineer | RHCE · HCIE · KYCP

GitHub: https://github.com/itter81
