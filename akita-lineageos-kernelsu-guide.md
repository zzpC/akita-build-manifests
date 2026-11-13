# Pixel 8a（akita）LineageOS 24.0 + KernelSU 定制构建与刷机指南

## 1. 适用范围

本指南面向 Google Pixel 8a（代号 `akita`），说明如何在 LineageOS 官方源码基础上整合个人 GKI、设备构建配置和 KernelSU 定制，完成内核编译、ROM 打包、Recovery 刷入和 root 验证。

LineageOS 24.0 对应 Android 17，akita 使用 6.1 内核。官方设备元数据要求安装前具备 Android 17 固件，并采用 `fastboot_nexus` 安装方式。[cite:2][cite:4]

**适用分支与已验证提交：**

| 项目 | 基线或用途 | 本文固定版本 |
|---|---|---|
| LineageOS 主源码 | 官方 `lineage-24.0` | Android 17 [cite:1][cite:2] |
| AOSP GKI | `android14-6.1-2026-03_r25` | `7b58131aec77682c068b072dd055e130cfc971f7` [cite:7][cite:11] |
| `zzpC/kernel_common` | GKI + 内置 KernelSU 源码与 UAPI | `9d4c6b2a74cc1889a4f125e878748e70e9b712a3` [cite:7] |
| `zzpC/android_kernel_google_gs-6.1_devices` | akita Kleaf 配置 | `69d8014525a927ba640e33115630a380a7d272ab` [cite:8] |
| `zzpC/KernelSU` | KernelSU 参考源码、Manager 和用户空间组件 | 标签 `akita-ksu-v3.3.1`，提交 `87849999f8c57829714aa6eed10602b3aad8d429` [cite:9] |
| `zzpC/akita-build-manifests` | local manifest 与配置备份 | `main` 分支 [cite:10] |

> **重要：**本文固定到具体提交，不要求跟随各 Fork 的最新分支。后续新增提交应先构建验证，再更新固定值。

**2026-11-13 实机验证结果：**

- 成功刷入 `lineage-24.0-20261113-UNOFFICIAL-akita.zip`，系统为 Android 17。
- 内核 `Image` 的内嵌配置包含 `CONFIG_KSU=y`，`System.map` 中有 231 个 `ksu_*` 符号。
- KernelSU Manager `v3.3.1` 显示“工作中 / GKI / 32680-5”，SELinux 为强制执行。
- Manager 中“自 v3.0.0 起 GKI 工作模式仅用于测试环境”的横幅是模式提醒，不是构建或 root 失败。

其他机器复刻时应以本文的验证条件为准；构建日期、ZIP 文件名和 SHA-256 不要求与本次完全一致。

## 2. 定制代码与官方源码的关系

完整构建由两个源码工作区组成：

```text
LineageOS 官方主源码
  ├─ device/google/akita-kernels/6.1/   ← 接收自编译内核产物
  └─ 生成 LineageOS Recovery 与 OTA ZIP

GS-6.1 内核工作区
  ├─ common/gki                         ← AOSP + 个人 KernelSU 集成
  ├─ private/devices                    ← 个人 akita Kleaf 配置
  ├─ private/google-modules             ← LineageOS 官方设备模块
  └─ common/gki/KernelSU                ← manifest 固定的参考源码
```

个人 devices 配置包含 `build:akita --nouse_prebuilt_kernel`，因此 Kleaf 会从源码构建 GKI，而不是直接采用预编译内核。[cite:8]

GKI 中的 `drivers/kernelsu` 是实际参与内核编译的 KernelSU 源码。独立 KernelSU 仓库用于版本参考、Manager、用户空间组件和后续升级，不应在两处分别独立修改同一功能。[cite:7][cite:9]

## 3. 刷机前必读

### 3.1 数据、账号与 Bootloader

- 备份个人数据、照片和重要文件。解锁 Bootloader 和格式化数据都会清除用户数据。
- 首次启动官方系统后，先确认通话、移动网络、相机、Wi-Fi 等功能正常。
- 移除设备上的 Google 账号，避免恢复出厂设置后触发 FRP。[cite:3]
- 已经解锁 Bootloader 的设备不要重复解锁。
- **自编译 LineageOS 使用测试密钥时，不要重新锁定 Bootloader。**重新上锁可能导致设备无法启动；LineageOS 官方也明确警告重新上锁风险。[cite:5]

