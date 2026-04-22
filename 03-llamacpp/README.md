# 04-llamacpp | PXE Automated Deploy - llama.cpp Inference Stack

实现从裸机到 llama.cpp 推理服务就绪的全自动交付，零人工干预。
底层交付方式：iPXE + Ubuntu Autoinstall 离线装机。

Automated GPU Inference Node Deploy - llama.cpp (CUDA)

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
    | 7. late-commands: NVIDIA driver + CUDA + llama.cpp (CUDA build)
    v
Reboot -> GPU driver + CUDA + llama-server (CUDA backend) all ready
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
        │   ├── user-data                  # Active config
        │   ├── user-data-ollama           # Ollama version
        │   ├── user-data-vllm             # vLLM version
        │   └── user-data-llamacpp         # llama.cpp version  <-- this doc
        ├── payload/
        │   ├── base/
        │   │   └── gcc-pkgs.tar.gz        # 45 offline deb packages
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
        │           ├── current -> llama-cuda-src2504      # active version
        │           ├── llama-cuda-src2504/                # CUDA build (current)
        │           │   ├── llama/                         # 87 static binaries
        │           │   │   ├── llama-server               # main inference server ★
        │           │   │   ├── llama-cli
        │           │   │   ├── llama-bench
        │           │   │   ├── llama-quantize
        │           │   │   └── ...                        # no .so files (static link)
        │           │   ├── llama-server.service
        │           │   └── llama.tar.gz
        │           └── llama-vulkan-b8495/                # Vulkan build (backup)
        │               ├── llama-server.service
        │               └── llama.tar.gz
        └── ubuntu-22.04.5-custom-live-server-amd64.iso
```

**Version upgrade: only change symlink, user-data unchanged:**

```bash
ln -sfn llama-cuda-src2505 /data/pxe/iso/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/llama/current
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

### llama.tar.gz (CUDA build from source)

**Build info:**
- llama.cpp version: b8881 (0dedb9ef7)
- Built with: GNU 11.4.0 for Linux x86_64
- CUDA architecture: 750 (GTX 1650, compute capability 7.5)
- Build date: 2025-04
- Link type: **static** — no .so dependencies, single binary deployment

**How to build on a machine with CUDA installed:**

```bash
# Clone source
px git clone https://github.com/ggerganov/llama.cpp.git /opt/llama-src

# Build with CUDA, static linking, GTX 1650 architecture
cd /opt/llama-src
cmake -B build \
    -DGGML_CUDA=ON \
    -DGGML_VULKAN=OFF \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CUDA_ARCHITECTURES=75 \
    -DBUILD_SHARED_LIBS=OFF \
    -DLLAMA_CURL=OFF
cmake --build build -j$(nproc)

# Verify GPU recognition
/opt/llama-src/build/bin/llama-server --list-devices
# Expected: ggml_cuda_init: found 1 CUDA devices

# Package for PXE distribution
mkdir -p /tmp/llama-pkg/llama
cp /opt/llama-src/build/bin/* /tmp/llama-pkg/llama/
cd /tmp/llama-pkg
tar -zcf llama.tar.gz llama/
# Transfer to PXE server
scp llama.tar.gz root@192.168.70.230:.../llama/llama-cuda-src2504/
```

**Directory structure inside llama.tar.gz:**

```
llama/
├── llama-server          # Main HTTP inference server  ★
├── llama-cli             # Interactive CLI
├── llama-bench           # Benchmark tool
├── llama-quantize        # Model quantization tool
├── llama-batched-bench
├── llama-imatrix
├── llama-gguf-split
├── llama-mtmd-cli
├── export-graph-ops
├── test-*                # Test binaries (87 files total)
└── ...
# No .so files — all statically linked into each binary
```

> **Note:** CMAKE_CUDA_ARCHITECTURES=75 targets GTX 1650 (Turing).
> For other GPUs: RTX 30xx = 86, RTX 40xx = 89, A100 = 80, H100 = 90

### llama-server.service

```ini
[Unit]
Description=llama.cpp inference server
After=network.target

[Service]
Type=simple
User=root
SupplementaryGroups=render video
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
- `-ngl 99` — offload all layers to GPU (CUDA auto-selects device)
- `-m` — path to the GGUF model file
- `SupplementaryGroups=render video` — ensures GPU device access

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
  # Matches disks 700G ~ 1024G (system disk only, data disk untouched)
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
    # Lock kernel version to prevent driver breakage on kernel upgrade
    - curtin in-target -- sh -c "apt-mark hold linux-image-5.15.0-119-generic linux-headers-5.15.0-119-generic linux-image-generic linux-headers-generic linux-generic"

    # --- CUDA ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/cuda/current/cuda.run -O /target/tmp/cuda.run
    - curtin in-target -- chmod +x /tmp/cuda.run
    - curtin in-target -- sh -c "/tmp/cuda.run --silent --toolkit --no-drm --tmpdir=/tmp || true"
    - curtin in-target -- update-initramfs -u
    - curtin in-target -- sh -c "echo 'export PATH=/usr/local/cuda/bin:\$PATH' > /etc/profile.d/cuda.sh"
    - curtin in-target -- sh -c "echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:\$LD_LIBRARY_PATH' >> /etc/profile.d/cuda.sh"

    # --- llama.cpp (CUDA build, statically linked) ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/llama/current/llama.tar.gz -O /target/tmp/llama.tar.gz
    - curtin in-target -- mkdir -p /opt/
    - curtin in-target -- tar xzf /tmp/llama.tar.gz -C /opt/

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

## Key Lesson: Kernel Lock

**Problem:** After apt installing Vulkan packages, kernel upgraded automatically.
NVIDIA driver compiled for old kernel → driver breaks after reboot.

**Solution:** Lock kernel immediately after driver compilation:

```bash
apt-mark hold linux-image-5.15.0-119-generic \
    linux-headers-5.15.0-119-generic \
    linux-image-generic \
    linux-headers-generic \
    linux-generic
