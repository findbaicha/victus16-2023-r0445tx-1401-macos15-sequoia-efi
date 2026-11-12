# HP Victus 16-r0445tx · OpenCore EFI（macOS Sequoia）

> OpenCore EFI for **HP Victus 16-r0445tx** (2023), featuring **Intel AX211 native Wi-Fi** on macOS Sequoia.
>
> 本仓库提供一个可用的 OpenCore EFI，重点是让 **Intel AX211 在 macOS 里以原生方式工作**（不再需要 HeliPort）。
>
> 验证环境：i7-13620H · RTX 4060 Laptop · Intel AX211（`8086:51f1`）· Realtek ALC245 · macOS 15.8.2 (24H232) · OpenCore 1.0.9

---

## 状态

| 项目 | 状态 | 说明 |
| :-- | :--: | :-- |
| 显卡加速 | ✅ | RTX 4060，NullMoth 驱动，Metal 3，双屏 |
| **Wi-Fi（AX211）** | ✅ | **原生**，ForbiddenItlwm，不需要 HeliPort |
| 蓝牙 | ✅ | Intel AX211，固件 v12490 |
| 触控板 | ✅ | Synaptics SYNA32E3，轮询模式 |
| 音频输出 | ✅ | 内置喇叭 + 耳机 + USB（layout-id 11） |
| 电池 / 亮度 / USB | ✅ | |
| AirDrop / 无线 Sidecar | ❌ | Sequoia 上 AWDL 不可用，见[已知问题](#已知问题) |
| 睡眠唤醒 | ❌ | 固件级恢复失败 |

---

## ⚠️ 使用前必读

### 1. 必须自己生成 SMBIOS

本仓库已清除所有个人信息：

| 字段 | 当前值 |
| :-- | :-- |
| `SystemSerialNumber` | `SERIAL-HERE` |
| `MLB` | `MLB-HERE` |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` |
| `ROM` | `00 00 00 00 00 00` |

使用前请用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 生成自己的值，否则 iServices（iMessage / FaceTime / iCloud）不可用。

### 2. 显卡驱动不在本仓库内

RTX 4060 需要 **NullMoth**，需单独安装到：

```
/Library/Extensions/NVRM.kext
/Library/Extensions/NVRMFB.kext
/Library/Extensions/NVAccel.kext
/Library/Extensions/NVRMAGDC.kext
```

### 3. 睡眠未修复

```bash
sudo pmset -a disablesleep 1
```

### 4. 调试参数建议删除

`boot-args` 中的 `-v debug=0x100 keepsyms=1` 仅用于调试，日常使用建议移除。

---

## AX211 原生 Wi-Fi

### 为什么换掉 itlwm

常见的 `itlwm.kext` + HeliPort 方案自行实现了一套 Wi-Fi 栈，macOS 因此看不到真正的 AirPort 接口。而 [ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm) 让 AX211 接入 Apple 自带的 `IO80211Family`，于是 Wi-Fi 菜单、系统设置都是原生的。

| | itlwm + HeliPort | ForbiddenItlwm |
| :-- | :--: | :--: |
| Wi-Fi 菜单 | 需要 HeliPort | ✅ 系统原生 |
| 系统设置里的网络 | 非标准 | ✅ 和真 Mac 一样 |
| 上网 | ✅ | ✅ |
| AirDrop | ❌ | ❌（Sequoia，见下文） |

### 本 EFI 改了什么

**Kexts**：移除 `itlwm.kext`、`AirportItlwm.kext`，加入两个新驱动：

```
EFI/OC/Kexts/
├── ForbiddenItlwm.kext     # Sequoia 构建
└── BTLEAdvFixup.kext       # Sequoia 构建
```

**`Kernel → Add`**：顺序有依赖关系，不能乱：

| 序号 | BundlePath | MinKernel | MaxKernel |
| :--: | :-- | :-- | :-- |
| 1 | `Lilu.kext` | | |
| 2 | `IntelBluetoothFirmware.kext` | | |
| 3 | `BlueToolFixup.kext` | `21.0.0` | |
| 4 | `BTLEAdvFixup.kext` | `24.0.0` | `24.99.99` |
| 5 | `ForbiddenItlwm.kext` | `24.0.0` | `24.99.99` |

`IntelBTPatcher.kext` 已**禁用**——这是必须的，否则会和 BTLEAdvFixup 冲突。

**`boot-args`**：追加 AWDL 参数组：

```text
-btleadv -itlawdlcoexprobe -itlawdliface -itlawdlnativeup -itlawdlcoexrx -itlawdlcoextx
-itlawdlcoexmulti -itlawdlcoexdata -itlawdlmitx -itlawdlelicit -itlawdlcaps
-itlawdlsvcmerge -itlawdlawsync itlawdlintcoal=4 -itlawdlcoexbatx -itlawdlcoexpeerpause
-itlawdlcoexbatrace itlawdltxqueue=4096
```

完整参数说明见 [BOOT-FLAGS.md](https://github.com/retropineaple/ForbiddenItlwm/blob/main/docs/BOOT-FLAGS.md)。

### 验证

```bash
# 驱动是否加载
kextstat | grep -i forbidden

# 原生接口是否存在
networksetup -listallhardwareports | grep -A2 Wi-Fi
ifconfig en0 | grep "inet "
```

驱动的健康状态：

```bash
ioreg -l | grep -E '"ItlwmAWDL(BuildCheck|InterfaceBuild)"'
```

```text
"ItlwmAWDLBuildCheck"     = "verified"
"ItlwmAWDLInterfaceBuild" = "forbidden-0.5.0-test1-sequoia"
```

也可以直接跑项目自带的兼容性检查：

```bash
bash tools/compat/check_compat.sh
```

---

## 已知问题

### AirDrop 在 Sequoia 上不可用

两个方向都失败：Mac 搜不到 iPad / iPhone，对方也搜不到 Mac。

原因是 macOS 始终没有完成 `awdl0` 接口的创建，驱动拿不到发送所需的帧模板：

```bash
$ ifconfig awdl0
ifconfig: interface awdl0 does not exist
```

```bash
$ ioreg -l | grep ItlwmAWDLSyncProbeStage
"ItlwmAWDLSyncProbeStage" = "snapshot-not-ready"
```

```bash
$ ioreg -w0 -r -c ItlAWDLInterface -d 1 -l | grep ClientEvent
"ItlwmAWDLClientEventCount" = 2
"ItlwmAWDLClientEvents" = (
  "24785 ms open IOSkywalkNetworkBSDClient r=1 inactive=0 proc=kernel_task",
  "24785 ms close IOSkywalkNetworkBSDClient r=0 inactive=0 proc=kernel_task ..."
)
```

接口被打开后**同一毫秒内就被关闭**。这是上游已知问题，见 [issue #13](https://github.com/retropineaple/ForbiddenItlwm/issues/13)。普通 Wi-Fi 不受影响。

### Sidecar（iPad 作副屏）同样不可用

无线 Sidecar 依赖同一套 AWDL。有线 Sidecar 还需要 **VideoToolbox HEVC 编码器**，而本机核显无驱动、NVIDIA 驱动不提供 VideoToolbox 编码器。USB 配对成功、Finder 也能看到 iPad，但它不会出现在显示器列表里。

不依赖 AWDL 的替代方案：Deskreen（免费，浏览器方案）、Duet Display（付费，USB）。

### 信道 149 依赖国家码

AirDrop 想跑满速需要 149 信道（80 MHz），而固件要先从附近网络学到国家码。部分国家把 149–161 标为 passive。

### 升级 macOS 前先去 `-btleadv`

BTLEAdvFixup 会修改 `bluetoothd`，增量更新检测到文件被改动会失败。升级前先把 `-btleadv` 从 `boot-args` 移除，启动一次，升级完再加回来。

---

## 目录结构

```text
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/         # DSDT（已打补丁）+ SSDT
    ├── Drivers/      # HfsPlus、OpenRuntime、ResetNvramEntry、apfs_aligned、OpenCanopy
    ├── Kexts/        # 36 个 kext，其中 12 个在 config 里禁用
    ├── Resources/    # OpenCanopy 主题（Flavours-Sequoia）
    ├── Tools/
    ├── OpenCore.efi
    └── config.plist  # 已清理，无个人信息
```

ACPI 目录说明：

- `DSDT.aml` —— 固件 DSDT，只做了一处二进制补丁：把 `I2C0.TPD0._CRS` 改名为 `XCRS` 并重算校验和。这解决了 GSI 27 冲突导致 I2C 控制器无法启动的问题。
- `SSDT-TPD0CRS.aml` —— 为触控板提供新的 `_CRS`，只声明 `I2cSerialBus`、不声明中断，让控制器驱动能够启动，VoodooI2CHID 退回轮询模式。
- `SSDT-I2CTM.aml` —— I2C 时序，从约 178 kHz 提升到 400 kHz。

---

## 回滚

Wi-Fi 的改动只有三处：两个 kext 和 `config.plist`。

1. 删除 `EFI/OC/Kexts/ForbiddenItlwm.kext` 和 `BTLEAdvFixup.kext`
2. 恢复引用了 `itlwm.kext`（或 `AirportItlwm.kext`）的 `config.plist`
3. 把旧驱动放回 `EFI/OC/Kexts/`
4. 重启

建议把旧 `config.plist` 备份到 **FAT32 分区**，这样用 Windows 也能恢复。

---

## 致谢

- [ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm) —— AX211 原生 Wi-Fi 与 AWDL，作者 retropineaple
- [OpenCore](https://github.com/acidanthera/OpenCorePkg)、[Lilu](https://github.com/acidanthera/Lilu)、[WhateverGreen](https://github.com/acidanthera/WhateverGreen)、[VirtualSMC](https://github.com/acidanthera/VirtualSMC)、[AppleALC](https://github.com/acidanthera/AppleALC)
- [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) —— 触控板
- [IntelBluetoothFirmware](https://github.com/OpenIntelWireless/IntelBluetoothFirmware)、[BrcmPatchRAM](https://github.com/acidanthera/BrcmPatchRAM) —— 蓝牙
- [HP-VICTUS-16_Hackintosh](https://github.com/akhil838/HP-VICTUS-16_Hackintosh) —— ALC245 `layout-id 11` 的参考来源

> **ForbiddenItlwm 是运行在内核里的 beta 软件。** 出问题可能导致内核崩溃、启动循环、Wi-Fi/蓝牙丢失。请先用 U 盘上的副本 EFI 测试，确认无误后再写入内置盘。
