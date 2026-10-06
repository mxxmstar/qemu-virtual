# Buildroot + QEMU ARM64 + Linux 6.12 驱动学习编译指南

检查日期：2026-10-06。本文按 SSH 实际检查结果编写，目标是先启动 ARM64 Linux 并进入 Shell，然后在独立源码目录中逐步开发 Mini-VPU、Mini-NPU 驱动。

## 1. 你的实际环境

| 项目 | 检查结果 |
| --- | --- |
| SSH | `ssh ub22@192.168.117.131`，端口 22 |
| 宿主机 | Ubuntu，x86_64，宿主内核 6.8.0-138-generic |
| 资源 | 2 个逻辑 CPU，约 15 GiB 内存，约 380 GiB 可用磁盘 |
| Buildroot | `/home/ub22/mxxmstar/buildroot/buildroot-2025.02.15` |
| 已选择的配置 | `qemu_aarch64_virt_defconfig` |
| 当前 `.config` 内核版本 | `6.12.27` |
| 实际下载并解压的内核 | `6.12.112` |
| 独立源码目录 | `/home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112` |
| 内核压缩包位置 | Buildroot 下的 `linux/dl/linux-6.12.112.tar.xz` |
| 内核配置模板 | `board/qemu/aarch64-virt/linux.config` |
| 工具链 | Buildroot 自建 AArch64 工具链，GCC 13.4.0，glibc |
| 根文件系统 | ext4，当前大小 60 MiB |
| QEMU | 已有系统 `qemu-system-aarch64`；Buildroot 也开启了 host QEMU |
| 基础依赖 | 实测 `make dependencies` 成功 |

**关键问题：直接执行当前配置的 `make`，会选择 6.12.27，而不会自动使用旁边解压的 6.12.112。** 本文通过 `local.mk` 将 Buildroot 接到你的独立内核源码。

本文提供操作步骤；检查时尚未运行完整编译、修改 Buildroot 配置或验证 QEMU 启动。文中的命令需要你按顺序执行。

## 2. 理解目录和两个配置层次

当前目录可以直接沿用，不需要重新下载或搬动内核：

```text
/home/ub22/mxxmstar/
├── buildroot/
│   └── buildroot-2025.02.15/
│       ├── .config          # 整个系统的 Buildroot 配置
│       ├── local.mk         # 待创建：指定独立内核源码
│       └── output/          # 自动生成的编译结果
├── linux/
│   └── linux-6.12/
│       └── linux-6.12.112/  # 以后在这里修改驱动
└── docs/
```

- `make menuconfig`：配置 Buildroot，决定架构、工具链、软件包、内核来源和根文件系统。
- `make linux-menuconfig`：配置 Linux 内核，决定哪些驱动和内核功能参与编译。

`qemu_aarch64_virt_defconfig` 是 Buildroot 的整机配置名称。它选用的内核模板是 `board/qemu/aarch64-virt/linux.config`，不是让你在 Linux 内核目录执行同名 defconfig。

6.12.27 和 6.12.112 都属于 6.12 LTS 系列。这里选择已经下载的 6.12.112。Buildroot 与内核版本可以分别选择，但仍需要注意工具链头文件和软件包兼容性；版本解耦不代表任意组合都能工作。

## 3. 接入独立的 Linux 6.12.112 源码

登录虚拟机，进入 Buildroot：

```bash
ssh ub22@192.168.117.131
cd /home/ub22/mxxmstar/buildroot/buildroot-2025.02.15

# 确认这个目录确实是内核源码根目录
head -n 6 /home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112/Makefile

# 保存现有配置
cp -a .config ".config.before-local-linux.$(date +%Y%m%d-%H%M%S)"
```

应看到 `VERSION = 6`、`PATCHLEVEL = 12`、`SUBLEVEL = 112`。

执行：

```bash
make menuconfig
```

进入 `Kernel`，保持以下设置：