```

This is included in user-data late-commands after nvidia.run.

---

## CUDA vs Vulkan Backend

llama.cpp supports multiple GPU backends. This deployment uses **CUDA** as primary.

```
CUDA backend (current):
  - NVIDIA proprietary API, best performance
  - Only works on NVIDIA GPUs
  - llama.cpp official first recommendation
  - Statically compiled into binary (no dlopen at runtime)
  - GTX 1650 compute capability 7.5 (Turing architecture)

Vulkan backend (backup, llama-vulkan-b8495):
  - Open standard: NVIDIA / AMD / Intel all supported
  - Performance ~10% less than CUDA
  - Requires runtime Vulkan ICD: libnvidia-gl-580 + nvidia_icd.json
  - Known issue: nvidia.run overwrites apt-installed libGLX_nvidia.so,
    breaking Vulkan. Fix: reinstall libnvidia-gl-580 after nvidia.run
```

**Why CUDA was chosen over Vulkan:**

- Vulkan ICD conflicts between nvidia.run and apt packages caused GPU not found
- CUDA backend has no such conflict — compiled directly into binary
- Static linking eliminates all runtime library dependency issues

---

## Verification

```bash
# 1. Verify GPU driver
nvidia-smi

# 2. Verify CUDA toolkit
nvcc --version

# 3. Verify llama-server binary and GPU recognition
/opt/llama/llama-server --list-devices
# Expected output:
# ggml_cuda_init: found 1 CUDA devices (Total VRAM: 3712 MiB):
#   Device 0: NVIDIA GeForce GTX 1650, compute capability 7.5

# 4. Verify model file
ls -lh /data/models/gguf/

# 5. Verify service status
systemctl status llama-server

# 6. Verify port listening
ss -tlnp | grep 8080

# 7. Check startup logs (look for CUDA keywords)
journalctl -u llama-server -n 30 --no-pager | grep -E "cuda|CUDA|offload|GPU"

# 8. Test inference (final validation)
curl -s http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen","messages":[{"role":"user","content":"你好"}],"max_tokens":50}' \
  | python3 -m json.tool --no-ensure-ascii

# 9. Watch GPU utilization during inference
watch -n 1 nvidia-smi
```

**Validation criteria (all must pass):**

```
✅ --list-devices: found 1 CUDA devices
✅ logs: ggml_cuda_init: found 1 CUDA devices
✅ logs: CUDA : ARCHS = 750
✅ nvidia-smi: VRAM > 0MiB during inference
✅ nvidia-smi: GPU-Util > 0% during inference
```

**Expected startup log (validated on GTX 1650):**

```
ggml_cuda_init: found 1 CUDA devices (Total VRAM: 3712 MiB):
  Device 0: NVIDIA GeForce GTX 1650, compute capability 7.5, VMM: yes, VRAM: 3712 MiB
build_info: b8881-0dedb9ef7
system_info: n_threads = 6 | CUDA : ARCHS = 750 | USE_GRAPHS = 1 | ...
```

**Expected nvidia-smi during inference:**

```
| 0  NVIDIA GeForce GTX 1650  ...  | 2660MiB / 4096MiB |  96%  Default |
| 0   N/A  N/A   xxx   C   /opt/llama/llama-server   xxxxMiB |
```

---

## Troubleshooting

**1. overlay root is read-only**
All persistent operations must use `curtin in-target --` or write to `/target/*` directly.

**2. nouveau in initramfs**
Blacklisting alone is not enough. Must run `update-initramfs -u` twice: before driver install (remove nouveau) and after (inject nvidia module).

**3. installer rollback**
`--no-nouveau-check` skips detection but NOT modprobe. Only `--skip-module-load` prevents rollback.

**4. Kernel upgrade breaks driver**
apt installing any package may pull in a new kernel as dependency.
Fix: `apt-mark hold` on kernel packages immediately after driver install.
Symptom: `nvidia-smi` fails after reboot, `lsmod | grep nvidia` empty.

**5. CUDA binary not finding GPU**
Check CUDA environment variables are set:
```bash
echo $PATH | grep cuda
echo $LD_LIBRARY_PATH | grep cuda
# If missing: source /etc/profile.d/cuda.sh
```

**6. ldd shows no cuda libs**
With static linking (`-DBUILD_SHARED_LIBS=OFF`), CUDA is compiled into the binary.
`ldd /opt/llama/llama-server` will not show cuda libs — this is expected.
Verify with `--list-devices` instead.

**7. Vulkan backend (backup) GPU not found**
Root cause: `nvidia.run` overwrites `libGLX_nvidia.so` installed by `libnvidia-gl-580` apt package.
Fix sequence (must be in this order):
```bash
sh nvidia.run ...           # 1. compile kernel module
apt reinstall libnvidia-gl-580  # 2. restore correct Vulkan ICD library
ldconfig
```

---

## Version Upgrade

```bash
# Upgrade llama.cpp — recompile from source, repackage, update symlink
ln -sfn llama-cuda-src2505 .../payload/gpu-inference/llama/current

# Upgrade NVIDIA driver
ln -sfn 590.x.xx .../payload/gpu-inference/nvidia-driver/current
# Note: also update --kernel-name in user-data if kernel changes

# Upgrade CUDA
ln -sfn cuda-14.x.x .../payload/gpu-inference/cuda/current

# Switch to a different model
# Change -m path in llama-server.service, no user-data change needed
```

---

## Author

**安栋梁 (An Dongliang)**
Infrastructure & AI Ops Engineer | RHCE · HCIE · KYCP

GitHub: https://github.com/itter81
