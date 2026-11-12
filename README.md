 # HP Victus 16-r0445tx · OpenCore EFI（macOS Sequoia）

> OpenCore EFI for HP Victus 16-r0445tx (2023), with **Intel AX211 native Wi-Fi** on macOS Sequoia.
>
> 验证环境：macOS 15.8.2 (24H232) · OpenCore 1.0.9

> 适用于 **HP Victus 16-r0445tx**（2023 款，i7-13620H + RTX 4060 + AX211 + ALC245）
>
> 这份文档同时是 **README 和改动记录**：每个改动解决什么问题、依据是什么，
> 以及哪些方向试过但失败了，都写在这里。

## ⚠️ 使用前必读

### 1. 必须重新生成 SMBIOS

本 EFI 已清空个人标识，**下载后必须填入自己的值**：

| 字段 | 当前值 | 你需要做的 |
|---|---|---|
| `SystemSerialNumber` | `SERIAL-HERE` | 用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 生成 |
| `MLB` | `MLB-HERE` | 同上 |
| `SystemUUID` | `00000000-0000-0000-0000-000000000000` | 同上 |
| `ROM` | `00 00 00 00 00 00` | 填自己网卡的 MAC（6 字节） |

> 不填的话 iServices（iMessage / FaceTime / iCloud）无法使用；
> 多人共用同一套序列号会被苹果封禁。

### 2. 显卡驱动不在 EFI 内

RTX 4060 需要 **NullMoth**（社区驱动），装在：

```
/Library/Extensions/NVRM.kext
/Library/Extensions/NVRMFB.kext
/Library/Extensions/NVAccel.kext
/Library/Extensions/NVRMAGDC.kext
```

### 3. WiFi 是原生的（AX211）

EFI 用的是 **ForbiddenItlwm**：AX211 接入 macOS 自带的 `IO80211Family`，**不再需要 HeliPort**。

但 **AirDrop 和无线 Sidecar 在 Sequoia 上不可用**，原因见 §4.4。

### 4. 睡眠需要手动禁用

```bash
sudo pmset -a disablesleep 1
```

原因见 §4.5。

### 5. 调试参数建议删除

`boot-args` 里有 `-v debug=0x100 keepsyms=1`，正式使用建议去掉。

---

## 作者的话

> 这篇文章应该能够引导 AI 做出正确的决定。
>
> 我从 workbuddy 转到 dsh 终于搞定，虽然也是靠我一直在他改的同时找同款硬件配置的仓库和文章，
> 减少他乱猜的情况（虽然也有）。
>
> 此 EFI 里会有一些个人硬件标识已标注，谨记要按自己的电脑硬件信息改动，
> 不要直接套用，避免进系统失败。
>
> 同时更建议准备一个 Linux 系统（Live 镜像就行）：Linux 对 x86 笔记本硬件适配性很好，
> 而且不用像 Windows 的 pwsh 那样难运行指令。
> 利用 Linux 找到硬件所在位置尤为重要。
>
> Windows 准备好 agent 随时救 EFI。

---

## 一、机型与硬件

| 项目 | 规格 |
|---|---|
| 机型 | HP Victus by HP Gaming Laptop 16-r0445tx（2023，B&O 音频） |
| CPU | Intel Core i7-13620H（6P + 4E / 16 线程） |
| 独显 | NVIDIA GeForce RTX 4060 Laptop（8GB，Device ID `0x28A0`） |
| 核显 | Intel UHD（无 macOS 驱动） |
| WiFi / 蓝牙 | Intel AX211（CNVi，WiFi `8086:51F1`，蓝牙 `8087:0033`） |
| 音频 | Realtek ALC245（`0x10EC0245`）+ CS35L41 功放（HP Audio Boost） |
| 触控板 | Synaptics SYNA32E3（I2C-HID，`PNP0C50`，I2C0 地址 `0x2C`） |
| 网卡 | Realtek RTL8111 |
| macOS | 15.8.2（24H232） |
| 引导 | OpenCore 1.x |

---

## 二、当前工作状态