```text
[*] Linux Kernel
    Kernel version -> Custom version
    Kernel version -> 6.12.112
    Kernel configuration -> Using a custom (def)config file
    Configuration file path -> board/qemu/aarch64-virt/linux.config
    Kernel binary format -> Image
```

保存退出。这里的版本号用于让配置记录与实际源码一致；下一步的源码覆盖机制才决定实际从哪里取得内核。

检查时 `local.mk` 尚不存在。创建它：

```bash
cat > local.mk <<'EOF'
LINUX_OVERRIDE_SRCDIR = /home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112
EOF

make olddefconfig

grep -E '^BR2_LINUX_KERNEL_(CUSTOM_VERSION_VALUE|VERSION|CUSTOM_CONFIG_FILE)=' .config
cat local.mk
```

如果以后 `local.mk` 已有其他内容，只编辑其中的 `LINUX_OVERRIDE_SRCDIR`，不要再次用上面的命令覆盖整个文件。

工作方式：

```text
你维护的 linux-6.12.112 源码
          ↓ rsync
output/build/linux-custom/
          ↓ 交叉编译
output/images/Image
```

Buildroot 会把源码同步到 `output/build/linux-custom/` 再编译。它不会直接在你维护的源码目录里生成所有构建文件。不要在 `output/build/linux-custom/` 中长期修改驱动，清理构建目录会丢失这些修改。

**你当前开启了 `BR2_KERNEL_HEADERS_AS_KERNEL=y`。** 已核对本版本 `package/linux-headers/linux-headers.mk`：工具链的 Linux headers 会自动继承 `LINUX_OVERRIDE_SRCDIR`。因此，只设置上面这一行，不要再设置 `LINUX_HEADERS_OVERRIDE_SRCDIR`，否则该配置会报错。

源码覆盖方式会跳过该包的下载、解压和自动打补丁。如果以后需要 Buildroot 提供的内核补丁，应在独立源码中自行应用并通过 Git 记录。当前 QEMU 内核补丁目录中的补丁面向 MIPS/PowerPC，并非 ARM64 的启动补丁。

## 4. 第一次编译整套系统

在 Buildroot 顶层执行，不要用 `sudo make`：

```bash
cd /home/ub22/mxxmstar/buildroot/buildroot-2025.02.15

# 可选：把包内编译并行度固定为 2，符合当前虚拟机 CPU 数量
make menuconfig
# Build options -> Number of jobs to run simultaneously -> 2

set -o pipefail
make 2>&1 | tee build.log
```

Buildroot 会按依赖关系编译宿主工具、交叉工具链、内核、BusyBox、根文件系统以及已启用的 host QEMU。普通 `make` 会使用 Buildroot 的并行度配置；目前 `BR2_JLEVEL=0` 表示自动选择，不需要在第一步照搬内核编译的 `make -j$(nproc)`。

首次编译还会联网下载其他软件包。已有 Linux 源码不代表所有依赖都已下载。两个 CPU 的首次构建可能耗时较长，具体取决于网络和磁盘性能。

失败后先看 `build.log` 最后一段，解决原因后再次执行 `make`，会利用已有构建结果继续。不要习惯性执行 `make clean`，它会删除构建成果。

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

## 5. 启动 QEMU 并进入 Shell

优先用 Buildroot 生成的脚本：

```bash
cd /home/ub22/mxxmstar/buildroot/buildroot-2025.02.15
./output/images/start-qemu.sh serial-only
```

检查过脚本模板，`serial-only` 是受支持的参数；脚本会优先使用 `output/host/bin` 的 QEMU。

也可以使用虚拟机已有的系统 QEMU：

```bash
./output/images/start-qemu.sh --use-system-qemu serial-only
```

或手工执行与板级 README 对应的启动命令：

```bash
qemu-system-aarch64 \
  -M virt -cpu cortex-a53 -m 512M -smp 1 -nographic \
  -kernel output/images/Image \
  -append "rootwait root=/dev/vda console=ttyAMA0" \
  -netdev user,id=eth0 \
  -device virtio-net-device,netdev=eth0 \
  -drive file=output/images/rootfs.ext4,if=none,format=raw,id=hd0 \
  -device virtio-blk-device,drive=hd0
```

