# 红米 K90 Pro Max（已 root）编译与上机验证指南

依据本仓库 `main`（`524d27b`）的 `scripts/product.py`、`configs/product-k27.json`、`k27/`、
`scripts/droidvm-app.py`、`scripts/run-macos-vm.py` 与 `docs/ci-env-plan.md` 整理。
目标不是刷 Android 固件，而是：**编译 macOS 27.2 的内核 / 内核缓存（KC）/ 板级 kext / Metal 插件 / 启动盘，
把它们放到 DroidVM 的 macOS 虚拟机里启动并逐项验证。**

> 先说明范围：本仓库**不产出手机刷机包**（boot/vbmeta/super 都不动），也不编译 DroidVM App 本身。
> 所谓“固件”在这里指两样东西：
> 1. 启动盘 `boot.img` 里的启动链：`UniXNU.efi`（`EFI/BOOT/BOOTAA64.EFI`）、`DarwinBoot.efi`、`kernel.macho`（KC）、
>    信任缓存、ADT、Data 种子；
> 2. 虚拟机固件 `edk2-macos.fd`（VM 的 UEFI，放在手机 `DroidVM/MacOS/` 目录，**不由本仓库自动构建**）。

## 先看结论（TL;DR）

1. **硬门槛**：DroidVM App 必须能在 K90 Pro Max 上启动任意 VM，且内核暴露 `gh_hugepage_reserve`、App 带
   `crosvm-mmio`。这一条不过，编译产物没有地方运行。
2. **最省事的编译**：有 Ubuntu 机器就走路线 B（全本地）；没有就走路线 A（Actions 出产物）。
   两条路线都需要在最后连手机，用 K90 自己的 `generated.dtb` 重建一次 `boot.img`。
3. **三样仓库外的东西**：`edk2-macos.fd`、`ios/tools/droidvm-app.py`、`data-pcie.img` 转换器。
   没有它们可以完成“编译 + 内核/系统启动验证”，但无法完成“产品化首次安装 + GPU”。
4. **只想先验证内核能起**：直接走第 8 节的 `run-macos-vm.py`，不需要 App 的 VM 注册，也不需要 `edk2-macos.fd`。

---

## 0. 先做兼容性判定（K90 Pro Max 是否具备条件）

仓库里**没有任何机型白名单或 K90/SM8850 适配代码**。能否跑起来，先看下面六项，缺一项就不要继续编译：

| # | 检查项 | 命令（手机上执行，需 root shell） | 通过标准 |
|---|---|---|---|
| 1 | root / su | `adb shell su -c id` | 输出 `uid=0(root)`；脚本内部就是 `adb shell su -c ...` |
| 2 | DroidVM App | `adb shell su -c 'ls -l /data/data/cn.classfun.droidvm/usr/bin/crosvm-mmio'` | 文件存在且可执行 |
| 3 | crosvm 运行库 | `adb shell su -c 'ls /data/data/cn.classfun.droidvm/usr/lib'` | 有 `libcrosvm` 等运行库 |
| 4 | 固件共享件 | `adb shell su -c 'ls -l /data/data/cn.classfun.droidvm/usr/share/droidvm/edk2-gunyah.fd'` | 文件存在 |
| 5 | Gunyah 大页池 | `adb shell su -c 'cat /sys/module/gh_hugepage_reserve/parameters/pool_size_max'` | 能读到整数；模块不存在则 VM 起不来 |
| 6 | 能起 VM | 在 DroidVM App 里新建/启动任意一个现成 VM（Linux 即可） | 能正常启动到 shell/桌面 |

补充检查：

```sh
adb shell su -c 'getprop ro.product.model; getprop ro.board.platform; uname -a'
adb shell su -c 'ls -l /dev/gunyah* 2>/dev/null; dmesg | grep -i gunyah | tail'
adb shell su -c 'df -h /data/media/0'
adb shell su -c 'stat -c "%u %U:%G" /data/data/cn.classfun.droidvm'
```

- 第 6 项是**最终判定**。Gunyah、`gh_hugepage_reserve`、`crosvm-mmio` 三者是这套方案的前提，HyperOS 3 的
  K90 Pro Max 上是否开放，本仓库无法替你确认。
