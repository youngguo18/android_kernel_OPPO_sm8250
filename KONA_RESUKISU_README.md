# kona BakaSU/ReSukiSU 集成说明（stock/13.1）

本分支在 `youngguo18/android_kernel_OPPO_sm8250` 的 **stock/13.1**（Linux 4.19.160，OPPO sm8250 / kona）
基础上集成 **ReSukiSU / BakaSU**（KernelSU 的 non-GKI fork），并修正了原树的旧 KernelSU 残留。

适用设备：OPPO Reno5 Pro+（PDRM00）/ ColorOS 13.1（作者声明 OPPO 骁龙865 全系通用）。

---

## 一、内核侧改了什么（本仓库内的全部改动）

### 1) 清理原树遗留的旧 KernelSU + SuSFS（关键！）

原 `stock/13.1` 树里躺着 2024 年的两套旧东西，会与 BakaSU 冲突（表现为**刷入后卡开机 logo**）：

| 原提交 | 内容 |
|---|---|
| `a8728a80531`（backslashxx，2024-02-02） | 旧 KernelSU 手动文件系统钩子 + selinux hook |
| `ba468635a48`（youngguo18，2024-11-16） | 为旧 KSU 开配置 |
| `5d6ba1ee48e` / `c34569b4d7c`（Bruce Teng，2024-11~12） | SuSFS 1.5.x 整套补丁（21 个文件） |

本分支已将以上 4 个提交反向 revert（见提交记录 `清理旧 KernelSU 手动钩子 + SuSFS`）：
删除 `fs/susfs.c`、`fs/sus_su.c`、`include/linux/susfs.h` 等，并移除 `fs/*`、`security/selinux/hooks.c`、
`drivers/input/input.c` 中的旧钩子。

**不兼容项**（应保持为 0，BakaSU 会检查）：
`ksu_vfs_read_hook`（fs/read_write.c）、`is_ksu_transition`（security/selinux/hooks.c）、`ksu_handle_rename`（security/security.c）。

### 2) 打上 BakaSU 要求的 7 个手动钩子

钩子签名与 BakaSU 源码（`KernelSU/kernel/feature/sucompat.c`、`runtime/ksud_integration.c`、
`supercall/supercall.c`）**逐字一致**：

| 钩子 | 位置 | 说明 |
|---|---|---|
| `ksu_handle_execveat` | `fs/exec.c` | `__do_execve_file()` 开头 |
| `ksu_handle_faccessat` | `fs/open.c` | `do_faccessat()` 开头 |
| `ksu_handle_stat` | `fs/stat.c` | `SYSCALL_DEFINE4(newfstatat)` 开头 |
| `ksu_handle_newfstat_ret` | `fs/stat.c` | `SYSCALL_DEFINE2(newfstat)` 内 `cp_new_stat()` 之后 |
| `ksu_handle_fstat64_ret` | `fs/stat.c` | `SYSCALL_DEFINE2(fstat64)` 内 `cp_new_stat64()` 之后 |
| `ksu_handle_sys_reboot` | `kernel/reboot.c` | `SYSCALL_DEFINE4(reboot)` 开头 |
| `ksu_handle_input_handle_event` | `drivers/input/input.c` | `input_handle_event()` 开头（配合 `KSU_MANUAL_HOOK_AUTO_INPUT_HOOK`） |

全部用 `#ifdef CONFIG_KSU_MANUAL_HOOK` 包裹。

> ⚠️ **为什么用手动钩子**：BakaSU 的 `CONFIG_KSU_TRACEPOINT_HOOK`（自动钩子）**仅支持 GKI 2.0**，
> 在 4.19 这类 Non-GKI 上构建时会直接报错终止：
> `*** TP hooks are incompatible with Non-GKI/GKI 1.0 kernels. Stop.`
> 所以 4.19 的**唯一可行方案是 `CONFIG_KSU_MANUAL_HOOK`**。

### 3) 驱动接线

```
drivers/Kconfig     + source "drivers/kernelsu/Kconfig"
drivers/Makefile    + obj-$(CONFIG_KSU) += kernelsu/
```

### 4) 构建配置

| 文件 | 说明 |
|---|---|
| `arch/arm64/configs/kona-resukisu-defconfig` | **成品配方**：设备 dump 配置 + WiFi 驱动 + KSU 手动钩子 |
| `arch/arm64/configs/kona-op13-bakasu_defconfig` | 上一版（未含 WiFi 驱动），可作回退基线 |

---

## 二、编译前必须补上的 KernelSU 源码（本仓库不含）

KernelSU 源码**不在本仓库内**（避免重复维护，改为构建时拉取最新 ReSukiSU）。