### 3.2 固件与版本边界

- 开始前应具备 Android 17 官方固件，不能用其他定制 ROM 的版本号代替这一确认。[cite:3][cite:4]
- 不要混刷 Android 16 与 Android 17 的内核、启动镜像或厂商分区。
- `boot.img`、`dtbo.img`、`vendor_kernel_boot.img`、`vendor_boot.img` 和 OTA ZIP 应来自同一套构建，避免新旧模块不匹配。

### 3.3 主机环境

官方构建指南建议准备 Linux 构建环境、`adb`、`fastboot`、`repo`、Git、Python 3 和足够磁盘空间；LineageOS 21 及以上建议 64 GB RAM、约 400 GB 可用存储，额外缓存和多设备构建需要更多空间。[cite:1]

按官方指南安装构建依赖、Platform-Tools 和 `repo`，并配置 Git 作者信息。另需 `cpio`、`lz4`、`unzip` 和 `rsync`，用于处理官方镜像与 Recovery ramdisk。可选启用 `ccache`：

```bash
export USE_CCACHE=1
export CCACHE_EXEC=/usr/bin/ccache
ccache -M 100G
```

以下命令使用当前目录布局；路径不同时需要相应调整：

```bash
export ANDROID_ROOT=/mnt/Disk2T/android
export LINEAGE_ROOT=$ANDROID_ROOT/lineage
export KERNEL_WS=$LINEAGE_ROOT/out-kernel/google/gs-6.1
export MANIFESTS=$ANDROID_ROOT/akita-build-manifests
export FIRMWARE_ROOT=$ANDROID_ROOT/firmware
export SKIP_KERNEL_SYNC=1
mkdir -p "$FIRMWARE_ROOT"
```

`SKIP_KERNEL_SYNC=1` 只阻止 LineageOS 的 lunch/brunch 自动重新同步内核工作区，不影响后文手动执行的 `repo sync`。

## 4. 获取配置备份与主 ROM 官方源码

### 4.1 获取个人配置备份

```bash
cd "$ANDROID_ROOT"
git clone https://github.com/zzpC/akita-build-manifests.git
```

该仓库保存 local manifest 和镜像/浅克隆配置差异。`default.xml.local-changes.patch` 用于记录历史环境配置，不建议直接覆盖或应用到同步中的 `default.xml`。[cite:10]

### 4.2 初始化 LineageOS 24.0

```bash
mkdir -p "$LINEAGE_ROOT"
cd "$LINEAGE_ROOT"
repo init -u https://github.com/LineageOS/android.git \
  -b lineage-24.0 --git-lfs --no-clone-bundle
```

官方初始化命令与分支以此为准。[cite:1]

### 4.3 恢复 akita local manifest

```bash
mkdir -p "$LINEAGE_ROOT/.repo/local_manifests"
cp "$MANIFESTS/lineage-24.0/local_manifests/roomservice.xml" \
  "$LINEAGE_ROOT/.repo/local_manifests/roomservice.xml"
```

该 manifest 添加 `device/google/akita`、`device/google/zuma` 和 `device/google/akita-kernels` 三个官方项目。[cite:10]

### 4.4 同步源码

```bash
cd "$LINEAGE_ROOT"
repo sync
```

官方建议优先使用 manifest 默认的同步配置；网络不稳定时可适当降低并行数。[cite:1]

## 5. 初始化内核工作区并整合个人 Fork

### 5.1 初始化 GS-6.1 内核源码

```bash
mkdir -p "$KERNEL_WS"
cd "$KERNEL_WS"
repo init -u https://github.com/LineageOS/android_kernel_google_gs-6.1_manifest.git \
  -b lineage-24.0
```

内核 manifest 将 GKI 固定到 `android14-6.1-2026-03_r25`，并包含 devices、google-modules、Bazel 和工具链项目。[cite:11]

### 5.2 恢复 KernelSU 固定版本配置

