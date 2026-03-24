# 04-llamacpp | PXE Automated Deploy - llama.cpp Inference Stack

实现从裸机到 llama.cpp 推理服务就绪的全自动交付，零人工干预。
底层交付方式：iPXE + Ubuntu Autoinstall 离线装机。

Automated GPU Inference Node Deploy - llama.cpp (Vulkan + CUDA)

---

## Architecture Overview

```
Client (bare-metal server)
    |
    | 1. Power on, PXE boot
    v
DHCP Server (192.168.70.230)
    |
    | 2. First request: assign IP, return ipxe.efi
    | 3. Second request (iPXE): return http://server/boot.ipxe
    v
HTTP Server (Nginx on 192.168.70.230:80)
    |
    | 4. Load boot menu (boot.ipxe)
    | 5. Load Ubuntu ISO + cloud-init user-data
    v
Ubuntu Autoinstall (Cloud-init)
    |
    | 6. Partition, install base system
    | 7. late-commands: NVIDIA driver + CUDA + Vulkan deps + llama.cpp
    v
Reboot -> GPU driver + CUDA + llama-server all ready
```

---

## Server Environment

### 1. DHCP Configuration

File: `/etc/dhcp/dhcpd.conf`

```bash
option domain-name-servers 192.168.70.230;
default-lease-time 60;
max-lease-time 120;
authoritative;

option space ipxe;
option ipxe-encap-opts code 175 = encapsulate ipxe;

subnet 192.168.70.0 netmask 255.255.255.0 {
    range 192.168.70.193 192.168.70.200;
    option routers 192.168.70.230;
    next-server 192.168.70.230;

    if exists user-class and option user-class = "iPXE" {
        filename "http://192.168.70.230/boot.ipxe";
    } else {
        filename "ipxe.efi";
    }
}
```

### 2. TFTP

Only used for Stage 1 (delivering ipxe.efi bootloader).

```
/var/lib/tftpboot/
└── ipxe.efi
```

### 3. Nginx Configuration

File: `/etc/nginx/conf.d/pxe-80.conf`

```nginx
server {
  listen 80;
  server_name www.pxe.com;
  root /data/pxe/iso/;
  autoindex on;
  autoindex_localtime on;
  location / {
    index index.html;
  }
}
```

### 4. iPXE Boot Menu

File: `/data/pxe/iso/boot.ipxe`

```bash
#!ipxe
:start
menu PXE Boot Menu
item ubuntu-22.04.5-custom  Ubuntu-22.04.5-Custom-Autoinstall
item shell iPXE Shell (Debug)

choose --default ubuntu-22.04.5-custom --timeout 10000 target && goto ${target}

:ubuntu-22.04.5-custom
set os-name ubuntu-22.04.5-custom
goto boot-ubuntu

:boot-ubuntu
set base-url http://192.168.70.230/wubantu
kernel ${base-url}/${os-name}/casper/vmlinuz \
    initrd=initrd \
    boot=casper \
    url=${base-url}/${os-name}/${os-name}-live-server-amd64.iso \
    autoinstall \
    ds=nocloud-net;s=${base-url}/${os-name}/cloud-init/ \
    ip=dhcp \
    splash
initrd ${base-url}/${os-name}/casper/initrd
boot

:shell
shell
```

---

## Server Directory Structure

```
/data/pxe/iso/
├── boot.ipxe
└── wubantu/
    └── ubuntu-22.04.5-custom/
        ├── casper/
        │   ├── initrd
        │   └── vmlinuz
        ├── cloud-init/
        │   ├── meta-data                  # Empty file (required)
        │   ├── user-data                  # Active config (symlinked or copied)
        │   ├── user-data-ollama           # Ollama version
        │   ├── user-data-vllm             # vLLM version
        │   └── user-data-llamacpp         # llama.cpp version  <-- this doc
        ├── payload/
        │   ├── base/
        │   │   └── gcc-pkgs/              # 45 offline deb packages
        │   │       └── gcc-pkgs.tar.gz
        │   └── gpu-inference/
        │       ├── cuda/
        │       │   ├── current -> cuda-13.2.0-595.45.04
        │       │   └── cuda-13.2.0-595.45.04/
        │       │       └── cuda.run
        │       ├── nvidia-driver/
        │       │   ├── current -> 580.126.09
        │       │   └── 580.126.09/
        │       │       └── nvidia.run
        │       ├── models/
        │       │   └── qwen2.5-0.5b-instruct-q4_k_m.gguf
        │       └── llama/
        │           ├── current -> llama-vulkan-b8495
        │           ├── llama-vulkan-b8495/
        │           │   ├── llama/         # binaries + shared libs
        │           │   │   ├── llama-server
        │           │   │   ├── libggml-vulkan.so
        │           │   │   ├── libggml.so.0.9.8
        │           │   │   ├── libllama.so.0.0.8495
        │           │   │   └── ...
        │           │   ├── llama-server.service
        │           │   └── llama.tar.gz
        │           └── vulkan-pkgs.tar.gz # Vulkan offline deb packages
        └── ubuntu-22.04.5-custom-live-server-amd64.iso
```

