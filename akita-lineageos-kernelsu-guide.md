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

按官方指南安装构建依赖、Platform-Tools 和 `repo`，并配置 Git 作者信息。可选启用 `ccache`：

```bash
export USE_CCACHE=1
export CCACHE_EXEC=/usr/bin/ccache
ccache -M 50G
```

以下命令使用当前目录布局；路径不同时需要相应调整：

```bash
export ANDROID_ROOT=/mnt/Disk2T/android
export LINEAGE_ROOT=$ANDROID_ROOT/lineage
export KERNEL_WS=$LINEAGE_ROOT/out-kernel/google/gs-6.1
export MANIFESTS=$ANDROID_ROOT/akita-build-manifests
```

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
git remote add fork git@github.com:zzpC/kernel_common.git
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
git remote add fork git@github.com:zzpC/android_kernel_google_gs-6.1_devices.git
git fetch fork akita-ksu
git switch -c akita-ksu 69d8014525a927ba640e33115630a380a7d272ab
git branch --set-upstream-to=fork/akita-ksu akita-ksu
```

同样，已有分支时只需切换并核对提交。该版本明确启用源码构建 GKI。[cite:8]

> 后续进行上述定制构建时，继续使用 `SKIP_KERNEL_SYNC=1`，避免同步任务覆盖本地定制分支。需要升级时单独创建升级分支，不在当前工作区直接强制同步。

## 6. 编译 akita 内核

在内核工作区根目录执行：

```bash
cd "$KERNEL_WS"
SKIP_KERNEL_SYNC=1 ./build_akita.sh
```

脚本实际调用：

```bash
tools/bazel run --config=akita //private/devices/google/akita:akita/dist
```

主要产物位于：

```text
$KERNEL_WS/out/akita/dist/
```

其中包括：

- `boot.img`
- `dtbo.img`
- `vendor_kernel_boot.img`
- `system_dlkm.img`
- `vendor_dlkm_staging_archive.tar.gz`
- 各类 `.ko` 模块、DTB、模块列表和符号文件

**不要执行 KernelSU 的 `setup.sh`。**当前定制已经完成驱动、Kconfig、Makefile 和 Kleaf 集成，运行官方安装脚本可能自动拉取或切换版本，破坏固定提交和本地修改。

## 7. 将内核产物接入主 ROM

### 7.1 同步 Kleaf 产物

```bash
rsync -a "$KERNEL_WS/out/akita/dist/" \
  "$LINEAGE_ROOT/device/google/akita-kernels/6.1/"
```

这里不使用 `--delete`，避免误删目录中需要保留的配置文件。

主 ROM 的 `.gitignore` 会排除 `.img`、`.ko`、DTB 和多数内核构建产物，因此这些二进制通常不会出现在普通 `git status` 中，也不需要为了它们 Fork `akita-kernels`。[cite:14]

### 7.2 验证产物衔接

```bash
sha256sum "$KERNEL_WS/out/akita/dist/boot.img" \
  "$LINEAGE_ROOT/device/google/akita-kernels/6.1/boot.img"
```

两个 SHA-256 必须一致。还可检查 `vendor_kernel_boot.img` 和 `dtbo.img`，防止主 ROM 意外使用旧内核。

### 7.3 准备主 ROM 构建环境与厂商文件

```bash
cd "$LINEAGE_ROOT"
source build/envsetup.sh
breakfast akita
```

按照官方指南连接已开启 ADB 的设备，在 `device/google/akita` 中运行 `extract-files.py`；若该脚本不存在，则运行 `extract-files.sh`。提取结果应进入 `vendor/google`。[cite:1]

### 7.4 编译完整 ROM

```bash
cd "$LINEAGE_ROOT"
croot
brunch akita
```

构建成功后进入输出目录：

```bash
cd "$OUT"
```

重点产物为：

- `vendor_boot.img`：LineageOS Recovery 镜像。
- `lineage-24.0-*-UNOFFICIAL-akita.zip`：完整安装包。[cite:1]

## 8. 刷入 Pixel 8a

### 8.1 进入 Bootloader 并按需解锁

```bash
adb reboot bootloader
fastboot devices
```

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
fastboot flash vendor_boot "$OUT/vendor_boot.img"
```