登录名为 `root`，当前配置的 root 密码为空。进入 Shell 后检查：

```bash
uname -r
uname -m
cat /proc/cmdline
mount
ip addr
```

预期内核版本为 `6.12.112`（可能带附加后缀），架构为 `aarch64`，根设备为 `/dev/vda`。这才算完成“Linux → QEMU ARM64 → Shell”的第一个里程碑。

关机可在客体内执行 `poweroff`。需要退出 QEMU 时，先按 `Ctrl+A`，松开，再按 `X`。

当前配置没有要求单独构建 DTB。QEMU `virt` 会生成描述模拟设备的设备树并交给内核，因此首次启动不需要手工加 `-dtb`。

## 6. 以后修改内核和驱动怎样编译

### 6.1 只修改驱动源码

在独立内核目录编辑，例如：

```text
/home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112/drivers/misc/mini_vpu/
```

新增目录时还需要修改对应的 `Kconfig` 和 `Makefile`，并在内核配置中选中驱动。仅创建 `.c` 文件不会自动参与编译。

回到 Buildroot：

```bash
cd /home/ub22/mxxmstar/buildroot/buildroot-2025.02.15
make linux-rebuild
make
```

`linux-rebuild` 会重新同步独立源码并增量编译内核；随后 `make` 重新生成根文件系统等产物，尤其适用于驱动编译为模块时。之后重新启动 QEMU。正在运行的 QEMU 不会自动切换到新内核。

### 6.2 修改内核配置

```bash
make linux-menuconfig
make linux-update-defconfig
make linux-rebuild
make
```

当前配置使用自定义配置文件，`linux-update-defconfig` 会把精简配置写回 `board/qemu/aarch64-virt/linux.config`。这样下次干净构建也能保留你的选择；该文件的改动也应该进入版本管理。

如果你直接修改了这个配置模板，并希望重新加载它，用：

```bash
make linux-reconfigure
make
```

源码目录的修改通常用 `linux-rebuild`；配置输入发生变化时用 `linux-reconfigure`。正常学习迭代不需要每次全量清理。

### 6.3 保存 Buildroot 配置

不要直接覆盖发行版自带的 `configs/qemu_aarch64_virt_defconfig`。保存你自己的配置：

```bash
mkdir -p /home/ub22/mxxmstar/board
make savedefconfig BR2_DEFCONFIG=/home/ub22/mxxmstar/board/qemu_aarch64_virt_dev_defconfig
```

`local.mk` 的内容不会包含在 defconfig 中，需要单独保存。将来恢复自定义 defconfig 时，文件名仍以 `qemu_` 开头、以 `_defconfig` 结尾，当前 post-image 脚本也需要在板级 README 中找到对应标签才能生成启动脚本；换名字后应同步维护这个标签，或者使用本文的手工 QEMU 命令。

不要在每次构建前重新执行 `make qemu_aarch64_virt_defconfig`，这会重新载入发行版默认配置，覆盖你已修改的 Buildroot 设置。

## 7. 用 Git 管理独立内核

检查时 `/home/ub22/mxxmstar` 已有 Git 仓库，但内核源码目录没有独立仓库，并且被上层 `.gitignore` 忽略。对内核源码单独初始化仓库，可以保留原始版本并逐步记录驱动开发。

```bash
cd /home/ub22/mxxmstar/linux/linux-6.12/linux-6.12.112
git init
git status --short
git add .
git commit -m "Import Linux 6.12.112 baseline"
```

如果 Git 提示未配置身份，先设置你自己的名字和邮箱，再提交。Linux 源码文件多，首次 `git add` 和提交需要时间。这是嵌套的独立仓库，上层项目默认不会保存其内部历史，备份时要同时备份内核仓库。

后续按功能提交，例如：