```bash
cd <kernel-source>

# 1) 拉取 ReSukiSU/BakaSU（经 gh-proxy 加速）
git clone https://v4.gh-proxy.org/https://github.com/ReSukiSU/ReSukiSU.git KernelSU

# 2) 让内核构建能找到它（BakaSU 要求 KernelSU/.git 与其 kernel/ 同级存在）
ln -sfn ../KernelSU/kernel drivers/kernelsu

# 3) 建议加入本地排除，避免误提交
printf 'KernelSU/\ndrivers/kernelsu\n' >> .git/info/exclude
```

已验证可用的版本：`v4.2.0-rc3-36-g63268b9b`（commit `63268b9b`）。

---

## 三、构建步骤（实测可开机）

### 工具链

使用 **ZyC clang 11.1.0** 整包（与作者 youngguo18 一致，内含 LLD 11.1.0 与 binutils 2.39.50）：

```bash
curl -L -o zyc.tar.gz \
  "https://v4.gh-proxy.org/https://github.com/ZyCromerZ/Clang/releases/download/11.1.0-20220724-release/Clang-11.1.0-20220724.tar.gz"
mkdir -p toolchains && tar -xzf zyc.tar.gz -C toolchains && mv toolchains/<解压目录> toolchains/zyc-clang-11.1.0
```

### 配置与编译

```bash
export PATH="$PWD/toolchains/zyc-clang-11.1.0/bin:$PATH"
export ARCH=arm64 SUBARCH=arm64
export CROSS_COMPILE=aarch64-linux-gnu-
export CROSS_COMPILE_ARM32=arm-linux-gnueabi-
MK="CC=clang LD=ld.lld AR=llvm-ar NM=llvm-nm OBJCOPY=llvm-objcopy \
    OBJDUMP=llvm-objdump READELF=llvm-readelf STRIP=llvm-strip LLVM=1 LLVM_IAS=1"

# 注意：本树上 `make <defconfig名>` 的 kbuild 目标解析不可靠，
# 直接铺 .config 再 olddefconfig 最稳：
mkdir -p out && cp arch/arm64/configs/kona-resukisu-defconfig out/.config
make O=out ARCH=arm64 $MK olddefconfig
make O=out ARCH=arm64 $MK -j16 Image.gz
```

产物：`out/arch/arm64/boot/Image.gz`

### 打包（AnyKernel3）

把 `Image.gz` 放到 AK3 目录根部打包即可（`anykernel.sh` 中 `block=.../by-name/boot`、`do.devicecheck=0`）。

---

## 四、关键配置说明（WiFi 等）

构建配置以**设备 dump 的 `oplus-perf_defconfig`** 为基底，但**必须补上以下项**——
因为设备 dump 生成于 OPPO 加入这些符号之前，而它们的 Kconfig 默认值是 `default n`，
`savedefconfig`/`olddefconfig` 会**静默关掉**，导致对应功能"编译成功但硬件失效"：

```
CONFIG_QCA_CLD_WLAN=y                 # WiFi 驱动本体（缺失 → WiFi/热点不可用）
CONFIG_QCA_CLD_WLAN_PROFILE="qca6390" # 本机为 QCA6390
CONFIG_CNSS_GENL=y
```

> 来源：`arch/arm64/configs/vendor/kona-perf_defconfig`（875 行的权威配置）。
> 经验：**defconfig 以 vendor 那份为准，设备 dump 仅作参考/校验**（两者的缺项不同）。

另有 12 个符号设备配置里有、但**本树源码中从不存在**（235,734 个提交的历史里都没出现过），
属功能缺失（开机原因记录、虚拟网卡、NFC 型号、触摸算法等），不影响开机：

`OPLUS_FEATURE_LAST_BOOT_REASON`、`OPLUS_FEATURE_DATA_MODULE`、`OPLUS_FEATURE_ABNORMAL_FLAG`、
`OPLUS_FEATURE_VIRTUAL_NET`、`OPLUS_FEATURE_QCOM_SMEM_INTI`、`OPLUS_FEATURE_AUDIO_CAMUX_OFF`、
`OPLUS_CPU_AUDIO_PERF`、`OPLUS_DYNAMIC_CONFIG`、`OPLUS_LOCKING_STRATEGY`、`OPLUS_SS_LOCKER_OPT`、
`CONFIG_LOCKING_PROTECT`、`CONFIG_TOUCHPANEL_ALGORITHM`

---

## 五、已知状态

| 功能 | 状态 |
|---|---|
| 开机、KernelSU（root、重启后仍生效） | ✅ 实测通过 |
| 通话/免提、4G/5G 数据、关机充电与充电图标、耳机/扬声器、OTG、亮度与自动旋转 | ✅ 实测通过 |
| 触摸、指纹、蓝牙、NFC、相机、陀螺仪 | ✅ 实测通过 |
| WiFi / 热点 | 🔧 已在 `kona-resukisu-defconfig` 补 `QCA_CLD_WLAN` 修复（待复测） |
| GPS | ⚠️ 能定位、导航拿不到最新定位；配置层与设备一致，需设备日志进一步定位 |