| 功能 | 状态 | 说明 |
|---|---|---|
| 显卡加速 | ✅ | Metal 3，双屏（内屏 1080p60 + 外接 1440p120） |
| 触控板 | ✅ | 可用，轮询模式（已优化 I2C 时序） |
| 蓝牙 | ✅ | Intel 原生，可配对设备 |
| WiFi | ✅ | **原生**（ForbiddenItlwm，AX211，无需 HeliPort） |
| 音频输出 | ✅ | 内置喇叭、耳机、蓝牙、USB 均可用 |
| 音频输入 | ✅ | 内置麦克风 |
| 电池 / 亮度 / USB | ✅ | |
| **睡眠唤醒** | ❌ | 固件级恢复失败，建议禁用睡眠 |
| **AirDrop / 无线 Sidecar** | ❌ | Sequoia 上 AWDL 不可用，见 §4.4 |
| **B&O DSP 调音** | ⚠️ | 缺功放 DSP，音色偏"空"，需软件 EQ 补偿 |

---

## 三、改动总览

| # | 区域 | 改动 | 类型 |
|---|---|---|---|
| 1 | 触控板 | DSDT 里 I2C0 的 `TPD0._CRS` → `XCRS` | 改 DSDT.aml |
| 2 | 触控板 | 新增 `SSDT-TPD0CRS.aml` | 新增 ACPI |
| 3 | 触控板 | `SSDT-I2CTM.aml` 的 FMCN 提速 178kHz → 400kHz | 改 ACPI |
| 4 | 触控板 | 禁用 `SSDT-TPD0.aml`（旧的错误尝试） | 配置 |
| 5 | 触控板 | 禁用 `SSDT-I2C0IRQ.aml`（方向不对） | 配置 |
| 6 | 蓝牙 | `bluetoothExternalDongleFailed` 0x01 → **0x00** | 配置 |
| 7 | 蓝牙 | IntelBluetoothFirmware / IntelBTPatcher 2.5.0 → **2.5.2** | 换 kext |
| 8 | 音频 | `layout-id` 13 → **11** | 配置 |
| 9 | 音频 | 禁用 `SimpleHDA.kext` | 配置 |
| 10 | 音频 | 禁用 `CirrusAudioFixup.kext` | 配置 |
| 11 | WiFi | 用 **ForbiddenItlwm** 替换 `itlwm` / `AirportItlwm` | 换 kext |
| 11.1 | WiFi | 新增 `BTLEAdvFixup.kext`（蓝牙 LE 广播） | 新增 kext |
| 11.2 | WiFi | 禁用 `IntelBTPatcher.kext`（与 BTLEAdvFixup 冲突） | 配置 |
| 11.3 | WiFi | `boot-args` 追加 19 个 AWDL 参数 | 配置 |
| 12 | 睡眠 | 加 boot-arg `-nvrmnopm`（未解决） | 配置 |
| 13 | 清理 | 禁用 `VoodooGPIOTGL.kext`（实验性，未使用） | 配置 |

---

## 四、详细说明

### 1. 触控板（最重要的改动）

#### 问题现象

```
I2C0（触控板所在的 I2C 控制器）没有中断
  → VoodooI2CControllerDriver::start() 失败
    → 驱动链断裂，触控板完全不可用
```

#### 根因

```
DSDT 里 TPD0._CRS 声明了 APIC 中断（GSI 0x1B = 27）
I2C0 的 _PRT 也要 GSI 0x1B
→ macOS 把 27 分给了 TPD0，I2C0 得到空
→ 控制器驱动启动失败
```

证据：

```
ioreg:  TPD0.IOInterruptSpecifiers  = <1b000000 03000000>   ← 占着 27
ioreg:  I2C0.IOInterruptSpecifiers  = ()                    ← 空
ioreg:  I2C1.IOInterruptSpecifiers  = <2800000007000000>    ← 有中断，驱动正常挂载
```

#### 解决

**（a）改 DSDT.aml：把 I2C0 的 `TPD0._CRS` 重命名为 `XCRS`**

- 位置：文件偏移 `0x77C60`
- 替换：`...14 4C 04 **5F** 43 52 53 00` → `...14 4C 04 **58** 43 52 53 00`
  （`_CRS` → `XCRS`，单字节改动）
- 之后**重算 ACPI 校验和**（偏移 9，本文件结果为 `0xA7`）

> ⚠️ 注意：不要用 OpenCore 的 ACPI/Patch 做这个改名。
> OC 的顺序是「先打补丁 → 再用 ACPI/Add 里的 DSDT.aml 替换」，
> 补丁会打在固件原版 DSDT 上，随后被整体覆盖，等于没做。
> **必须直接改 DSDT.aml 文件本身。**

