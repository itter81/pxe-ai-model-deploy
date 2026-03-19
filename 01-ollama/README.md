01-ollama/README.md
# 01-ollama | Ollama 全自动 PXE 部署

基于 iPXE + Ubuntu Autoinstall，实现从裸机到 Ollama 推理服务就绪的零人工干预交付。

---

## 部署结果

装机完成重启后，以下服务自动就绪：

| 组件 | 版本 | 状态 |
|------|------|------|
| NVIDIA Driver | 580.126.09 | ✅ |
| CUDA | 13.2 | ✅ |
| Ollama | 0.17.7 | ✅ active (running) |

```
nvidia-smi 输出示例：
+-----------------------------------------------------------------------------------------+
| NVIDIA-SMI 580.126.09   Driver Version: 580.126.09   CUDA Version: 13.0    |
| GPU  Name         | Memory-Usage        | GPU-Util |
|   0  GTX 1650     | 0MiB / 4096MiB      |      0%  |
+-----------------------------------------------------------------------------------------+
```

---

## 文件说明

| 文件 | 说明 |
|------|------|
| `user-data` | Cloud-init 应答文件（已脱敏，YOUR_SERVER_IP 替换为实际地址） |
| `ollama.service` | Ollama systemd 服务文件 |

---

## 使用方法

**1. 替换配置中的服务端地址**

```bash
# 将 user-data 中所有 YOUR_SERVER_IP 替换为你的 PXE 服务端 IP
sed -i 's/YOUR_SERVER_IP/192.168.x.x/g' user-data
```

**2. 按服务端目录结构组织文件**

```
payload/
├── base/
│   ├── gcc-pkgs.tar.gz        # gcc/make/linux-headers 离线包
│   └── systemd/ollama.service
└── gpu-inference/
    ├── cuda/current/cuda.run
    ├── nvidia-driver/current/nvidia.run
    └── ollama/current/ollama.tar.gz
```

**3. 打包 ollama**

```bash
# 在 ollama 版本目录下打包（包含 bin/ 和 lib/ 两个目录）
tar -zcf ollama.tar.gz -C /path/to/ollama-0.17.7 .
```

**4. 触发 PXE 装机**

客户端从 PXE 启动，选择对应菜单，全程无需干预，约 15~20 分钟完成。

---

## 关键参数说明

### NVIDIA 驱动安装命令

```bash
/tmp/nvidia.run --silent \
  --no-nouveau-check \
  --skip-module-load \
  --kernel-source-path=/usr/src/linux-headers-5.15.0-119-generic \
  --kernel-name=5.15.0-119-generic
```

| 参数 | 作用 |
|------|------|
| `--no-nouveau-check` | 跳过 nouveau 检测，不因 nouveau 存在而 Abort |
| `--skip-module-load` | **关键参数**：跳过 modprobe 步骤，避免 installer 因加载失败而回滚 |
| `--kernel-source-path` | 强制指定内核头文件路径，避免 chroot 环境识别错内核 |
| `--kernel-name` | 强制指定目标内核版本 |

---

## 核心原理

### 为什么 nouveau 会导致安装失败

```
PXE 启动
  └─ 进入 Live 环境
       └─ Live 内核加载 nouveau（开源 NVIDIA 驱动）
            └─ chroot 到 /target 安装驱动
                 └─ nvidia-installer 执行 modprobe nvidia
                      └─ 失败：GPU 已被 nouveau 占用
                           └─ installer 回滚，删除所有已安装文件
```

### --skip-module-load 解决思路

```
安装文件 ✅
编译内核模块 ✅
跳过 modprobe ✅（不验证，不失败，不回滚）

重启后：
  nouveau 被 blacklist → 不加载
  nvidia 模块自动加载 → GPU 就绪
```

---

## 踩坑记录

**坑1：overlay 根只读**
late-commands 运行在 Live 环境，所有持久化操作必须用 `curtin in-target --` 或写到 `/target/*` 路径。

**坑2：nouveau 在 initramfs 里**
blacklist 写入后必须执行 `update-initramfs -u` 重建，把 nouveau 从启动镜像里移除。重建要在装驱动**之前**执行一次，装完**再执行一次**注入驱动。

**坑3：--kernel-module-only 顺序**
此参数必须在用户态已安装的前提下才能使用，顺序错误报错：`No NVIDIA driver is currently installed`。最终用 `--skip-module-load` 一步解决，不需要拆分两步。

**坑4：YAML 缩进**
late-commands 每行严格4个空格，少一个空格导致装机卡在语言选择界面。

**坑5：环境变量文件后缀**
`/etc/profile.d/` 只自动加载 `.sh` 结尾的文件，CUDA 环境变量文件必须命名为 `cuda.sh`，不能是 `cuda.env`。

**坑6：wget -r 下载目录污染**
递归下载会把 nginx 目录页 `index.html` 也下载进来。改用 `tar.gz` 打包整个目录，单文件下载解压，干净可靠。

---

## 版本升级

只需修改服务端软链接，user-data 无需改动：

```bash
# 升级 Ollama
ln -sfn ollama-0.18.0 /path/to/ollama/current

# 升级 NVIDIA 驱动
ln -sfn 590.x.xx /path/to/nvidia-driver/current

# 升级 CUDA
ln -sfn cuda-13.x.x /path/to/cuda/current
```
