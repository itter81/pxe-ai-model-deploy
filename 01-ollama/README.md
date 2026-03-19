# 01-ollama | PXE Automated Deploy - Ollama Inference Stack

基于 iPXE + Ubuntu Autoinstall，实现从裸机到 Ollama 推理服务就绪的零人工干预交付。

Automated bare-metal deployment of Ollama inference stack via iPXE + Ubuntu Autoinstall. Zero manual intervention required.

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
# Global options
option domain-name-servers 192.168.70.230;
default-lease-time 60;
max-lease-time 120;
authoritative;

# iPXE options space
option space ipxe;
option ipxe-encap-opts code 175 = encapsulate ipxe;

subnet 192.168.70.0 netmask 255.255.255.0 {
    range 192.168.70.193 192.168.70.200;
    option routers 192.168.70.230;
    next-server 192.168.70.230;

    # Two-stage boot logic:
    # Stage 1: Native PXE request -> download ipxe.efi bootloader
    # Stage 2: iPXE request (with user-class=iPXE) -> load HTTP menu
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
└── ipxe.efi          # iPXE bootloader (compiled with HTTP support)
```

### 3. Nginx Configuration

File: `/etc/nginx/conf.d/pxe-80.conf`

```nginx
server {
  listen 80;
  server_name www.pxe.com;
  root /data/pxe/iso/;
  access_log  /var/log/nginx/pxe-80-access.log main;
  error_log   /var/log/nginx/pxe-80-error.log notice;
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
# /data/pxe/iso/menus/main.ipxe

:start
#主界面菜单名单
menu PXE Boot Menu -itter
item --gap -- ------------------------- Operating Systems -------------------------

#这一行标签是引用ios和线的菜单;这一行是PXE中给人看的并不影响实际引用
item ubuntu-22.04.3           Ubuntu-22.04.3-Autoinstall
item ubuntu-22.04.5           Ubuntu-22.04.5-Autoinstall
item ubuntu-22.04.5-custom    Ubuntu-22.04.5-Custom-Autoinstall
item ubuntu-24.04.3           Ubuntu-24.04.3-Autoinstall
item ubuntu-25.04             Ubuntu-25.04-Autoinstall
item ubuntu-25.10             Ubuntu-25.10-Autoinstall

#centos菜单和上面的是一个功能
item centos-7.9               CentOS-7.9
item centos-longxi-8.10       CentOS-Longxi-8.10

item --gap -- --------------------------- Utilities ----------------------------
item shell        iPXE Shell (Debug)




######################默认调整界面,选择哪个菜单就默认PXE用那个###############################
choose --default ubuntu-22.04.5-custom --timeout 10000 target && goto ${target}
#############################################################################################






##################菜单界面起-上面的大菜单会对应到下面的子菜单################################
#wubantu的菜单#################################
:ubuntu-22.04.3
set os-name ubuntu-22.04.3
goto boot-ubuntu

:ubuntu-22.04.5
set os-name ubuntu-22.04.5
goto boot-ubuntu

:ubuntu-24.04.3
set os-name ubuntu-24.04.3
goto boot-ubuntu

:ubuntu-25.04
set os-name ubuntu-25.04
goto boot-ubuntu

:ubuntu-25.10
set os-name ubuntu-25.10
goto boot-ubuntu

:ubuntu-22.04.5-custom
set os-name ubuntu-22.04.5-custom
goto boot-ubuntu



#centos的子菜单################################
:centos-7.9
set os-name centos-7.9
set iso-name centos-7.9.iso
goto boot-centos

:centos-longxi-8.10
set os-name centos-longxi-8.10
set iso-name centos-8.10.iso
goto boot-centos


#################菜单界面止##############################################################


#####################乌班图引用应答文件逻辑#################################################
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



#####################CENTOS引用应答文件逻辑#################################################
:boot-centos
set base-url http://192.168.70.230/centos


#net.ifnames=0防止网卡名变成 ens33之类的
kernel ${base-url}/${os-name}/os/images/pxeboot/vmlinuz \
    initrd=initrd.img \
    inst.repo=${base-url}/${os-name}/os/ \
    inst.ks=${base-url}/${os-name}/ks/${os-name}.cfg \
    ip=dhcp \
    net.ifnames=0 biosdevname=0


#这里的initrd指令负责将驱动盘拉入内存
initrd ${base-url}/${os-name}/os/images/pxeboot/initrd.img
boot
###########################################################################################################################################


:shell
shell
 

#########全部截止###########################################################################################################################
```

---

## Server Directory Structure

```
/data/pxe/iso/
├── boot.ipxe                          # iPXE boot menu
└── wubantu/
    └── ubuntu-22.04.5-custom/
        ├── casper/
        │   ├── initrd                 # Ubuntu Live initrd
        │   └── vmlinuz                # Ubuntu Live kernel
        ├── cloud-init/
        │   ├── meta-data              # Empty file (required)
        │   └── user-data              # Autoinstall config (this file)
        ├── payload/
        │   ├── base/
        │   │   ├── gcc-pkgs.tar.gz    # Offline gcc/make/headers packages
        │   │   └── systemd/
        │   │       └── ollama.service
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
        │           │   └── ollama.tar.gz
        │           └── ollama-0.18.0/
        │               └── ollama.tar.gz
        └── ubuntu-22.04.5-custom-live-server-amd64.iso
```

**Version upgrade: only change symlink, user-data unchanged:**

```bash
ln -sfn ollama-0.18.0 /data/pxe/iso/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/ollama/current
```

### Packaging Ollama

```bash
# Package from inside the version directory
# Result: bin/ and lib/ at root level, no extra nesting
cd /path/to/ollama-0.17.7
tar -zcf ollama.tar.gz -C . bin/ lib/
```

### Preparing gcc offline packages

```bash
# On a machine with internet access, same OS version
apt-get download gcc make linux-headers-$(uname -r) linux-headers-generic
# Or get all dependencies at once
apt-get install --print-uris gcc make linux-headers-$(uname -r) | grep "'" | awk -F"'" '{print $2}' | wget -i -
# Pack them all
tar -zcf gcc-pkgs.tar.gz *.deb
```

---

## user-data (Cloud-init Autoinstall)

File: `/data/pxe/iso/wubantu/ubuntu-22.04.5-custom/cloud-init/user-data`

```yaml
#cloud-config
autoinstall:
  version: 1
  refresh-installer:
    update: false

  # Identity - default password: 123456
  # To change password hash:
  # python3 -c 'import crypt; print(crypt.crypt("yourpassword", crypt.mksalt(crypt.METHOD_SHA512)))'
  identity:
    hostname: ubuntu
    username: ubuntu
    password: "$6$IpXEZfUOrF4R7lPi$VmM3g0NroxoKrjB3EgzamMwn9TQTZ6vlmz3Yn0m5uhQJcqTDwk/2rcayY2scQvYzK999afy91HbUlgT0j8rwN1"
    realname: ubuntu

  locale: en_US.UTF-8
  keyboard:
    layout: us
  timezone: Asia/Shanghai

  # Network - DHCP on first en* interface
  network:
    network:
      version: 2
      ethernets:
        main_iface:
          match:
            name: en*
          dhcp4: true

  # Storage - 1G EFI + 1G Boot + remaining XFS Root
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

  # Offline apt - no network required
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
    # Mount virtual filesystems required by update-initramfs
    - mount --bind /proc /target/proc
    - mount --bind /sys /target/sys
    - mount --bind /dev /target/dev
    # Unload nouveau from Live kernel (best effort)
    - modprobe -r nouveau || true
    - modprobe -r drm_kms_helper || true
    - modprobe -r drm || true
    # Blacklist nouveau in target system
    - curtin in-target -- sh -c "echo 'blacklist nouveau' > /etc/modprobe.d/blacklist-nouveau.conf"
    - curtin in-target -- sh -c "echo 'options nouveau modeset=0' >> /etc/modprobe.d/blacklist-nouveau.conf"
    # Rebuild initramfs to remove nouveau from boot image
    - curtin in-target -- update-initramfs -u
    # Install gcc/make/headers (offline)
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/base/gcc-pkgs.tar.gz -O /target/tmp/gcc-pkgs.tar.gz
    - curtin in-target -- tar xzf /tmp/gcc-pkgs.tar.gz -C /tmp/
    - curtin in-target -- sh -c "dpkg -i /tmp/*.deb"

    # --- NVIDIA driver ---
    # Key: --skip-module-load skips modprobe step
    # Without this flag, installer rolls back all files when modprobe fails
    # (nouveau still occupies GPU in Live environment)
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/nvidia-driver/current/nvidia.run -O /target/tmp/nvidia.run
    - curtin in-target -- chmod +x /tmp/nvidia.run
    - curtin in-target -- sh -c "/tmp/nvidia.run --silent --no-nouveau-check --skip-module-load --kernel-source-path=/usr/src/linux-headers-5.15.0-119-generic --kernel-name=5.15.0-119-generic 2>&1 | tee /var/log/nvidia-install.log || true"
    # Rebuild initramfs again to inject compiled nvidia module
    - curtin in-target -- update-initramfs -u

    # --- CUDA ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/cuda/current/cuda.run -O /target/tmp/cuda.run
    - curtin in-target -- chmod +x /tmp/cuda.run
    - curtin in-target -- sh -c "/tmp/cuda.run --silent --toolkit --no-drm --tmpdir=/tmp || true"
    # CUDA environment variables (must be .sh extension to auto-load)
    - curtin in-target -- sh -c "echo 'export PATH=/usr/local/cuda/bin:\$PATH' > /etc/profile.d/cuda.sh"
    - curtin in-target -- sh -c "echo 'export LD_LIBRARY_PATH=/usr/local/cuda/lib64:\$LD_LIBRARY_PATH' >> /etc/profile.d/cuda.sh"

    # --- Ollama ---
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/gpu-inference/ollama/current/ollama.tar.gz -O /target/tmp/ollama.tar.gz
    - curtin in-target -- sh -c "mkdir -p /tmp/ollama-extract && tar xzf /tmp/ollama.tar.gz -C /tmp/ollama-extract"
    - curtin in-target -- mv /tmp/ollama-extract/bin/ollama /usr/local/bin/ollama
    - curtin in-target -- mv /tmp/ollama-extract/lib/ollama /usr/local/lib/ollama
    - curtin in-target -- chmod 755 /usr/local/bin/ollama
    - curtin in-target -- chmod -R 755 /usr/local/lib/ollama/
    - wget http://192.168.70.230/wubantu/ubuntu-22.04.5-custom/payload/base/systemd/ollama.service -O /target/etc/systemd/system/ollama.service
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
# Output includes:
#   --skip-module-load
#       Skip the test load of the NVIDIA kernel modules after the modules
#       are built, and skip loading them after installation is complete.
```

---

## Verification

After installation completes and system reboots:

```bash
# NVIDIA driver
nvidia-smi

# CUDA
nvcc --version

# Ollama service
systemctl status ollama

# One-liner check
nvidia-smi && nvcc --version && systemctl is-active ollama
```

Expected output:

```
# nvidia-smi
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09   Driver Version: 580.126.09   CUDA Version: 13.0    |
|   0  NVIDIA GeForce GTX 1650   0MiB / 4096MiB   0%   Default               |
+-----------------------------------------------------------------------------------------+

# nvcc --version
Cuda compilation tools, release 13.2, V13.2.51

# systemctl is-active ollama
active
```

---

## Troubleshooting

**1. overlay root is read-only**
All persistent operations must use `curtin in-target --` or write directly to `/target/*`.
Direct writes in late-commands go to the Live overlay and are lost on reboot.

**2. nouveau in initramfs**
Blacklisting nouveau is not enough. Must run `update-initramfs -u` to rebuild the boot image and remove nouveau. Run once before driver install, once after.

**3. installer rollback**
`--no-nouveau-check` skips detection but does NOT skip modprobe. installer still rolls back on modprobe failure. Only `--skip-module-load` prevents rollback.

**4. --kernel-module-only order**
This flag requires user-space driver to be installed first. Wrong order gives: `No NVIDIA driver is currently installed`. Solved by using `--skip-module-load` instead (installs everything in one pass).

**5. /tmp cleared on reboot**
Write logs to `/var/log/` via `curtin in-target`, not to `/tmp`.

**6. YAML indentation**
late-commands lines require exactly 4 spaces. One space short causes installer to stop at language selection screen.

**7. profile.d file extension**
`/etc/profile.d/` only auto-loads `.sh` files. CUDA env file must be named `cuda.sh`, not `cuda.env`.

**8. wget -r directory pollution**
Recursive download includes nginx directory listing as `index.html`. Use tar.gz packaging instead.

**9. ollama tar structure**
Package with `-C version-dir .` so extraction gives `bin/` and `lib/` directly, no extra nesting layer.

---

## Version Upgrade

```bash
# Upgrade Ollama - user-data unchanged
ln -sfn ollama-0.18.0 /path/to/payload/gpu-inference/ollama/current

# Upgrade NVIDIA driver
ln -sfn 590.x.xx /path/to/payload/gpu-inference/nvidia-driver/current

# Upgrade CUDA
ln -sfn cuda-14.x.x /path/to/payload/gpu-inference/cuda/current
```

---

## Author

**安栋梁 (An Dongliang)**
Infrastructure & AI Ops Engineer | RHCE · HCIE · KYCP

GitHub: https://github.com/itter81