**Version upgrade: only change symlink, user-data unchanged:**

```bash
ln -sfn llama-vulkan-b8496 /data/pxe/iso/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/llama/current
```

---

## Payload Preparation

### gcc-pkgs (Base compilation dependencies)

45 offline deb packages required for NVIDIA driver compilation.

**How to prepare on a machine with internet access (same OS: Ubuntu 22.04):**

```bash
mkdir gcc-pkgs && cd gcc-pkgs
apt-get download gcc make \
  linux-headers-5.15.0-119 \
  linux-headers-5.15.0-119-generic \
  $(apt-cache depends --recurse --no-recommends gcc make | grep "^\w" | sort -u)
cd ..
tar -zcf gcc-pkgs.tar.gz gcc-pkgs/
```

Key packages included: gcc, gcc-11, cpp-11, make, linux-headers, libc6-dev, libgcc-11-dev, and all their dependencies.

### llama.tar.gz

llama.cpp official pre-built Vulkan binary release. Download from:

```
https://github.com/ggml-org/llama.cpp/releases/download/b8496/llama-b8496-bin-ubuntu-vulkan-x64.tar.gz
```

**How to prepare:**

```bash
# Download and rename the extracted directory
wget https://github.com/ggml-org/llama.cpp/releases/download/b8496/llama-b8496-bin-ubuntu-vulkan-x64.tar.gz
tar xzf llama-b8496-bin-ubuntu-vulkan-x64.tar.gz
# The extracted folder contains the llama/ directory with all binaries and .so files
# Re-pack for PXE distribution (extracts to /opt/llama/)
tar -zcf llama.tar.gz -C llama-b8496-bin-ubuntu-vulkan-x64 .
```

Extraction result: `/opt/llama/llama-server` and all shared libraries directly under `/opt/llama/`.

**Directory structure inside llama.tar.gz:**

```
llama/
├── llama-server          # Main HTTP inference server  ★
├── llama-cli             # Interactive CLI
├── llama-bench           # Benchmark tool
├── llama-quantize        # Model quantization tool
├── libggml-vulkan.so     # Vulkan GPU backend  ★
├── libggml.so.0.9.8      # GGML core library
├── libggml-base.so.0.9.8
├── libllama.so.0.0.8495  # llama inference library
├── libmtmd.so.0.0.8495   # Multimodal support
├── libggml-cpu-*.so      # CPU architecture backends (auto-selected)
│   ├── libggml-cpu-haswell.so
│   ├── libggml-cpu-alderlake.so
│   └── ...
└── LICENSE
```

### vulkan-pkgs.tar.gz

Vulkan runtime dependencies — **GPU vendor-specific**, not universal.
The packages below are for **NVIDIA GTX 1650 on Ubuntu 22.04**.

**How to prepare on a machine with internet access:**

```bash
mkdir -p /tmp/vulkan-pkgs && cd /tmp/vulkan-pkgs
apt-get download \
    libvulkan1 \
    vulkan-tools \
    mesa-vulkan-drivers \
    libnvidia-gl-580 \
    libnvidia-common-580 \
    libnvidia-compute-580 \
    nvidia-firmware-580-580.126.09 \
    nvidia-kernel-common-580 \
    libdrm-amdgpu1 \
    libllvm15 \
    libwayland-client0 \
    libwayland-server0 \
    libx11-xcb1 \
    libxcb-dri3-0 \
    libxcb-present0 \
    libxcb-randr0 \
    libxcb-shm0 \
    libxcb-sync1 \
    libxcb-xfixes0 \
    libxshmfence1 \
    libgbm1 \
    libpciaccess0 \
    libnvidia-egl-wayland1
cd /tmp
tar -zcf vulkan-pkgs.tar.gz vulkan-pkgs/
```

> **Note:** Replace `580` with your actual driver version. These packages must match the NVIDIA driver version installed by `nvidia.run`.

### llama-server.service