```bash
mkdir -p "$KERNEL_WS/.repo/local_manifests"
cp "$MANIFESTS/kernel-gs-6.1/local_manifests/kernelsu.xml" \
  "$KERNEL_WS/.repo/local_manifests/kernelsu.xml"
```

当前配置从 `https://github.com/zzpC/KernelSU` 获取代码，并固定到 `87849999f8c57829714aa6eed10602b3aad8d429`，不会自动跟随后续分支更新。[cite:9][cite:10]

### 5.3 同步内核源码

```bash
cd "$KERNEL_WS"
repo sync
```

### 5.4 切换 GKI 到个人定制提交

```bash
cd "$KERNEL_WS/common/gki"
git remote add fork https://github.com/zzpC/kernel_common.git
git fetch fork akita-ksu
git switch -c akita-ksu 9d4c6b2a74cc1889a4f125e878748e70e9b712a3
git branch --set-upstream-to=fork/akita-ksu akita-ksu
```

若本地已经存在 `akita-ksu`，不要重复创建；改为检查当前提交：

```bash
git switch akita-ksu
git rev-parse HEAD
```

输出应为 `9d4c6b2a74cc1889a4f125e878748e70e9b712a3`。该提交位于真实 AOSP 标签之上，不再使用源码快照根提交。[cite:7]

### 5.5 切换 devices 到个人定制提交

```bash
cd "$KERNEL_WS/private/devices"
git remote add fork https://github.com/zzpC/android_kernel_google_gs-6.1_devices.git
git fetch fork akita-ksu
git switch -c akita-ksu 69d8014525a927ba640e33115630a380a7d272ab
git branch --set-upstream-to=fork/akita-ksu akita-ksu
```

同样，已有分支时只需切换并核对提交。该版本明确启用源码构建 GKI。[cite:8]

> 后续进行上述定制构建时，继续使用 `SKIP_KERNEL_SYNC=1`，避免同步任务覆盖本地定制分支。需要升级时单独创建升级分支，不在当前工作区直接强制同步。

## 6. 编译并验证 akita 内核

在内核工作区根目录执行：

```bash
cd "$KERNEL_WS"
./build_akita.sh
```

脚本实际调用：

```bash
tools/bazel run --config=akita //private/devices/google/akita:akita/dist
```

主要产物位于 `$KERNEL_WS/out/akita/dist/`，包括：

- `Image`、`Image.lz4`、`System.map` 和 `.config`
- `boot.img`、`dtbo.img`、`vendor_kernel_boot.img`
- `system_dlkm.img`、`vendor_dlkm_staging_archive.tar.gz`
- 各类 `.ko` 模块、DTB、模块列表和符号文件

**不要执行 KernelSU 的 `setup.sh`。**固定提交已经完成驱动、Kconfig、Makefile、UAPI 和 Kleaf 集成，运行官方安装脚本可能自动拉取或切换版本，破坏本地定制。

### 6.1 必须验证实际 Image，不能只看 defconfig

构建成功后执行：

```bash
"$KERNEL_WS/common/gki/scripts/extract-ikconfig" \
  "$KERNEL_WS/out/akita/dist/Image" | \
  grep -E '^CONFIG_(KSU|KPROBES|KPROBE_EVENTS|EXT4_FS)=y$'

grep -cE ' ksu_[A-Za-z0-9_]+$' \
  "$KERNEL_WS/out/akita/dist/System.map"
```

第一条命令应输出 4 行配置，第二条命令的符号数必须大于 0。本次实测为 231；后续版本数量可以不同。

**只有 `akita_defconfig` 或 dist `.config` 里出现 `CONFIG_KSU=y` 还不够。**如果 Kleaf 错用预构建 GKI，配置可能正确，但最终 `Image` 仍不含 KernelSU。个人 devices 提交中的 `build:akita --nouse_prebuilt_kernel` 正是用于避免这一问题。[cite:8]

如果配置缺失或符号数为 0，应停止后续刷机，核对：