```text
Import Linux 6.12.112 baseline
Add mini-vpu platform driver skeleton
Add mini-vpu MMIO registers
Add mini-vpu interrupt handling
Add mini-vpu DMA buffer handling
Add mini-vpu V4L2 M2M interface
Add mini-npu platform driver
```

每一步先验证、再提交，避免一次同时引入设备模型、IRQ、DMA 和用户态接口，难以定位问题。

## 8. 第一阶段不用增加大量内核功能

沿用 `board/qemu/aarch64-virt/linux.config` 即可。检查时该模板已经启用了模块加载、PL011 串口、VirtIO 网卡、VirtIO 块设备、ext4，以及 ARM SMMU v3 等功能，不需要为了“完整”先把所有驱动勾上。

建议学习顺序：

1. 编译并进入 QEMU Shell，确认运行的是 6.12.112。
2. 编写最简单的内核模块，练习 `insmod`、`rmmod`、`dmesg`。
3. 实现 `platform_driver`，理解 `probe`、设备树匹配和资源获取。
4. 在 QEMU 中实现 Mini-VPU 设备模型及设备树节点，再逐步增加 MMIO 和 IRQ。
5. 增加 DMA，再学习 DMA-BUF、IOMMU 和 V4L2 M2M。
6. 用同样方法发展 Mini-NPU。

**Linux 驱动与 QEMU 设备模型是两份独立代码。** 仅添加 Linux 的 `platform_driver` 或设备树节点，不会让 QEMU 自动出现真实的寄存器、IRQ 和 DMA 行为；这些行为需要 QEMU 侧实现。当前 VM 中的系统 QEMU 和 Buildroot 自动构建的 QEMU 可以完成首次启动，后续修改设备模型时再安排独立 QEMU 源码和构建目录。

## 9. 常见问题

| 现象 | 检查方法和处理 |
| --- | --- |
| 仍然下载 6.12.27 | 确认在正确 Buildroot 目录执行，`local.mk` 路径和 `LINUX_OVERRIDE_SRCDIR` 是否正确 |
| 修改 `.c` 后没有变化 | 使用 `make linux-rebuild` 同步源码；确认 Kconfig/Makefile 已接入并且配置已选中 |
| QEMU 显示旧版本 | 确认启动使用的 `Image` 路径，关闭旧 QEMU 并重新启动 |
| 内核无法挂载 rootfs | 检查 `root=/dev/vda`、VirtIO 块设备参数，内核 VirtIO 块驱动和 ext4 应内建为 `y` |
| 没有串口输出 | 检查 `console=ttyAMA0`、PL011 console 配置和 `-nographic` |
| rootfs 打包空间不足 | 在 Buildroot 的 Filesystem images 中增大 ext2/3/4 文件系统大小，再执行 `make` |
| 修改配置后干净构建丢失 | 用 `linux-update-defconfig` 保存内核配置，用 `savedefconfig` 保存 Buildroot 配置，并保存 `local.mk` |
| 只改下载版本后出现 hash 错误 | 当前启用了强制 hash 检查，板级 hash 仅列出 6.12.27；本文用源码覆盖跳过内核下载流程。若改用 tarball 下载，应验证来源并维护相应 hash |

切换架构、工具链或头文件系列通常需要新的完整构建。当前尚未完整构建，所以应该先接好独立内核源码，再进行第一次 `make`。日后不要把开发驱动的日常增量构建与工具链切换混为一谈。

## 10. 本文依据

以下文件均已在你的虚拟机上读取核对：

- Buildroot `.config` 和 `configs/qemu_aarch64_virt_defconfig`
- `board/qemu/aarch64-virt/linux.config`、`readme.txt`
- `board/qemu/post-image.sh`、`start-qemu.sh.in`
- `docs/manual/using-buildroot-development.adoc` 的源码覆盖说明
- `package/linux-headers/linux-headers.mk` 的 headers 源码继承逻辑
- 独立内核源码 `Makefile` 中的 6.12.112 版本信息

本文给出的首要路线是：**接入独立源码 → 完整 `make` → 启动 QEMU → 确认 Shell 和内核版本 → 再开始驱动开发。**