```ini
[Unit]
Description=llama.cpp inference server
After=network.target

[Service]
Type=simple
User=ubuntu
Environment=GGML_VK_VISIBLE_DEVICES=0
ExecStart=/opt/llama/llama-server \
    -m /data/models/gguf/qwen2.5-0.5b-instruct-q4_k_m.gguf \
    --host 0.0.0.0 \
    --port 8080 \
    -ngl 99
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Key parameters:
- `GGML_VK_VISIBLE_DEVICES=0` — use GPU 0 via Vulkan
- `-ngl 99` — offload all 99 layers to GPU
- `-m` — path to the GGUF model file

### Model file

Test model used: Qwen2.5-0.5B-Instruct Q4_K_M quantization (~469MB, fits in GTX 1650's 4GB VRAM).

```
https://huggingface.co/Qwen/Qwen2.5-0.5B-Instruct-GGUF/resolve/main/qwen2.5-0.5b-instruct-q4_k_m.gguf
```

Place in PXE server:

```
/data/pxe/iso/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/models/
└── qwen2.5-0.5b-instruct-q4_k_m.gguf
```

---

## user-data (Cloud-init Autoinstall)

File: `/data/pxe/iso/wubantu/ubuntu-22.04.5-custom/cloud-init/user-data-llamacpp`

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
    # Default password: 123456
    # To generate new hash: python3 -c 'import crypt; print(crypt.crypt("yourpassword", crypt.mksalt(crypt.METHOD_SHA512)))'
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
          nameservers:
            addresses: [223.5.5.5, 8.8.8.8]

  # 1G EFI + 1G Boot + remaining XFS Root
  # Matches disks 700G ~ 1024G
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
    # Key: --skip-module-load prevents installer rollback when modprobe fails in chroot
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/nvidia-driver/current/nvidia.run -O /target/tmp/nvidia.run
    - curtin in-target -- chmod +x /tmp/nvidia.run
    - curtin in-target -- sh -c "/tmp/nvidia.run --silent --no-nouveau-check --skip-module-load --kernel-source-path=/usr/src/linux-headers-5.15.0-119-generic --kernel-name=5.15.0-119-generic 2>&1 | tee /var/log/nvidia-install.log || true"

    # --- CUDA ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/cuda/current/cuda.run -O /target/tmp/cuda.run
    - curtin in-target -- chmod +x /tmp/cuda.run
    - curtin in-target -- sh -c "/tmp/cuda.run --silent --toolkit --no-drm --tmpdir=/tmp || true"
    - curtin in-target -- update-initramfs -u
    - curtin in-target -- sh -c "echo 'export PATH=/usr/local/cuda/bin:\$PATH' > /etc/profile.d/cuda.sh"
    - curtin in-target -- sh -c "echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:\$LD_LIBRARY_PATH' >> /etc/profile.d/cuda.sh"

    # --- Vulkan runtime dependencies ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/llama/vulkan-pkgs.tar.gz -O /target/tmp/vulkan-pkgs.tar.gz
    - curtin in-target -- tar xzf /tmp/vulkan-pkgs.tar.gz -C /tmp/
    - curtin in-target -- sh -c "dpkg -i /tmp/vulkan-pkgs/*.deb || true"

    # --- llama.cpp (Vulkan GPU build) ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/llama/current/llama.tar.gz -O /target/tmp/llama.tar.gz
    - curtin in-target -- mkdir -p /opt/llama
    - curtin in-target -- tar xzf /tmp/llama.tar.gz -C /opt/llama
    - curtin in-target -- chmod +x /opt/llama/llama-server

    # --- Model ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/models/qwen2.5-0.5b-instruct-q4_k_m.gguf -O /target/tmp/qwen2.5-0.5b-instruct-q4_k_m.gguf
    - curtin in-target -- mkdir -p /data/models/gguf/
    - curtin in-target -- mv /tmp/qwen2.5-0.5b-instruct-q4_k_m.gguf /data/models/gguf/

    # --- llama-server systemd service ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/llama/current/llama-server.service -O /target/etc/systemd/system/llama-server.service
    - curtin in-target -- chmod 644 /etc/systemd/system/llama-server.service
    - curtin in-target -- systemctl enable llama-server

  shutdown: reboot
```

---

## Key Parameter: --skip-module-load

This is the core technical solution shared across all GPU inference deployments in this project.

**Why NVIDIA driver installation fails in PXE chroot:**

```
PXE boot -> Live kernel loads nouveau -> chroot to /target
-> nvidia-installer runs modprobe nvidia
-> FAILS: GPU already bound to nouveau
-> installer ROLLS BACK all installed files
-> nothing installed
```

**Why --skip-module-load works:**

```
nvidia-installer:
  1. Install driver files       -> OK
  2. Compile kernel module      -> OK
  3. modprobe nvidia            -> SKIPPED (--skip-module-load)
  4. No failure, no rollback    -> files stay on disk

After reboot:
  - nouveau is blacklisted      -> not loaded
  - nvidia module auto-loads    -> GPU ready
```

**How it was found:**

```bash
sh /tmp/nvidia.run --advanced-options 2>&1 | grep -i "load"
# --skip-module-load: Skip the test load of the NVIDIA kernel modules
# after the modules are built, and skip loading them after installation.
```

---

