# QEMU 环境搭建

## 安装基础环境
```bash
# 更新软件源
sudo apt-get update

# 安装基础编译工具
sudo apt-get install build-essential git wget curl

# 安装 qemu dts
sudo apt-get install -y device-tree-compiler qemu-system-arm

```

## 安装buildroot

```bash
sudo apt-get install -y bash binutils build-essential bzip2 cpio diffutils findutils g++ gcc gawk git gzip make patch perl pkg-config sed tar unzip wget ncurses-dev python3 python-is-python3 rsync

```

## 下载 buildroot
```bash
mkdir -p buildroot

export BR_VER=2025.02.15
wget https://buildroot.org/downloads/buildroot-${BR_VER}.tar.xz

# 解压
tar -xvf buildroot-2025.02.15.tar.xz

```

## 查找支持的板子

Buildroot 通过 `configs/` 目录下的 defconfig（默认配置）文件提供板子支持，每个文件对应一块开发板，命名格式为 `<板子名>_defconfig`。

### 选择 make qemu_aarch64_virt_defconfig

```bash
make qemu_aarch64_virt_defconfig
```

### 下载 Linux kernel
```bash
# 确认这个目录确实是内核源码根目录
head -n 6 /home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112/Makefile

# 保存现有配置
cp -a .config ".config.before-local-linux.$(date +%Y%m%d-%H%M%S)"

stty rows 40 cols 120
make menuconfig

```
### 设置 Linux Kernel
进入 `Kernel`，保持以下设置：

```text
[*] Linux Kernel
    Kernel version -> Custom version
    Kernel version -> 6.12.112
    Kernel configuration -> Using a custom (def)config file
    Configuration file path -> board/qemu/aarch64-virt/linux.config
    Kernel binary format -> Image
```

### 指定位置
检查时 `local.mk` 尚不存在。创建它：

```bash
cat > local.mk <<'EOF'
LINUX_OVERRIDE_SRCDIR = /home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112
EOF

make olddefconfig

grep -E '^BR2_LINUX_KERNEL_(CUSTOM_VERSION_VALUE|VERSION|CUSTOM_CONFIG_FILE)=' .config
cat local.mk

# 检查
ub22@ub22-virtual-machine:~/mxxmstar/buildroot/buildroot-2025.02.15$ make olddefconfig
#
# configuration written to /home/ub22/mxxmstar/buildroot/buildroot-2025.02.15/.config
#
ub22@ub22-virtual-machine:~/mxxmstar/buildroot/buildroot-2025.02.15$ grep -E '^BR2_LINUX_KERNEL_(CUSTOM_VERSION_VALUE|VERSION|CUSTOM_CONFIG_FILE)=' .config
BR2_LINUX_KERNEL_CUSTOM_VERSION_VALUE="6.12.112"
BR2_LINUX_KERNEL_VERSION="6.12.112"
BR2_LINUX_KERNEL_CUSTOM_CONFIG_FILE="board/qemu/aarch64-virt/linux.config"
ub22@ub22-virtual-machine:~/mxxmstar/buildroot/buildroot-2025.02.15$ cat local.mk
LINUX_OVERRIDE_SRCDIR = /home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112
```

如果以后 `local.mk` 已有其他内容，只编辑其中的 `LINUX_OVERRIDE_SRCDIR`，不要再次用上面的命令覆盖整个文件。


## 编译整套系统

在 Buildroot 顶层执行，不要用 `sudo make`：

```bash
cd /home/ub22/mxxmstar/buildroot/buildroot-2025.02.15

# 可选：把包内编译并行度固定为 2，符合当前虚拟机 CPU 数量
make menuconfig
# Build options -> Number of jobs to run simultaneously -> 2

set -o pipefail
make 2>&1 | tee build.log
```

检查产物：

```bash
ls -lh output/images/
ls -l output/host/bin/*gcc
```

预期主要产物：

| 路径 | 用途 |
| --- | --- |
| `output/images/Image` | ARM64 内核镜像 |
| `output/images/rootfs.ext4` | QEMU 使用的根文件系统镜像 |
| `output/images/start-qemu.sh` | 板级 post-image 脚本生成的启动脚本 |
| `output/host/bin/` | 交叉编译工具和宿主工具 |
| `output/build/linux-custom/.config` | 本次实际内核配置 |

完整编译前，单独执行 `make linux` 也会先构建内核依赖的工具链，但不会生成完整可启动的 rootfs。你的第一次目标是进入 Shell，因此推荐直接完整 `make`。