1. `common/gki` 是否为 `9d4c6b2a74cc1889a4f125e878748e70e9b712a3`。
2. `private/devices` 是否为 `69d8014525a927ba640e33115630a380a7d272ab`。
3. `private/devices/google/akita/device.bazelrc` 是否包含 `build:akita --nouse_prebuilt_kernel`。
4. `common/gki/drivers/kernelsu/uapi` 是否存在；缺少 `uapi/app_profile.h` 会导致源码内核编译失败。

## 7. 准备 blobs、编译完整 ROM 并验证衔接

### 7.1 先离线提取厂商专有文件

官方构建指南可以从设备提取 blobs，但官方 `user` 系统通常不允许 `adb root`，直接读取 `/vendor` 可能出现大量 `Permission denied`。本次可复现的做法是：**从与源码基线匹配的官方 LineageOS 包离线提取，并单独补齐 Recovery ramdisk 中的 HAL。**

从 LineageOS 官方渠道下载同一构建日期的 OTA ZIP 和 Recovery 镜像。若 Recovery 以 ZIP 提供，先解压并取其中的 `vendor_boot.img`。本文示例为 2026-11-11：

```text
$FIRMWARE_ROOT/lineage-24.0-20261111-nightly-akita-signed.zip
$FIRMWARE_ROOT/vendor_boot-official-20261111.img
```

先提取主包：

```bash
cd "$LINEAGE_ROOT/device/google/akita/akita"
./extract-files.py -n \
  "$FIRMWARE_ROOT/lineage-24.0-20261111-nightly-akita-signed.zip"
```

再将官方 `vendor_boot.img` 放入独立目录，让 extract-utils 自动解开 Recovery ramdisk，并只处理两个 HAL section：

```bash
rm -rf /tmp/akita-recovery-blobs
mkdir -p /tmp/akita-recovery-blobs
cp "$FIRMWARE_ROOT/vendor_boot-official-20261111.img" \
  /tmp/akita-recovery-blobs/vendor_boot.img

./extract-files.py -n -s '*HAL' /tmp/akita-recovery-blobs
```

`-s` 使用的是 `fnmatch` 模式，必须写成 `'*HAL'`，不要写成正则表达式。提取结果应位于 `vendor/google/akita`。至少检查：

```bash
ls "$LINEAGE_ROOT/vendor/google/akita/akita-vendor.mk" \
  "$LINEAGE_ROOT/vendor/google/akita/proprietary/recovery/system/bin/hw/android.hardware.boot-service.default_recovery-pixel" \
  "$LINEAGE_ROOT/vendor/google/akita/proprietary/recovery/system/bin/hw/android.hardware.health-service.zuma_recovery"
```

如果换用其他日期的官方包，OTA ZIP 与 `vendor_boot.img` 必须来自同一构建。不要把官方 Android 17 测试版固件、其他日期的 LineageOS 包和本次自编译镜像混合提取。[cite:1][cite:3]

### 7.2 使用 brunch 自动接入内核并编译 ROM

先完成 blobs 提取，再执行：

```bash
cd "$LINEAGE_ROOT"
source build/envsetup.sh
export USE_CCACHE=1 CCACHE_EXEC=/usr/bin/ccache
export SKIP_KERNEL_SYNC=1
brunch akita
```

**不能省略 `SKIP_KERNEL_SYNC=1`。**`brunch` 中的 breakfast/lunch 会调用 `build_kernel`；未设置该变量时，它会执行 `repo sync --detach --force-sync`，可能覆盖个人 GKI 和 devices 分支。

在当前 LineageOS 24.0 中，内核构建成功后会自动将 `$KERNEL_WS/out/akita/dist/` 复制到 `device/google/akita-kernels/6.1/`，因此推荐直接使用上述流程，不需要手工 `rsync`。

如果确实要手工同步 Kleaf 产物，则必须同时设置 `SKIP_KERNEL_BUILD=1`，否则 brunch 会再次构建并覆盖手工复制的内容。主 ROM 的 `.gitignore` 会排除大部分内核二进制生成物，不需要为了这些文件 Fork `akita-kernels`。[cite:14]

### 7.3 验证最终 ROM 中的内核

构建成功后执行：