- 至少要留 **30 GB** 可写空间：`data-pcie.img` 逻辑 48 GiB（稀疏，实占约 20 GB），启动盘约 384 MiB，
  还要留 `boot.img.prev`。
- 记下最后一条 `stat` 输出的 App uid（例如 `u0_a258`）。`scripts/run-macos-vm.py` 里的 `OWNER` 是写死的
  `u0_a258:media_rw`，K90 上装出来大概率不是这个值，见第 4.4 节。

> 如果 App 在 K90 上根本起不了 VM，后面的编译没有意义，先找 DroidVM 作者确认机型/内核支持。

---

## 1. 两条编译路线

| | 路线 A：GitHub Actions | 路线 B：本地 Linux |
|---|---|---|
| 适合 | 手头没有 Ubuntu/Docker 环境 | 有 x86_64 Linux，想完全掌控 |
| 依赖 | fork 的 Actions + 两个 secret | Ubuntu 24.04/26.04、Docker、sudo、约 45 GB 磁盘 |
| 时间 | 首次约 23 分钟（inputs 未命中时更久） | 环境约 10 分钟；inputs 约 5 分钟；disk 约 1 分钟；full 约 8 分钟 |
| 产物 | 加密分块的 `image-27.2-<run>` artifact | 直接落在 `out/`、`artifacts/` |
| 注意 | 启动盘的 ADT 用仓库里的参考 DTB，**不是 K90 的 FDT** | 可以连手机，用本机 FDT 重建启动盘 |

两条路线最后都要走第 5 节的手机端部署；**最终启动盘建议在本机连手机重建**（原因见 5.2）。

---

## 2. 路线 A：用 GitHub Actions 编译

### 2.1 准备 secret

在 fork `bushan/macOS-DroidVM` 的 **Settings → Secrets and variables → Actions** 新建：

| Secret | 作用 | 说明 |
|---|---|---|
| `ARTIFACT_KEY` | 所有含苹果文件的产物都用它 AES-256 加密 | 自己生成一串长随机口令并保存好；丢了产物就解不开 |
| `MACOS_USER` | 新装盘里的本地管理员，格式 `用户名:密码` | 可选；不设则用 `configs/product-k27.json` 的 `droid:droid` |

并确认 **Actions** 已启用。

### 2.2 触发 workflow

1. 打开 **Actions → image → Run workflow**。
2. 选项：
   - `refresh_inputs`：首次必须让它是 `false`（第一次本来就会构建 IPSW 输入）；
   - `release`：可选，把加密产物同时发到 Release。
