# 01-ollama | PXE Automated Deploy - Ollama Inference Stack

实现从裸机到 Ollama 推理服务就绪的全自动交付，零人工干预。
底层交付方式：iPXE + Ubuntu Autoinstall 离线装机。

Automated GPU Inference Node Deploy - Ollama

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
    | 7. late-commands: NVIDIA driver + CUDA + Ollama
    v
Reboot -> GPU driver + CUDA + Ollama all ready
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
        │   ├── meta-data              # Empty file (required)
        │   ├── user-data              # Active config (symlinked or copied)
        │   ├── user-data-ollama       # Ollama version
        │   └── user-data-vllm        # vLLM version
        ├── payload/
        │   ├── base/
        │   │   └── gcc-pkgs/          # 45 offline deb packages
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
        │       └── ollama/
        │           ├── current -> ollama-0.17.7
        │           ├── ollama-0.17.7/
        │           │   ├── ollama.tar.gz
        │           │   └── ollama.service
        │           └── ollama-0.18.0/
        │               ├── ollama.tar.gz
        │               └── ollama.service
        └── ubuntu-22.04.5-custom-live-server-amd64.iso
```

**Version upgrade: only change symlink, user-data unchanged:**

```bash
ln -sfn ollama-0.18.0 /data/pxe/iso/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/ollama/current
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

### ollama.tar.gz

**How to prepare:**

```bash
# Download official Ollama release
# Package with bin/ and lib/ at root level (no extra nesting)
cd /path/to/ollama-extracted
tar -zcf ollama.tar.gz -C . bin/ lib/
```

Extraction result: `bin/ollama` and `lib/ollama/` directly, no extra nesting.

### ollama.service

```ini
[Unit]
Description=Ollama Service
After=network.target

[Service]
Type=simple
ExecStart=/usr/local/bin/ollama serve
Restart=always
RestartSec=3
Environment="HOME=/root"
Environment="OLLAMA_ORIGINS=*"

[Install]
WantedBy=multi-user.target
```

---

## user-data (Cloud-init Autoinstall)

File: `/data/pxe/iso/wubantu/ubuntu-22.04.5-custom/cloud-init/user-data-ollama`

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
    - curtin in-target -- sh -c "apt-mark hold linux-image-5.15.0-119-generic linux-headers-5.15.0-119-generic linux-image-generic linux-headers-generic linux-generic"

    # --- CUDA ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/cuda/current/cuda.run -O /target/tmp/cuda.run
    - curtin in-target -- chmod +x /tmp/cuda.run
    - curtin in-target -- sh -c "/tmp/cuda.run --silent --toolkit --no-drm --tmpdir=/tmp || true"
    - curtin in-target -- update-initramfs -u
    - curtin in-target -- sh -c "echo 'export PATH=/usr/local/cuda/bin:\$PATH' > /etc/profile.d/cuda.sh"
    - curtin in-target -- sh -c "echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:\$LD_LIBRARY_PATH' >> /etc/profile.d/cuda.sh"

    # --- Ollama ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/ollama/current/ollama.tar.gz -O /target/tmp/ollama.tar.gz
    - curtin in-target -- sh -c "mkdir -p /tmp/ollama-extract && tar xzf /tmp/ollama.tar.gz -C /tmp/ollama-extract"
    - curtin in-target -- mv /tmp/ollama-extract/bin/ollama /usr/local/bin/ollama
    - curtin in-target -- mv /tmp/ollama-extract/lib/ollama /usr/local/lib/ollama
    - curtin in-target -- chmod 755 /usr/local/bin/ollama
    - curtin in-target -- chmod -R 755 /usr/local/lib/ollama/
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/ollama/current/ollama.service -O /target/etc/systemd/system/ollama.service
    - curtin in-target -- chmod 644 /etc/systemd/system/ollama.service
    - curtin in-target -- systemctl enable ollama

  shutdown: reboot
```

---

## Key Parameter: --skip-module-load

This is the core technical solution of this project.

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

## Verification

```bash
nvidia-smi && nvcc --version && systemctl is-active ollama
```

Expected:

```
Driver Version: 580.126.09   CUDA Version: 13.0
Cuda compilation tools, release 13.2, V13.2.51
active
```

---

## Troubleshooting

**1. overlay root is read-only**
All persistent operations must use `curtin in-target --` or write to `/target/*` directly.

**2. nouveau in initramfs**
Blacklisting alone is not enough. Must run `update-initramfs -u` twice: before driver install (remove nouveau) and after (inject nvidia module).

**3. installer rollback**
`--no-nouveau-check` skips detection but NOT modprobe. Only `--skip-module-load` prevents rollback.


---

## Version Upgrade

```bash
# Upgrade Ollama only - user-data unchanged
ln -sfn ollama-0.18.0 .../payload/gpu-inference/ollama/current

# Upgrade NVIDIA driver
ln -sfn 590.x.xx .../payload/gpu-inference/nvidia-driver/current

# Upgrade CUDA
ln -sfn cuda-14.x.x .../payload/gpu-inference/cuda/current
```

---

## Author

**安栋梁 (An Dongliang)**
Infrastructure & AI Ops Engineer | RHCE · HCIE · KYCP

GitHub: https://github.com/itter81