```bash
export OUT="$LINEAGE_ROOT/out/target/product/akita"
cd "$OUT"
ROM_ZIP=$(ls -1t "$OUT"/lineage-24.0-*-UNOFFICIAL-akita.zip | head -n 1)
unzip -tq "$ROM_ZIP"
```

ZIP 检查应输出 `No errors detected`。然后解出最终 `boot.img` 中的内核，与源码构建的 `Image.lz4` 比较：

```bash
rm -rf /tmp/akita-boot-check
mkdir -p /tmp/akita-boot-check
"$LINEAGE_ROOT/out/host/linux-x86/bin/unpack_bootimg" \
  --boot_img "$OUT/boot.img" --out /tmp/akita-boot-check

sha256sum /tmp/akita-boot-check/kernel \
  "$KERNEL_WS/out/akita/dist/Image.lz4"

grep -cE ' ksu_[A-Za-z0-9_]+$' \
  "$LINEAGE_ROOT/device/google/akita-kernels/6.1/System.map"
```

两个 SHA-256 必须一致，KSU 符号数必须大于 0。**最终 ROM 的 `boot.img` 才是实际刷机对象，不能只比较 `dist/boot.img` 与预编译目录中的同名文件。**

刷机所需产物为：

- `boot.img`、`dtbo.img`、`vendor_kernel_boot.img`
- `vendor_boot.img`：LineageOS Recovery 镜像
- `lineage-24.0-*-UNOFFICIAL-akita.zip`：完整安装包 [cite:1]

## 8. 刷入 Pixel 8a

### 8.1 进入 Bootloader 并按需解锁

确保电量高于 50%、数据已备份、Google 账号已移除，然后执行：

```bash
adb reboot bootloader
fastboot getvar product
fastboot getvar current-slot
```

`product` 必须为 `akita`。记录当前槽位，fastboot 默认向该槽位写入启动镜像。

仅在 Bootloader 尚未解锁时执行：

```bash
fastboot flashing unlock
```

解锁会清除数据；已解锁时跳过。不要重新锁定。[cite:4][cite:5]

### 8.2 刷入启动镜像与 Recovery

akita 官方元数据列出的启动栈为 `boot`、`dtbo`、`vendor_kernel_boot`，Recovery 分区为 `vendor_boot`。[cite:4]

在 `$OUT` 下执行：

```bash
fastboot flash boot "$OUT/boot.img"
fastboot flash dtbo "$OUT/dtbo.img"
fastboot flash vendor_kernel_boot "$OUT/vendor_kernel_boot.img"
fastboot reboot bootloader
fastboot flash vendor_boot "$OUT/vendor_boot.img"
fastboot reboot recovery
```

这些镜像必须来自同一套定制构建，不能用旧官方镜像与新的 OTA ZIP 混合。本已验证流程不额外刷写 bootloader、基带或其他固件分区。

如果 `fastboot reboot recovery` 未进入 Recovery，也可以通过 Bootloader 菜单选择 **Recovery Mode**，或关机后按住 Volume Down + Power 再选择 Recovery。[cite:4]

### 8.3 首次安装时格式化数据

从官方系统、其他 ROM 或不同签名体系迁移时，在 Lineage Recovery 中依次选择 **Factory reset → Format data / Factory reset → Format data**。此操作会清除用户数据，执行前再次确认备份。

同一套定制系统的后续升级是否保留数据，应按具体升级路径验证，不在每次刷机时默认格式化。

### 8.4 通过 ADB Sideload 安装并重启

在 Recovery 中选择 **Apply update → Apply from ADB**，然后检查设备状态：

```bash
adb devices
```

设备应显示为 `sideload`。执行：

```bash
adb -d sideload "$ROM_ZIP"
```

如果新打开了终端，请将 `$ROM_ZIP` 替换为实际 ZIP 的绝对路径。Google 的 OTA 指南同样要求先进入 Apply update from ADB，再使用 `adb sideload <文件>`。[cite:13]

成功后电脑端应显示 `Total xfer: 1.00x`。手机屏幕显示 **Install completed / 安装完成** 后，选择 **Reboot system now**。如果 Recovery 报告设备不匹配、文件损坏或安装失败，不能忽略错误继续重启。