3. 等待这些 job：`inputs`、`install`、`kernel`、`bridge`、`turnip`、`plugin`、`image`（依赖关系见
   [image.yml](file:///workspace/.github/workflows/image.yml) 顶部注释）。

首次运行会下载约 10 GB 的 IPSW 成员并生成 `inputs-27.2-<KEY>` Release；以后命中就很快。

### 2.3 下载与解密

在本次 run 页面下载两个 artifact：

- `image-27.2-<run>`：zip，里面是加密分块的镜像；
- `image-27.2-<run>-plain`：`manifest.json`、`disk.json`、`sha256.txt` 等明文清单。

先解 zip，再解密：

```sh
unzip image-27.2-<run>.zip -d out/ci-parts
mkdir -p out/ci-image
ARTIFACT_KEY='你的ARTIFACT_KEY' \
  scripts/ci-blob.sh unpack out/ci-image out/ci-parts/image.tar.zst.enc.[0-9][0-9][0-9]
ls -lh out/ci-image
```

应看到：`boot.img`、`kernel.macho.trustcache`、`manifest.json`、`data-store.img`、
`scratch-data.img`、`disk.json`。

### 2.4 放回工作区路径（供 `product.py status/verify/deploy` 使用）

```sh
mkdir -p artifacts/27.2/disk out/product/k27
cp out/ci-image/{data-store.img,scratch-data.img,disk.json} artifacts/27.2/disk/
cp out/ci-image/{boot.img,manifest.json,kernel.macho.trustcache} out/product/k27/
```

### 2.5 路线 A 的两个关键限制

1. **启动盘不是为 K90 生成的**：`image` job 的 `MACOS_FDT` 是
   [configs/k27-generated.dtb](file:///workspace/configs/k27-generated.dtb)，来自开发者的手机。ADT 里的
   内存布局、中断号、virtio 窗口地址都可能不同。首次上机前，最好在本机用 K90 的 FDT 重建
   `boot.img`（见 5.2），而不是直接部署 CI 的 `boot.img`。
2. **Data 盘格式**：`configs/product-k27.json` 当前是 `MACOS_DATA_PCI=1`，App 的 VM 会打开
   `MacOS/data-pcie.img`（原生 GPT/APFS + virtio-blk PCI）。CI 只产出 `data-store.img`
   （`DroidVMDataStore` 覆盖层格式），**仓库里没有把 `data-store.img` 转成 `data-pcie.img` 的脚本**
   （转换器只在未入库的 `research/` 工作目录里）。新装一台 K90 时，要么自备/自行转换
   `data-pcie.img`，要么改走旧的 MMIO 数据盘路径（需要额外 `NativeVirtioBlock`/`NativeRamBlock` kext，
   见 [try.sh](file:///workspace/k27/try.sh#L22-L26)，不建议作为第一步）。

---

## 3. 路线 B：本地 Linux 编译

以下命令都在仓库根目录执行。

### 3.1 宿主要求

| 项 | 要求 |
|---|---|
| 系统 | Ubuntu 24.04 / 26.04，x86_64 |
| CPU | 8 核可用（开发机 20 核；CI 是 4 核 16 GB） |
| 内存 | ≥ 16 GB |
| 磁盘 | ≥ 45 GB 可用（inputs 16 GB + 新装盘约 15 GB + toolchains 5.3 GB + out） |
| 权限 | `sudo`（`losetup`、挂载 APFS、`unshare -r`） |
| 容器 | Docker（Darling 镜像，提供 `ld-classic` 等） |
| 网络 | 能访问 GitHub、Apple 的 IPSW/AEA 服务、crates.io、PyPI |

注意：`/tmp` 配额通常不够，统一把临时目录放到仓库里：

```sh
export TMPDIR=$PWD/out/tmp
mkdir -p "$TMPDIR"
```

### 3.2 安装宿主依赖

先让脚本按锁文件打印精确清单：

```sh
python3 scripts/setup-env.py host
```

然后（以它打印的内容为准）：

```sh
# LLVM 21（apt.llvm.org）
wget -qO /tmp/llvm.sh https://apt.llvm.org/llvm.sh
sudo bash /tmp/llvm.sh 21

sudo apt-get update
sudo apt-get install -y \
  build-essential autoconf automake libtool pkg-config git python3 \
  python3-mako python3-yaml python3-packaging meson ninja-build cmake bison flex m4 \
  glslang-tools mtools docker.io unifdef tcsh pax libbsd-dev uuid-dev libssl-dev \
  zlib1g-dev cpio xz-utils zstd ccache llvm-21 clang-21 lld-21 7zip gcc-aarch64-linux-gnu \
  "linux-headers-$(uname -r)"

# 不带版本号的 clang/llvm-ar/llvm-nm
export PATH=/usr/lib/llvm-21/bin:$PATH

# Python 依赖（Ubuntu 24.04 自带 meson 太老）
python3 -m pip install 'meson==1.10.1' 'mako==1.3.10' 'pyyaml==6.0.3' 'packaging==26.0'

# bridge 用固定 nightly
rustup toolchain install nightly-2026-09-05 --profile minimal --component rust-src

# Ubuntu 24.04 的 AppArmor 默认禁止非特权 user namespace（mkapfs 会失败）
sudo sysctl -qw kernel.apparmor_restrict_unprivileged_userns=0
```

确认 `docker info` 能正常执行。

### 3.3 生成工具链与源码树

```sh
python3 scripts/setup-env.py toolchain   # SDK、cctools、MIG、CLTools、Darling、ld-classic、bin26
python3 scripts/setup-env.py x12377      # 内核构建环境、KDK 头、fakeroot、ArmCpu/ArmDTPlatform
python3 scripts/setup-env.py sources     # 所有上游源码 + mkapfs/apfsck/apfs.ko/ipsw + UniXNU.efi/DarwinBoot.efi
```

验收点：

- `toolchains/sdks/MacOSX26.1.sdk/`、`toolchains/linux/bin26/` 存在；
- `out/x12377/xnu-m/`（xnu `ac9718f` + `patches/xnu-12377` 回放后的 `k27` 分支）；
- `out/k27/xnu/` 工作树存在；
- `sources/UniXNU/`、`sources/UniXNU-droidvm/`、`sources/UniXNU-turnip/`、`sources/edk2/` 等按
  [configs/env.lock.json](file:///workspace/configs/env.lock.json) 固定在指定提交；
- `artifacts/boot-inputs/UniXNU.efi` 与 `artifacts/boot-inputs/DarwinBoot.efi` 已生成，并有
  `provenance.json`；
- `toolchains/ipsw/ipsw` 可执行。

### 3.4 取 IPSW 输入

```sh
python3 scripts/product.py inputs
```

- 按 [configs/product-k27.json](file:///workspace/configs/product-k27.json) 的 `inputs`，只 HTTP Range 下载用到的成员，
  按苹果清单校验，AEA 解密，生成信任缓存并解出 kernelcache。
- 需要联网取 AEA 密钥；下载约 10 GB，开发机实测约 5 分钟（网速慢会到 8 分钟以上）。
- 产物：`artifacts/27.2/{os.raw, dec/, app-cryptex/, kernelcache.release.vma2,
  boot/kernel.macho.trustcache, DeviceTree...}` 与 `out/kc/kc27.bin`。
- 中断后可以重跑，已完成的步骤会跳过。

### 3.5 生成新装系统盘

```sh
python3 scripts/product.py disk artifacts/27.2/disk
```

- 内部会调用 `make-macos-disk.py`，需要 `sudo`（`losetup` + `linux-apfs-rw` 的 `apfs.ko`）。
- 生成单 APFS 容器：System（原苹果 System 卷 + SystemOS cryptex）、Data、Preboot、VM、Update、Hardware、xART。
- 产物：`artifacts/27.2/disk/{data-store.img, scratch-data.img, disk.json}`。
  - `data-store.img`：给手机的数据存储（稀疏，逻辑 48 GiB）；
  - `scratch-data.img`：启动盘里的 Data 种子；
  - `disk.json`：卷 UUID，`system.uuid` 就是 `MACOS_ROOT_UUID`。
- 失败常见原因：`apfs.ko` 与运行内核不匹配（重装 `linux-headers` 后重跑 `setup-env.py sources`）、
  AppArmor 拒绝 `unshare -r`（3.2 的 sysctl）。

### 3.6 编译内核 / kext / KC / Metal / 启动盘

最省事：

```sh
python3 scripts/product.py full
```

它顺序执行：

| 步骤 | 命令 | 产物 |
|---|---|---|
| 内核 | `bash k27/build-kernel27.sh` | `out/k27/kernel-k27.macho` |
| 板级 kext | 配置里的 `kexts` 列表 | `out/x12377/native-kexts/` |
| 内核缓存 | `bash k27/try.sh 0`（`K27_NO_BOOT=1`） | `out/k27/kc-k27.bin` |
| Turnip | `scripts/build-turnip-macos.py` | `out/metal/mesa-tu/` |
| bridge | `scripts/build-metal-bridge-macos.py`（与其他步骤并行，约 7 分钟） | `out/metal/bridge/` |
| Metal 插件 | `scripts/build-metal-plugin.sh --no-deps` | `out/metal/plugin/NativeAdrenoMetal.bundle` |
| Shader Cache App | `scripts/build-shader-cache-app.py` | `out/metal/shader-cache/Shader Cache.app` |
| 启动盘 | `scripts/droidvm-app.py` 的 `build_boot` | `out/product/k27/{boot.img, manifest.json, kernel.macho.trustcache}` |

注意：`full` **不包含** `stage`（3.7 要单独跑）。

只想更新其中一项时：

```sh
python3 scripts/product.py kernel
python3 scripts/product.py kexts
python3 scripts/product.py kc
python3 scripts/product.py gpu --deps     # 改了 GPU 用户态才需要 --deps
python3 scripts/product.py boot
python3 scripts/product.py build          # kc + gpu + boot
```

**编译启动盘时的 FDT**

`product.py boot` 默认从手机拉最新的 `generated.dtb`，再用它生成 ADT，所以需要手机已连接、App 里已有
这个 VM 的注册（见 5.2）。如果只想先产出一份镜像，可以借用参考 DTB：

```sh
MACOS_FDT=$PWD/configs/k27-generated.dtb python3 scripts/product.py boot
```

这份镜像可以用于“构建是否成功”的检查，但**不要把它当作 K90 的最终启动盘**。

### 3.7 把插件和设置写进 Data 种子

```sh
python3 scripts/product.py stage
```

- 把 `NativeAdrenoMetal.bundle`、`Shader Cache.app`、`disk.stage` 里的设置、用户 `droid:droid` 写进新装盘的
  Data 卷；需要 `sudo`。
- 之后插件包和启动盘是**配套的**：启动盘信任缓存里记的是这个插件包内 Mach-O 的 CDHash。换插件后必须
  重新 `boot` 生成启动盘，并在 guest 里 `install-plugin`（见 5.3）。

### 3.8 产物清单

| 文件 | 用途 |
|---|---|
| `out/k27/kernel-k27.macho` | 源码内核 |
| `out/k27/kc-k27.bin` | 上机用的内核缓存 |
| `out/metal/plugin/NativeAdrenoMetal.bundle` | guest `/Library/GPUBundles/` |
| `out/metal/shader-cache/Shader Cache.app` | guest `/Applications/` |
| `out/product/k27/boot.img` | 启动盘（FAT，含 EFI/UniXNU/Payloads） |
| `out/product/k27/manifest.json` | 提交号、KC/启动盘哈希、信任的 CDHash、内核参数 |
| `artifacts/27.2/disk/data-store.img` | 手机数据存储 |
| `artifacts/27.2/disk/scratch-data.img` | 启动盘 Data 种子 |
| `artifacts/27.2/disk/disk.json` | 卷 UUID |

---

## 4. 手机端准备（两条路线通用）

### 4.1 连接与 root

```sh
adb devices
adb shell su -c 'id'          # 必须 uid=0
adb shell su -c 'getenforce'  # 记录 SELinux 状态，便于排查
```

如果 `adb shell` 本身就是 root（`adb root`），可以不写 `su -c`；脚本默认按 `su -c` 写。

### 4.2 App 与目录

```sh
APP=/data/data/cn.classfun.droidvm
adb shell su -c "ls -l $APP/usr/bin/crosvm-mmio $APP/usr/share/droidvm/edk2-gunyah.fd"
adb shell su -c "ls -l /data/media/0/DroidVM/MacOS 2>/dev/null || echo no-folder"
adb shell su -c "mkdir -p /data/media/0/DroidVM/MacOS"
adb shell su -c "df -h /data/media/0"
```

### 4.3 固件 `edk2-macos.fd`

产品路径（`MACOS_VIRTIO_GPU=1`）要求手机目录里有一份 **关掉 virtio-gpu 的自编固件**：

```sh
adb shell su -c 'ls -l /data/media/0/DroidVM/MacOS/edk2-macos.fd'
```

- 有：继续。
- 没有：`scripts/droidvm-app.py` 会把 VM 的 `boot.uefi.firmware` 指到
  `/data/media/0/DroidVM/MacOS/edk2-macos.fd`，crosvm 因打不开固件而启动失败。
  两种处理：
  1. 从上一台机器/备份的 `DroidVM/MacOS/` 目录复制过来；
  2. 不用 GPU 做第一次启动：直接运行 `droidvm-app.py` 时设 `MACOS_VIRTIO_GPU=0 MACOS_GPU=0`，
     这样 VM 用 App 自带的 `edk2-gunyah.fd` 和 simplefb，先验证内核/系统能起来，再补固件开 GPU。
     （`product.py` 会用 `configs/product-k27.json` 的 env 覆盖你的环境变量；要临时改走路
     `droidvm-app.py`，或改配置。）
- 本仓库只有 `overlay/edk2/GunyahBootPkg/` 和 `sources/edk2/`，**没有** `edk2-macos.fd` 的构建脚本；
  它来自仓库外的 `~/edk2-gunyah-topo`（`-D VIRTIO_GPU_ENABLE=FALSE`）。

### 4.4 App uid 与 `OWNER`

`scripts/run-macos-vm.py` 第 36 行把文件属主写死为 `u0_a258:media_rw`：

```sh
adb shell su -c 'stat -c "%u %U:%G" /data/data/cn.classfun.droidvm'
```

如果输出不是 `u0_a258`，把
[run-macos-vm.py](file:///workspace/scripts/run-macos-vm.py#L36-L36) 的 `OWNER` 改成实际值，否则
`chown` 会失败，App 的 daemon 也可能打不开磁盘。

### 4.5 Gunyah 大页池

```sh
POOL=/sys/module/gh_hugepage_reserve/parameters
adb shell su -c "cat $POOL/pool_size_max $POOL/refill_stat"
```

`scripts/phone_pool.py` 会按 `MACOS_MEM_MB`（默认 5120）调整 `pool_want`；要求当前没有其它 VM 占用。
启动前先在 App 里停掉其它 VM。

---

## 5. 首次部署（产品路径）

### 5.1 前置：App 的配置工具

`scripts/droidvm-app.py` 通过 `../ios/tools/droidvm-app.py`（仓库外）和 App 的 daemon IPC 写 `vms.json`。
没有这份工具时，`install`/`prepare`/`register`/`start` 等会报
`.../ios/tools/droidvm-app.py (the DroidVM app plumbing) is needed for adb`
（`scripts/droidvm-app.py` 的 `NoPhone` 分支只在设置 `MACOS_FDT` 时还能构建启动盘）。

- 有该工具：按下面命令走。
- 没有：至少先在 App 里手动创建 macOS VM，并确认它能按下面的参数启动（`--block boot.img`、
  `--virtio-mmio-block data-store.img` 或 `--block data-pcie.img`、`--dump-device-tree-blob`、内核参数）；
  否则只能用第 8 节的调试路径。

### 5.2 安装数据盘并注册 VM

```sh
# 1) 把新装盘写进手机（稀疏推送，只传有数据的区间）
python3 scripts/droidvm-app.py install artifacts/27.2/disk

# 2) 推送占位启动盘 -> 注册 VM -> 启动一次让 crosvm 导出 generated.dtb -> 停 VM
#    再用这份 FDT 生成 ADT、组装启动盘并推送
python3 scripts/droidvm-app.py prepare
```

`prepare` 结束时手机目录里已经有 K90 自己的 `generated.dtb`，VM 处于停止状态。
但 `prepare` 组装启动盘用的是**基础信任缓存**（不含 Metal 插件的 CDHash），所以还要再走一次产品流程：

```sh
python3 scripts/product.py boot      # 用手机上的 generated.dtb 重建，信任缓存里加入插件
python3 scripts/product.py deploy
```

然后进 App 点启动。如果只看系统能否启动、不要 GPU，可以跳过这两步（插件也不会加载）。

如果只是更新已有 VM 的启动盘：

```sh
python3 scripts/product.py boot      # 或 build
python3 scripts/product.py deploy    # 推送、校验 sha256、保留 boot.img.prev、下次启动生效
```

`deploy` 只替换 `boot.img`，不会动数据盘。

### 5.3 插件变化时的顺序

```sh
# 1) 重新生成启动盘（信任缓存里要有新插件的 CDHash）
python3 scripts/product.py build

# 2) guest 运行中，把新插件放进 guest（需要 guest 网络和密码）
MACOS_GUEST_PASSWORD='guest密码' \
  python3 scripts/product.py install-plugin droid@<guest-ip>

# 3) 部署启动盘并重启 VM
python3 scripts/product.py deploy
```

> 规则：**启动盘和信任缓存每次从头生成**；guest 里的插件必须就是构建出的那个包。先在 guest 里换插件、
> 再部署信任它的启动盘、最后重启，顺序不能反。

---

## 6. 上机验证

### 6.1 启动与日志位置

App 里启动 VM 后：

```sh
VM_ID=3307dca0-8a10-59af-8abf-1a1501cd5b40
adb shell su -c "tail -n 200 /data/data/cn.classfun.droidvm/cache/console_${VM_ID}_serial1.log"
adb shell su -c "tail -n 100 /data/data/cn.classfun.droidvm/cache/console_${VM_ID}_stderr.log"
adb shell su -c "tail -n 100 /data/data/cn.classfun.droidvm/cache/daemon.log"
```

预期里程碑（按顺序）：

1. `UniXNU.efi` 读配置、出启动菜单；
2. `DARWINBOOT: ARM64 va=0x... pa=0x...` 交接行；
3. 内核 kext 加载、`launchd` 启动，**没有 panic**；
4. 屏幕出现登录窗口（`--setup-done` 已写 `.AppleSetupDone`）；
5. 登录后 Dock/Finder、WindowServer 正常。

### 6.2 构建与部署一致性

```sh
python3 scripts/product.py status
python3 scripts/product.py verify        # 拉手机上的 boot.img，逐载荷比对
```

`status` 应显示手机上的启动盘就是 manifest 里那份（或“部署后已被启动写过”）；
`verify` 最后一行应为 `boots the same`。

### 6.3 内核日志 / panic

```sh
# 从宿主读 guest 的内核 msgbuf（需要 root，串口日志里有 DarwinBoot 交接行）
python3 scripts/rig-dmesg.py /path/to/serial1.log | tail -80

# 卡死/panic 现场（需要内核参数 panic-spin=1，VM 才会停在 panic）
bash k27/panic.sh 120                # 打印 panic 文本，存 out/k27/panic.txt
bash k27/dbg-state.sh                # debugger 各 CPU 状态
python3 k27/cpu-stacks.py            # 各 CPU 栈
```

### 6.4 功能验收清单

| 项目 | 怎么验 | 通过标准 |
|---|---|---|
| 系统启动 | 串口日志 + 屏幕 | 到登录窗口，无 panic |
| 磁盘 | guest 里 `diskutil list` / 读写文件 | System/Data/Preboot/VM/Update/Hardware/xART 可见；写入后重启仍在 |
| 网络 | guest 里 `ping <网关>`、`ssh droid@<ip>` | 10/10 通，无丢包 |
| 显示 | 拖动窗口、打开 Launchpad | 不黑屏、不明显掉帧 |
| Metal/GPU | Safari 打开网页、`Shader Cache.app`、窗口动画 | WebContent 不崩、窗口拖动流畅 |
| 音频 | 播放提示音/音乐、录音 | 扬声器、麦克风都工作 |
| 视频 | Safari/QuickTime 播放 H.264/HEVC | 有画面、音画同步 |
| USB | 插 U 盘 | 能挂载（需 `MACOS_USB_PCI=1`） |
| 睡眠/唤醒 | Apple 菜单 → 睡眠，再唤醒 | 能回到桌面（K90 上是否支持需实测） |

### 6.5 失败对照表

| 现象 | 优先检查 | 处理 |
|---|---|---|
| 打不开 `edk2-macos.fd` / 固件启动失败 | 4.3 | 放固件，或先关 GPU 启动 |
| 启动几秒后 panic，栈里有 launchd | 缺 `mac-amfi-launch-permissive=1`、`malloc-ranges-off=1`、`arm64_maxoffset` 等必需参数 | 确认 boot.img 来自 `droidvm-app.py`/`product.py`，不要手搓内核参数 |
| `MACOS_ROOT_UUID must name the System volume` | `artifacts/27.2/disk/disk.json` 不在 | 重新 `product.py disk` 或从 image artifact 放入 |
| `App 报 failed to start VM` | 大页池被占、VM 未停、App 不支持 | 停其它 VM；查 `refill_stat`；确认第 0 节第 6 项 |
| `Permission denied` 写 `/data/media/0/DroidVM/MacOS` | `OWNER`/uid | 改 4.4 的 `OWNER` 或用 root 手动 `chown` |
| `ios/tools/droidvm-app.py ... needed for adb` | 缺跨仓库工具 | 补 `../ios/tools/droidvm-app.py`，或改用手动部署/第 8 节调试路径 |
| `apfs.ko` 编译/加载失败 | 内核头版本 | 重装 `linux-headers-$(uname -r)` 后重跑 `setup-env.py sources` |
| `mkapfs` 报 user namespace | AppArmor | `sudo sysctl -qw kernel.apparmor_restrict_unprivileged_userns=0` |
| `data-pcie.img` 不存在 | 配置走 PCI 数据盘 | 自备转换后的 `data-pcie.img`，或改走 MMIO 数据盘路径 |
| `install` 报 `data-store.img exists: not replaced` | 已有 VM 的数据盘 | 这是保护；要另建一台 VM 就设 `MACOS_INSTANCE=<名字>` |
| 插件更新后 WindowServer 崩溃 | 启动盘信任缓存与 guest 插件不一致 | 重新 `build` → `install-plugin` → `deploy` → 重启 |

---

## 7. 回滚与安全

- `product.py deploy` 会把上一份启动盘保留为 `boot.img.prev`，直接移回即可回滚。
- 数据盘的改动不可逆时，保留一份 `data-store.img` / `data-pcie.img` 快照（App 里再建一个 VM 副本更安全）。
- 本项目**不刷手机分区**，不改 boot/vbmeta；风险主要在 App 的 VM 配置、Gunyah 宿主内核模块和磁盘空间。
- 调试完把 `panic-spin=1` 去掉，否则 panic 后 VM 一直停着。

---

## 8. 备用验证路径：不经过 App 的 VM 注册

如果只是想把内核/启动盘跑起来看日志，可以走 `scripts/run-macos-vm.py`（`k27/try.sh`、
`scripts/xnu12377-smoke.sh`、`scripts/boot-cycle.sh` 是同一目标下的其他入口）：

- 直接用 `adb` 在手机上运行 App 自带的 `crosvm-mmio`，不写 `vms.json`；
- 需要手机上已有 `data-store.img`、`data-pcie.img` 或已部署的数据盘；
- 需要 `sources/UniXNU-droidvm/board.toml` 和 `scripts/make-droidvm-adt.py`（`setup-env.py sources` 会取）；
- 自动导出 `generated.dtb` → 生成 `kernel.macho.adt` → 组装启动盘 → 启动并保留证据到
  `research/runs/<时间>-<label>/`。

直接调用（不依赖 `research/runs/p3-boot-20261022/` 那个硬编码目录）：

```sh
python3 scripts/run-macos-vm.py --serial "$(adb devices | awk 'NR==2{print $1}')" \
  --seconds 120 --keep --mem-mb 2560 --label k90-first \
  --kernel-arg exec-log=1 --kernel-arg crash-log=1 --kernel-arg syscall-err-log=1 \
  --kernel-arg mac-sandbox-permissive=1 --kernel-arg mac-amfi-launch-permissive=1
```

产物在 `research/runs/*-k90-first/`，里面有 `serial.log`、`crosvm.log`，`--keep` 保留全部证据。
`scripts/boot-cycle.sh` 只是它的包装，并会额外做 `rig-dmesg` 解码。

---

## 9. 日常更新流程（记住这一条）

```sh
python3 scripts/product.py build          # KC + 插件 + 启动盘
python3 scripts/product.py install-plugin droid@<guest-ip>   # 插件变了才需要
python3 scripts/product.py deploy         # 推启动盘
# 重启 VM
python3 scripts/product.py status
```

改动只涉及内核或 kext 时：`kernel`/`kexts` → `kc` → `boot` → `deploy`。
改动只涉及 Metal/GPU 时：`gpu --deps` → `boot` → `install-plugin` → `deploy`。

---

## 10. 仍然存在的缺口（K90 首次适配要自己补）

1. **机型支持**：仓库没有 K90 Pro Max 的验证记录；Gunyah、`gh_hugepage_reserve`、`crosvm-mmio` 必须由
   App/内核提供。
2. **App 配置工具**：`ios/tools/droidvm-app.py` 不在本仓库，部署脚本依赖它。
3. **`edk2-macos.fd`**：只有外部构建产物，仓库内没有构建脚本。
4. **`data-pcie.img` 转换器**：只在未入库的 `research/` 里；CI 产物只有 `data-store.img`。
   由于 `configs/product-k27.json` 默认 `MACOS_DATA_PCI=1`，**一台完全没有 macOS 数据的全新 K90
   无法只靠本仓库完成首次安装**：还需要一份已转换的 `data-pcie.img`（或改回 MMIO 数据盘并补
   `NativeVirtioBlock`/`NativeRamBlock` kext）。
5. **CI 启动盘用参考 DTB**：必须用 K90 自己 dump 的 `generated.dtb` 重建 ADT。

这五项里第 1 项是硬门槛，其余是流程/工具缺口。建议顺序：先过第 0 节，再用路线 B 或 A 产出
KC/插件/系统盘，最后在本机连手机用真实 FDT 重建 `boot.img` 并部署。