**（b）新增 `SSDT-TPD0CRS.aml`**

内容：给 `\_SB.PC00.I2C0.TPD0` 提供新的 `_CRS`，只返回 I2cSerialBus（地址 `0x2C`，400kHz），
**不声明任何中断**。

```asl
DefinitionBlock ("", "SSDT", 2, "HP", "TPD0CRS", 0x00000000)
{
    External (_SB_.PC00.I2C0.TPD0, DeviceObj)
    Scope (_SB_.PC00.I2C0.TPD0)
    {
        Method (_CRS, 0, NotSerialized)
        {
            Name (SBFB, ResourceTemplate ()
            {
                I2cSerialBusV2 (0x002C, ControllerInitiated, 0x00061A80,
                    AddressingMode7Bit, "\\_SB.PC00.I2C0",
                    0x00, ResourceConsumer, , Exclusive,
                    )
            })
            Return (SBFB)
        }
    }
}
```

**（c）`SSDT-I2CTM.aml` 提速**

原 DSDT 的 I2C 时序方法 `SSCN`/`FMCN` 被包在 `If (USTP)` 里，条件不成立时不可用；
该 SSDT 无条件提供，并把快速模式计数器从

```
FMCN: HCNT=0x0101 LCNT=0x012C  → 实际约 178 kHz
```

改为

```
FMCN: HCNT=0x005A LCNT=0x009D  → 400 kHz
```

效果：每次读 64 字节报告从 3.2ms 降到 1.4ms，解决了"画着画着顿一下"的问题。

**（d）禁用旧尝试**

| 文件 | 为什么禁用 |
|---|---|
| `SSDT-TPD0.aml` | 地址写成 `0x15`（实际是 `0x2C`），且与 DSDT 重复定义被丢弃 |
| `SSDT-I2C0IRQ.aml` | 改的是 `_CRS` 声明 IRQ，方向不对，且未启用过 |

#### 结果

```
I2C0@15 → VoodooI2CPCILakeController → VoodooI2CControllerNub
        → VoodooI2CControllerDriver → TPD0 (VoodooI2CDeviceNub)
        → VoodooI2CHIDDevice → VoodooI2CPrecisionTouchpadHIDEventDriver
        → AppleMultitouchTrackpadHIDEventDriver
```

触控板工作在**轮询模式**（VoodooI2CHID 的 fallback），可用。

> 未做成 GPIO 中断的原因：VoodooGPIO 的 TigerLake 类缺失，
> 自编译版本被 OpenCore 拒绝（`MachoInitializeContext` 校验失败）。见 §7。

---

### 2. 蓝牙

#### 问题

```
Chipset: BCM_4350C2（假状态）
Firmware Version: v0 c0
Address: NULL    State: Off
```

#### 根因

`bluetoothExternalDongleFailed` 这个 NVRAM 变量被写成了 **0x01**。

这是 macOS 在蓝牙固件加载失败后**自己写入**的粘性标志，值为 1 时它以后
每次开机都直接跳过初始化。原 config 里竟然主动写成了 0x01。

#### 解决

```
NVRAM/Add/7C436110-.../bluetoothExternalDongleFailed : 0x01 → 0x00
```

同时把 Intel 蓝牙三件套升级到 2.5.2（来自另一台同网卡机器的 EFI）。

#### 结果

```
Chipset: THIRD_PARTY_DONGLE
Firmware Version: v12490 c12490
BluetoothTransportConnected = Yes
MAC: b7:a6:5a:fd:aa:92
```

原生蓝牙可用。

---

### 3. 音频

#### 问题

```
耳机（3.5mm）有声音
内置扬声器完全无声
系统里能看到"内置扬声器"设备
```

#### 根因

`layout-id 13`（HP Omen 2020 的 ALC245 布局）的喇叭路径与本机不匹配。
**codec 型号相同（ALC245）不代表 pin 路径相同。**

#### 解决

```
layout-id : 13 → 11
```

依据：另一台 HP Victus 16（AMD Ryzen 版）的黑苹果仓库

```
https://github.com/akhil838/HP-VICTUS-16_Hackintosh
README:  Audio | Realtek ALC245
         Audio | ✅ | AppleALC working with layout-id 11
```

同 codec、同系列，照做后内置喇叭出声。

#### 结果

- 内置喇叭 ✅ 出声，立体声左右分离正常
- 音色偏"空"：缺 CS35L41 功放的 B&O DSP 调音（见 §7）