离开 sideload 后 `adb devices` 可能显示 `unauthorized`，此时直接操作 Recovery 界面即可。首次启动完成初始设置后，需要重新开启 USB 调试并授权电脑，才能安装 Manager。[cite:6]

## 9. 安装并验证 KernelSU Manager

**内置 KernelSU 内核不会自动在桌面安装 Manager。**从 KernelSU 官方 `v3.3.1` 发布页获取匹配的 APK，在系统启动并重新授权 ADB 后安装：[cite:12]

```bash
adb install -r /path/to/KernelSU_v3.3.1_32680.apk
adb shell pm path me.weishu.kernelsu
```

安装命令应返回 `Success`，`pm path` 应返回 Manager 的安装路径。启动 Manager 后检查：

1. 首页显示 **工作中**，模式为 **GKI**。
2. 内核版本显示 `32680` 或本次实测的 `32680-5`，Manager 为 `v3.3.1`。
3. SELinux 保持 **强制执行**。
4. 仅向可信应用授予 root。

还可以检查运行中内核：

```bash
adb shell uname -a
adb shell 'zcat /proc/config.gz | grep -E "^CONFIG_(KSU|KPROBES|KPROBE_EVENTS)=y$"'
```

普通 ADB shell 默认不等于已获 KernelSU root，读取 `/proc/kallsyms` 被拒绝也不能说明 KernelSU 失效。只有在明确启用 su 兼容并授权 shell 后，才使用 `adb shell su -c id` 作为附加验证；也可以在已授权的终端应用中执行 `su -c id`。

同时检查 Wi-Fi、相机、指纹、通话和移动网络，确认内核与厂商模块工作正常。

## 10. 长期维护建议

### 10.1 日常个人修改

- GKI、devices、KernelSU 和 manifest 分别在对应 Git 项目提交。
- 每个项目使用个人 Fork 的定制分支，不直接修改上游基线分支。
- 新提交先推送个人 Fork，再更新经过验证的固定提交记录。
- 不把 `out/`、Bazel 输出、OTA ZIP、`.img`、`.ko` 等生成物提交到源码仓库。

### 10.2 上游升级与 OTA

```text
获取新的 AOSP / LineageOS / KernelSU 基线
→ 创建升级分支
→ 迁移个人提交
→ 完整编译内核和 ROM
→ 验证实际 Image 和最终 boot.img 均含 KernelSU
→ 刷机验证
→ 推送并标记验证版本
→ 更新 manifest 固定提交
```

KernelSU 升级还需同步检查 Kleaf 版本号、UAPI、Manager 和 GKI 内置源码，不能只更新独立仓库分支名称。

**当前采用 GKI 内置模式，直接安装普通 LineageOS OTA 会替换目标槽位的 boot 内核并丢失 KernelSU/root。**升级时应重新构建并刷入含 KSU 的完整 ROM，或在确认新内核兼容后单独恢复定制 boot 镜像。不要只升级 Manager 而忽略内核与 UAPI 的匹配。

Manager 的 GKI 横幅表示官方更推荐手机采用便于 OTA 和卸载的 LKM 模式，并不表示当前内置模式失效。若后续更看重 OTA 便利性，可另行评估 LKM；这与本次已验证的 GKI 构建是两条不同安装路径。[cite:15]

### 10.3 不建议的操作

- 不要直接修改 `.repo/manifests/default.xml` 作为长期配置。
- 不要在定制工作区无条件执行 `repo sync` 后继续使用旧分支。
- 不要在主 ROM 构建时省略 `SKIP_KERNEL_SYNC=1`。
- 不要执行 KernelSU `setup.sh` 覆盖固定集成。
- 不要把 KernelSU 官方 `main` 的更新直接合并到稳定版本而不验证。
- 不要混用不同 Android 版本、不同构建批次或不同设备的启动镜像。
- 不要在自编译测试密钥系统上重新锁定 Bootloader。

## 11. 实测问题与排查