## Vulkan vs CUDA Backend

llama.cpp supports two GPU backends. This deployment uses **Vulkan** as primary with CUDA also installed.

```
CUDA backend:
  - NVIDIA proprietary API, best performance
  - Only works on NVIDIA GPUs
  - llama.cpp official first recommendation
  - Loaded via dlopen (not visible in ldd output)

Vulkan backend:
  - Open standard: NVIDIA / AMD / Intel all supported
  - Performance ~10% less than CUDA, negligible on small models
  - GTX 1650 fully supported
  - libggml-vulkan.so loaded at runtime
```

**How to verify which backend is active:**

```bash
# Check Vulkan ICD configuration
ls /usr/share/vulkan/icd.d/
# Should show: nvidia_icd.json

# Check GPU memory usage (Vulkan loaded = VRAM occupied)
nvidia-smi
# Process type C+G = Compute + Graphics (Vulkan)

# Check llama-server runtime libraries
ldd /opt/llama/llama-server | grep -E "vulkan|cuda"
# Note: CUDA backend uses dlopen, not visible via ldd
```

---

## Verification

```bash
# 1. Verify GPU driver
nvidia-smi

# 2. Verify CUDA toolkit
nvcc --version

# 3. Verify llama-server binary
ls -l /opt/llama/llama-server
/opt/llama/llama-server --version

# 4. Verify model file
ls -lh /data/models/gguf/

# 5. Verify service status
systemctl status llama-server

# 6. Verify port listening
ss -tlnp | grep 8080

# 7. Check startup logs (look for Vulkan/NVIDIA keywords)
journalctl -u llama-server -n 50 --no-pager

# 8. Test inference (final validation)
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen","messages":[{"role":"user","content":"你好"}],"max_tokens":50}' \
  | python3 -m json.tool

# 9. Watch GPU utilization during inference
watch nvidia-smi
```

**Validation criteria (all 4 must pass):**

```
✅ logs: offloaded 25/25 layers to GPU   # all layers on GPU
✅ logs: Vulkan0 model buffer size        # Vulkan backend active
✅ nvidia-smi: VRAM > 0MiB               # GPU memory occupied
✅ nvidia-smi: GPU-Util > 0%             # GPU computing during inference
```

**Expected output (GTX 1650 + Qwen2.5-0.5B Q4_K_M, validated):**

```json
{
  "choices": [{
    "finish_reason": "stop",
    "message": { "role": "assistant", "content": "你好！很高兴见到你..." }
  }],
  "timings": {
    "predicted_per_second": 61.54,
    "prompt_per_second": 14.22
  }
}
```

```
nvidia-smi output during inference:
| 0  NVIDIA GeForce GTX 1650  ...  | 1074MiB / 4096MiB |  24%  Default |
| 0   N/A  N/A   858   C+G   /opt/llama/llama-server   1063MiB |
```

---

## Troubleshooting

**1. overlay root is read-only**
All persistent operations must use `curtin in-target --` or write to `/target/*` directly.

**2. nouveau in initramfs**
Blacklisting alone is not enough. Must run `update-initramfs -u` twice: before driver install (remove nouveau) and after (inject nvidia module).

**3. installer rollback**
`--no-nouveau-check` skips detection but NOT modprobe. Only `--skip-module-load` prevents rollback.

**4. Vulkan ldconfig shows empty**
`ldconfig -p | grep nvidia | grep vulkan` returning empty is normal — Vulkan ICD is registered via `/usr/share/vulkan/icd.d/nvidia_icd.json`, not ldconfig. Verify with `ls /usr/share/vulkan/icd.d/` instead.

**5. ldd shows no vulkan/cuda**
llama.cpp loads GPU backends via `dlopen` at runtime, not static linking. Empty `ldd` output for vulkan/cuda is expected behavior. Check `nvidia-smi` for actual GPU usage.

**6. dpkg errors during vulkan-pkgs install**
The `|| true` at the end of the dpkg command allows partial installs. Some packages may have dependency conflicts with existing system packages — this is acceptable as long as `libvulkan1` and `libnvidia-gl-*` install successfully.

---

## Version Upgrade

```bash
# Upgrade llama.cpp only - user-data unchanged
ln -sfn llama-vulkan-b8496 .../payload/gpu-inference/llama/current

# Upgrade NVIDIA driver
ln -sfn 590.x.xx .../payload/gpu-inference/nvidia-driver/current

# Upgrade CUDA
ln -sfn cuda-14.x.x .../payload/gpu-inference/cuda/current

# Switch to a different model
# Just change the -m path in llama-server.service
# No user-data change needed
```

---

## Author

**安栋梁 (An Dongliang)**
Infrastructure & AI Ops Engineer | RHCE · HCIE · KYCP

GitHub: https://github.com/itter81