#### 同时禁用

| kext | 原因 |
|---|---|
| `SimpleHDA.kext` | 单独使用会导致内置音频设备消失 |
| `CirrusAudioFixup.kext` | 本机无 Cirrus 硬件，且有匹配 `VoodooI2CDeviceNub` 的 personality，会抢触控板 |

#### AppleALC 对 ALC245 的布局只有 3 个

| ID | 注释 | 本机结果 |
|---|---|---|
| **11** | ALC235 for Lenovo | ✅ 可用 |
| 12 | ALC245 for Lenovo | 未测试 |
| 13 | ALC245 for HP Omen 2020 | ❌ 喇叭静音 |

---

### 4. WiFi（AX211 原生）

#### 早期尝试：AirportItlwm 失败

`AirportItlwm`（原生 WiFi）在 15.8.2 上拒绝加载：

```
OCAK: Failed to patch symbol __ZN25IOSkywalkNetworkInterface13getTSOOptionsE...
OCAK: Vtable patching failed for kext com.zxystd.AirportItlwm
```

原因：AirportItlwm 需要 OC 在启动时给内核 `IOSkywalkNetworkInterface` 打 vtable 补丁，
而 `getTSOOptions` 这个符号在 macOS 15.8.2 的内核里**不存在**，整个 kext 被拒绝注入。

#### 现在的方案：ForbiddenItlwm

[ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm) 是 AirportItlwm 的 fork，
针对每个 macOS 版本单独构建，直接接入 macOS 自带的 `IO80211Family`。

```
ForbiddenItlwm.kext  ──  IO80211Family  ──  系统原生 Wi-Fi 菜单
```

**改动内容：**

| 项目 | 内容 |
|---|---|
| 移除 | `itlwm.kext`、`AirportItlwm.kext` |
| 新增 | `ForbiddenItlwm.kext`（Sequoia 构建） |
| 新增 | `BTLEAdvFixup.kext`（Sequoia 构建） |
| 禁用 | `IntelBTPatcher.kext`（与 BTLEAdvFixup 冲突，必须禁用） |
| boot-args | 追加 19 个 AWDL 参数 |

**Kernel 加载顺序**（有依赖，不能乱）：

| # | BundlePath | MinKernel | MaxKernel |
|---|---|---|---|
| 1 | `Lilu.kext` | | |
| 2 | `IntelBluetoothFirmware.kext` | | |
| 3 | `BlueToolFixup.kext` | `21.0.0` | |
| 4 | `BTLEAdvFixup.kext` | `24.0.0` | `24.99.99` |
| 5 | `ForbiddenItlwm.kext` | `24.0.0` | `24.99.99` |

**验证：**

```bash
kextstat | grep -i forbidden
ioreg -l | grep -E '"ItlwmAWDL(BuildCheck|InterfaceBuild)"'
ifconfig en0 | grep "inet "
```

健康状态：

```
"ItlwmAWDLBuildCheck"     = "verified"
"ItlwmAWDLInterfaceBuild" = "forbidden-0.5.0-test1-sequoia"
```

#### 已知限制：AirDrop / Sidecar 不可用

两个方向都搜不到对方。原因是 macOS 始终没有完成 `awdl0` 接口的创建：

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