| 现象 | 原因与处理 |
|---|---|
| defconfig 有 `CONFIG_KSU=y`，但 `Image` 没有 KSU | Kleaf 默认使用预构建 GKI。确认 akita `device.bazelrc` 含 `--nouse_prebuilt_kernel`，并检查 `System.map` 符号。 |
| 编译提示 `uapi/app_profile.h` 不存在 | KernelSU 根目录的 `uapi` 未纳入内核源码。使用固定 GKI 提交，确认 `drivers/kernelsu/uapi` 完整，不要重新运行 `setup.sh`。 |
| breakfast 报 `akita-vendor.mk` 不存在 | blobs 尚未提取或提取目录错误。先完成第 7.1 节，再运行 brunch。 |
| ADB 提取大量 `Permission denied` | 官方 `user` 系统不能 `adb root`。改用官方 OTA ZIP 和 `vendor_boot.img` 离线提取。 |
| Recovery 中 `adb unauthorized` | 直接通过屏幕选择 Format data、Apply from ADB 或 Reboot system now；只有 `sideload` 状态需要电脑推送 ZIP。 |
| 刷机后桌面没有 KernelSU | 内核支持不等于预装 Manager，按第 9 节单独安装 APK。 |
| Manager 提示 GKI 仅用于测试 | 这是内置模式的固定提示；首页显示“工作中”即表示已识别内核。 |
| 内核日志出现 `realpath ... common/gki` 或 `gcip-mailbox.c` 解析信息 | 若最终仍有 `Build completed successfully`，这些信息不是本次失败原因；真正的 `FAILED` / `Error` 才需要中止排查。 |

## Sources

<a id="cite-1"></a>[cite:1] [LineageOS：Build LineageOS for Google Pixel 8a](https://wiki.lineageos.org/devices/akita/build/)

<a id="cite-2"></a>[cite:2] [LineageOS：Google Pixel 8a 设备信息](https://wiki.lineageos.org/devices/akita/)

<a id="cite-3"></a>[cite:3] [LineageOS：Install LineageOS on Google Pixel 8a](https://wiki.lineageos.org/devices/akita/install/)

<a id="cite-4"></a>[cite:4] [LineageOS Wiki：akita 官方设备元数据](https://github.com/LineageOS/lineage_wiki/blob/main/_data/devices/akita.yml)

<a id="cite-5"></a>[cite:5] [LineageOS：常见问题与 Bootloader 重新上锁风险](https://wiki.lineageos.org/faq)

<a id="cite-6"></a>[cite:6] [LineageOS：使用 Lineage Flash Tools](https://wiki.lineageos.org/web_flash_tools)

<a id="cite-7"></a>[cite:7] [zzpC/kernel_common：akita-ksu](https://github.com/zzpC/kernel_common/tree/akita-ksu)

<a id="cite-8"></a>[cite:8] [zzpC/android_kernel_google_gs-6.1_devices：akita-ksu](https://github.com/zzpC/android_kernel_google_gs-6.1_devices/tree/akita-ksu)

<a id="cite-9"></a>[cite:9] [zzpC/KernelSU：akita-ksu-v3.3.1](https://github.com/zzpC/KernelSU/tree/akita-ksu-v3.3.1)

<a id="cite-10"></a>[cite:10] [zzpC/akita-build-manifests](https://github.com/zzpC/akita-build-manifests)

<a id="cite-11"></a>[cite:11] [LineageOS：android_kernel_google_gs-6.1_manifest](https://github.com/LineageOS/android_kernel_google_gs-6.1_manifest)

<a id="cite-12"></a>[cite:12] [KernelSU：v3.3.1 发布页](https://github.com/tiann/KernelSU/releases/tag/v3.3.1)

<a id="cite-13"></a>[cite:13] [Google：Pixel OTA 镜像与 ADB Sideload 指南](https://developers.google.cn/android/ota?hl=en)

<a id="cite-14"></a>[cite:14] [LineageOS：akita-kernels 的 .gitignore](https://github.com/LineageOS/android_device_google_akita-kernels/blob/lineage-24.0/.gitignore)

<a id="cite-15"></a>[cite:15] [KernelSU：安装指南（GKI 与 LKM 模式）](https://kernelsu.org/guide/installation.html)