这些镜像必须来自同一套定制构建，不能用旧官方镜像与新的 OTA ZIP 混合。

完成后通过 Bootloader 菜单选择 **Recovery Mode**。官方元数据也支持关机后按住 Volume Down + Power，再从菜单进入 Recovery。[cite:4]

### 8.3 首次安装时格式化数据

从官方系统、其他 ROM 或不同签名体系迁移时，在 Lineage Recovery 中执行 **Factory reset / Format data**。此操作会清除用户数据，执行前再次确认备份。

同一套定制系统的后续升级是否保留数据，应按具体升级路径验证，不在每次刷机时默认格式化。

### 8.4 通过 ADB Sideload 安装

在 Recovery 中选择 **Apply update → Apply from ADB**，然后检查设备状态：

```bash
adb devices
```

设备应显示为 `sideload`。执行：

```bash
adb -d sideload "$OUT/lineage-24.0-YYYYMMDD-UNOFFICIAL-akita.zip"
```

将 `YYYYMMDD` 和文件名替换为实际产物。Google 的 OTA 指南同样要求先进入 Apply update from ADB，再使用 `adb sideload <文件>`。[cite:13]

如果不希望本地安装 ADB，也可以使用官方 Lineage Flash Tools 的 ADB 页面选择 ZIP 并执行 Sideload；启动镜像仍应按 akita 的设备要求处理。[cite:6]

若 Recovery 提示非官方签名或测试密钥警告，只有在确认 ZIP 是本次自行构建、设备代号和文件路径均正确时才继续。设备不匹配、文件损坏等错误不能忽略。

## 9. 安装并验证 KernelSU Manager

从 KernelSU 官方 `v3.3.1` 发布页获取匹配的 Manager APK，并安装：[cite:12]

```bash
adb install /path/to/KernelSU-v3.3.1.apk
```

启动 Manager 后检查：

1. 能识别 KernelSU 内核，版本与 `v3.3.1` / `32680` 对应。
2. 仅向可信应用授予 root。
3. 使用终端或 ADB 验证授权后的身份：

```bash
adb shell su -c id
```

同时检查 `uname -a`、Wi-Fi、相机、指纹、通话和移动网络，确认内核与厂商模块工作正常。

## 10. 长期维护建议

### 10.1 日常个人修改

- GKI、devices、KernelSU 和 manifest 分别在对应 Git 项目提交。
- 每个项目使用个人 Fork 的定制分支，不直接修改上游基线分支。
- 新提交先推送个人 Fork，再更新经过验证的固定提交记录。
- 不把 `out/`、Bazel 输出、OTA ZIP、`.img`、`.ko` 等生成物提交到源码仓库。

### 10.2 上游升级

```text
获取新的 AOSP / LineageOS / KernelSU 基线
→ 创建升级分支
→ 迁移个人提交
→ 完整编译内核和 ROM
→ 刷机验证
→ 推送并标记验证版本
→ 更新 manifest 固定提交
```

KernelSU 升级还需同步检查 Kleaf 版本号、UAPI、Manager 和 GKI 内置源码，不能只更新独立仓库分支名称。

### 10.3 不建议的操作

- 不要直接修改 `.repo/manifests/default.xml` 作为长期配置。
- 不要在定制工作区无条件执行 `repo sync` 后继续使用旧分支。
- 不要执行 KernelSU `setup.sh` 覆盖固定集成。
- 不要把 KernelSU 官方 `main` 的更新直接合并到稳定版本而不验证。
- 不要混用不同 Android 版本、不同构建批次或不同设备的启动镜像。

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