接口被打开后**同一毫秒内就被关闭**。上游已知问题：[issue #13](https://github.com/retropineaple/ForbiddenItlwm/issues/13)。

无线 Sidecar 依赖同一套 AWDL；有线 Sidecar 还需要 **VideoToolbox HEVC 编码器**，
本机核显无驱动、NVIDIA 驱动也不提供 VideoToolbox 编码器，因此同样不可用。
USB 配对成功、Finder 能看到 iPad，但它不会出现在显示器列表里。

**替代方案**：Deskreen（免费，浏览器方案）、Duet Display（付费，USB）。

---

### 5. 睡眠（部分解决）

#### 已解决：睡下去立刻醒

**根因是蓝牙。**

```
蓝牙开启时：睡眠 → 蓝牙相关活动把它叫醒（HID Activity / UserActivity）
关闭蓝牙后：能正常进入睡眠，不再秒醒
```

之前日志里的证据（蓝牙还开着时）：

```
19:55:33  Entering Sleep state                              （1 秒后）
19:55:34  DarkWake from Invalid [CDN] : due to /UserActivity Assertion
```

以及 USB 无线接收器类似——设备在睡眠瞬间重新枚举会产生 HID 事件。

#### 未解决：睡着了唤不醒

当前现象（用户实测）：

- **能正常入睡**（关闭蓝牙后）
- **按键鼠无反应**，唤醒不了
- **硬件像是断电了**（USB 设备无供电）
- 触控板唤醒未测试

已尝试且无效：

```
pmset -a hibernatemode 0 standby 0 autopoweroff 0
pmset -a powernap 0 womp 0 proximitywake 0
boot-args: -nvrmnopm
拔掉外接显示器 / 拔掉 USB 无线接收器
```

早期日志里出现过：

```
Failure during sleep: 0x0000001F : EFI/Bootrom Failure after last point of entry to sleep
Failure during sleep: 0x3405F9000000012 : Informing clients about upcoming changes
```

固件支持 S3（DSDT 里有 `_S3`）。

#### 可能的方向（未验证）

| 方向 | 说明 |
|---|---|
| USB 端口唤醒能力 | S3 下 XHC 是否保持供电、端口是否允许 wake |
| `pmset -a lidwake 1` + 合盖/开盖 | 测试盖子唤醒是否有效 |
| 电源键唤醒 | 需要 EC 支持，黑苹果上常失效 |
| PS2 键盘唤醒 | VoodooPS2 一般不实现 S3 唤醒 |
| `_PRW` / GPRW 补丁 | 部分机器需要修正唤醒事件路由 |

#### 建议

现阶段最稳的做法：

```bash
sudo pmset -a disablesleep 1
```

> 注意：`pmset` 设置是**系统级**的，不在 EFI 里，不随 EFI 发布。

### 6. 显卡（不在 EFI 内）

NVIDIA RTX 4060 的驱动是 **NullMoth**（社区驱动），安装在：

```
/Library/Extensions/NVRM.kext
/Library/Extensions/NVRMFB.kext
/Library/Extensions/NVAccel.kext
/Library/Extensions/NVRMAGDC.kext
```

**不在 EFI 里**，需要单独安装。EFI 里只保留了相关的 boot-args。
***
其实ai也不知道我干了什么哈哈哈
用人话来说就是直接用1401的tools里有个gpu驱动直接装就好
这款机型如果用核显的话会很卡 现存只有8mb 配置好独显就有8gb了 动画什么都流畅舒服
***

---

## 五、当前配置快照

### boot-args

```
-v debug=0x100 keepsyms=1 ctrsmt=full
nvfb=1 nvaccel=1 nvfbheads=4 -nvkmsnosmooth
amfi_get_out_of_my_way=0x1 amfi=0x80
ipc_control_port_options=0 gfxrst=4 nvrmacpibl=1
-btlfxallowanyaddr -nvrmnopm -vi2c-force-polling
-btleadv -itlawdlcoexprobe -itlawdliface -itlawdlnativeup -itlawdlcoexrx -itlawdlcoextx
-itlawdlcoexmulti -itlawdlcoexdata -itlawdlmitx -itlawdlelicit -itlawdlcaps
-itlawdlsvcmerge -itlawdlawsync itlawdlintcoal=4 -itlawdlcoexbatx -itlawdlcoexpeerpause
-itlawdlcoexbatrace itlawdltxqueue=4096
```

> AWDL 参数含义见 [ForbiddenItlwm BOOT-FLAGS.md](https://github.com/retropineaple/ForbiddenItlwm/blob/main/docs/BOOT-FLAGS.md)。
> 升级 macOS 前请先去掉 `-btleadv`（BTLEAdvFixup 会修改 `bluetoothd`）。

> 开源前建议：`-v debug=0x100 keepsyms=1` 是调试参数，正式使用可去掉。

### 关键配置

```
csr-active-config      = 0x0A43
DeviceProperties 音频   = layout-id 11
DeviceProperties 显卡   = PciRoot(0x0)/Pci(0x2,0x0)（空）
```

### ACPI（启用）

```
DSDT.aml                 ← 已把 I2C0 的 TPD0._CRS 改名为 XCRS
SSDT-SBUS.aml
SSDT-EC.aml
SSDT-GPI0.aml
SSDT-MCHC.aml
SSDT-PLUG-ALT.aml
SSDT-RTCAWAC.aml
SSDT-USB-Reset.aml
SSDT-USBX.aml
SSDT-XOSI.aml
SSDT-I2CTM.aml           ← I2C 时序提速到 400kHz
SSDT-TPD0CRS.aml         ← 触控板 _CRS 覆盖
```

### ACPI（禁用）

```
SSDT-TPD0.aml            ← 旧的错误尝试
SSDT-I2C0IRQ.aml         ← 方向不对
```

### Kernel（启用）

```
Lilu.kext
IntelBluetoothFirmware.kext
BlueToolFixup.kext
BTLEAdvFixup.kext
ForbiddenItlwm.kext
VirtualSMC.kext
CPUFriend.kext
AMFIPass.kext
BrightnessKeys.kext
CpuTopologyRebuild.kext
CpuTscSync.kext
NVMeFix.kext
RealtekRTL8111.kext
RestrictEvents.kext
SMCBatteryManager.kext
SMCLightSensor.kext
SMCProcessor.kext
VoodooGPIO.kext
VoodooI2CServices.kext
VoodooI2C.kext
VoodooI2C.kext/Contents/PlugIns/VoodooInput.kext
VoodooI2CHID.kext
VoodooPS2Controller.kext
VoodooPS2Controller.kext/Contents/PlugIns/VoodooPS2Keyboard.kext
WhateverGreen.kext
XHCI-unsupported.kext
AppleALC.kext
HPVictusUSBPorts.kext
```

### Kernel（禁用）

```
CPUFriendDataProvider.kext
SMCSuperIO.kext
USBToolBox.kext
UTBDefault.kext
IntelBTPatcher.kext       ← 与 BTLEAdvFixup 冲突，必须禁用
VoodooGPIOTGL.kext       ← 实验性，自编译被 OC 拒绝
VoodooI2CELAN.kext
VoodooPS2Controller.kext/Contents/PlugIns/VoodooPS2Mouse.kext
VoodooPS2Controller.kext/Contents/PlugIns/VoodooPS2Trackpad.kext
SimpleHDA.kext
CirrusAudioFixup.kext
VoodooRMI.kext
UTBMap.kext
```

---

## 六、已知问题

| 问题 | 状态 | 原因 |
|---|---|---|
| 睡眠唤醒 | 部分解决 | 蓝牙致秒醒已解决；但睡着后键鼠唤不醒，硬件似断电 |
| AirDrop 不可用 | 未解决 | Sequoia 上 `awdl0` 创建失败（上游 issue #13） |
| 无线 Sidecar 不可用 | 未解决 | 依赖 AWDL；有线还缺 HEVC 编码器 |
| 音色偏"空" | 部分解决 | 缺功放 DSP，需软件 EQ |
| 触控板为轮询模式 | 部分解决 | GPIO 中断所需驱动无法加载 |
| 触控板按压力度偏敏感 | 未解决 | 驱动层触点判定 |

---

## 七、开源前注意事项

### 1. 不要发布的内容

| 文件 | 原因 |
|---|---|
| `config.plist.bak-*` | 历史备份，含实验参数 |
| `SystemSerialNumber` / `MLB` / `SystemUUID`（PlatformInfo 里） | **属于个人唯一标识，必须清空或替换成示例值** |
| `opencore-*.txt` 日志 | 含硬件路径、序列号等 |

### 2. 必须说明的依赖

- NVIDIA 显卡需要 **NullMoth** 驱动（`/Library/Extensions/`），不在 EFI 内 （见1401软件）
- WiFi 已原生（ForbiddenItlwm），不再需要 HeliPort
- 睡眠需手动执行 `sudo pmset -a disablesleep 1`

### 3. 版本敏感项

- 整套配置针对 **macOS 15.8.2** 验证
- 升级 macOS 小版本后需重新验证：触控板、音频 layout、蓝牙
- `layout-id 11` 是 ALC245 的 "Lenovo" 布局，但在 HP Victus 上可用（实测）

---

## 八、回滚

所有改动都有备份：

```bash
# config 备份
ls /Volumes/NO\ NAME/EFI/OC/config.plist.bak-*

# ACPI 备份
ls /Volumes/NO\ NAME/EFI/OC/ACPI/*.bak-*

# 关键备份
config.plist.bak-alcid11-*    ← layout 11 之前的配置（layout 13）
DSDT.aml.bak-before-crs-*     ← 改名之前的 DSDT
SSDT-I2CTM.aml.bak-178k-*     ← 提速之前的 I2C 时序
```

---

## 九、参考资料

| 来源 | 用途 |
|---|---|
| [AppleALC](https://github.com/acidanthera/AppleALC) | 音频布局资源 |
| [VoodooI2C](https://github.com/VoodooI2C/VoodooI2C) | 触控板驱动 |
| [HP-VICTUS-16_Hackintosh](https://github.com/akhil838/HP-VICTUS-16_Hackintosh) | **layout-id 11 的依据来源**（同 codec 同系列） |
| [国光黑苹果教程 6-3](https://apple.sqlsec.com/6-实用姿势/6-3/) | 触控板中断/GPIO 原理 |
| [OpenCore](https://github.com/acidanthera/OpenCorePkg) | 引导 |
| [ForbiddenItlwm](https://github.com/retropineaple/ForbiddenItlwm) | **AX211 原生 WiFi 驱动** |
| [issue #13](https://github.com/retropineaple/ForbiddenItlwm/issues/13) | Sequoia 上 awdl0 不创建 |

---

*最后更新：2026-11-12*

---

## 十、发布前清理记录（2026-11-12）

### 1. 已移除的个人信息

| 字段 | 原值 | 替换为 |
|---|---|---|
| `PlatformInfo/Generic/SystemSerialNumber` | 真实序列号 | `SERIAL-HERE` |
| `PlatformInfo/Generic/MLB` | 真实主板序列 | `MLB-HERE` |
| `PlatformInfo/Generic/SystemUUID` | 真实 UUID | `00000000-0000-0000-0000-000000000000` |
| `PlatformInfo/Generic/ROM` | 网卡 MAC（6 字节） | `00 00 00 00 00 00` |

> ⚠️ **使用者必须自己生成**
>
> 下载后请用 [GenSMBIOS](https://github.com/corpnewt/GenSMBIOS) 或 OpenCore Configurator
> 生成新的 `SystemSerialNumber` / `MLB` / `SystemUUID` / `ROM`，
> 否则 iServices（iMessage / FaceTime / iCloud）无法使用，且多人共用同一套序列号会被苹果封禁。

`SystemProductName` 保持 `MacBookPro16,2`（机型标识，不含个人信息）。

### 2. 已移除的备份文件（共 89 项）

移出到仓库之外（`_20261112_removed/`），内容包括：

```
.bak-* 配置与 ACPI 备份        70 个
config 变体（含 5 个 8.3 截断名 CONF~*.BA0）  9 个
Kexts/*.bak-*、*.bak/           8 个
Resources.bak-default-*         1 个
itlwm.kext、AirportItlwm.kext  2 个（旧驱动，config 已无引用，共 30MB）
```

**这些备份原本都含真实序列号**，不能随 EFI 发布。

### 3. 保留但需要注意的文件

| 文件 | 说明 |
|---|---|
| `OC/Tools/NullMothSafe.efi` | NVIDIA 驱动的辅助工具 |
| `OC/Kexts/VoodooGPIOTGL.kext` | 实验性 kext，**已禁用**，OpenCore 会拒绝加载；保留仅为记录尝试过程，可自行删除 |
| `OC/Kexts/ForbiddenItlwm.kext` | AX211 原生 WiFi 驱动（Sequoia 构建） |
| `OC/Kexts/SimpleHDA.kext`、`CirrusAudioFixup.kext` | **已禁用**，会导致问题，保留备查 |

### 4. 建议使用者自行调整

| 项目 | 建议 |
|---|---|
| `boot-args` 里的 `-v debug=0x100 keepsyms=1` | 调试参数，正式使用建议删除 |
| `Misc/Boot/Timeout` | 按需调整启动菜单停留时间 |
| `PlatformInfo/Generic/ROM` | 填自己网卡的 MAC（6 字节） |

### 5. 最终 EFI 结构

```
EFI/
├── BOOT/
│   └── BOOTx64.efi
└── OC/
    ├── ACPI/          12 张启用 + 2 张禁用
    ├── Drivers/       HfsPlus / OpenRuntime / ResetNvramEntry / OpenCanopy
    ├── Kexts/         36 个（其中 13 个在 config 里禁用）
    ├── Resources/     OpenCanopy 主题（Flavours-Sequoia）
    ├── Tools/         NullMothSafe.efi
    ├── OpenCore.efi
    └── config.plist   ← 已清理，无个人信息
```

- `Kernel/Add` 共 41 项：28 启用、13 禁用
- 体积：约 60 MB
