# DIR-X3260 OpenWrt — 深度文章索引

![D-Link DIR-X3260 A1 硬件架构概览](../images/DIRX3260_HARDWARE_ARCHITECTURE_OVERVIEW.png)

> D-Link DIR-X3260 A1 / MediaTek MT7622 + MT7915 OpenWrt 逆向工程项目

## 设备简介

DIR-X3260 A1 是一台 AX3200 Wi-Fi 6 Gigabit Router。本项目从 OEM firmware、boot/recovery、MT7622 2.4 GHz 初始化问题，一直研究到 MT7915 5 GHz 前面板 LED 的 OEM MCU 控制协议，并最终形成经过实体硬件验证的 OpenWrt Production Final。

### 官方硬件规格

| 项目 | DIR-X3260 A1 |
|---|---|
| 产品 | AX3200 Wi-Fi 6 Gigabit Router |
| CPU | 1.3 GHz dual-core |
| RAM | 512 MB |
| Flash | 128 MB |
| 2.4 GHz 标称速率 | Up to 800 Mbps |
| 5 GHz 标称速率 | Up to 2402 Mbps |
| Ethernet | 1× Gigabit WAN + 4× Gigabit LAN |
| 外置天线 | 4× dual-band high-gain dipole |
| 内置天线 | 1× 5 GHz ZeroWait DFS + 1× 2.4 GHz BLE |
| 前面板 LED | Power / Internet / 2.4 GHz Wi-Fi / 5 GHz Wi-Fi |
| 电源 | 12 V / 2 A |
| 尺寸 | 228 × 160 × 63.4 mm |
| 重量 | 436 g |

> 上表中的 CPU 主频/核心数、RAM、Flash 来自 D-Link 用户手册；接口、无线标称速率、天线、LED、电源、尺寸和重量来自 D-Link A1 datasheet。

### 本项目确认的硬件 / 运行时结构

下面这些内容来自 OEM binary 分析、OpenWrt runtime 和本项目实体设备验证，不应误写成 D-Link datasheet 的公开声明：

```text
D-Link DIR-X3260 A1
        │
        ├── MediaTek MT7622
        │       └── integrated 2.4 GHz WMAC
        │
        └── MediaTek MT7915
                └── 5 GHz radio
                        │
                        └── MCU LED engine
                                │
                                ├── OEM EXT_ID 0x17
                                ├── logical LED index 1
                                └── source 26
                                        │
                                        └── 5 GHz front LED
```

这一区分很重要：**官方产品规格**与**本项目逆向得到的芯片/驱动实现细节**属于两类证据。

---

## 推荐阅读顺序

### 1. MT7622：从 `driver own failed` 找到真正根因

**[01-《DIR-X3260 的 MT7622 Wi-Fi 为什么一直起不来：从 DriverOwn 到 IOC,WPDMA 的完整根因》.md](01-%E3%80%8ADIR-X3260%20%E7%9A%84%20MT7622%20Wi-Fi%20%E4%B8%BA%E4%BB%80%E4%B9%88%E4%B8%80%E7%9B%B4%E8%B5%B7%E4%B8%8D%E6%9D%A5%EF%BC%9A%E4%BB%8E%20DriverOwn%20%E5%88%B0%20IOC,WPDMA%20%E7%9A%84%E5%AE%8C%E6%95%B4%E6%A0%B9%E5%9B%A0%E3%80%8B.md)**

主题：

> **DIR-X3260 的 MT7622 Wi-Fi 为什么一直起不来：从 DriverOwn 到 IOC/WPDMA 的完整根因**

主要内容包括 `driver own failed`、MCU timeout、patch semaphore、WBSYS/IOC、WPDMA、DriverOwn/FirmwareOwn，以及为什么 500 ms delay 只是诊断手段。

---

### 2. MT7915：从 GPIO 假设走到 OEM MCU LED protocol

**[02-《逆向 D-Link DIR-X3260 OEM 驱动：如何找回 MT7915 5G LED 的 MCU 控制协议》.md](02-%E3%80%8A%E9%80%86%E5%90%91%20D-Link%20DIR-X3260%20OEM%20%E9%A9%B1%E5%8A%A8%EF%BC%9A%E5%A6%82%E4%BD%95%E6%89%BE%E5%9B%9E%20MT7915%205G%20LED%20%E7%9A%84%20MCU%20%E6%8E%A7%E5%88%B6%E5%8D%8F%E8%AE%AE%E3%80%8B.md)**

主题：

> **逆向 D-Link DIR-X3260 OEM 驱动：如何找回 MT7915 5G LED 的 MCU 控制协议**

核心恢复结果：

```text
MAP:
02 05 01 00 00 34 00 00

OFF:
02 01 01 00 00 00 00 00

ON:
02 01 01 01 00 00 00 00
```

文章还记录了 GPIO false lead、OEM Andes MCU path、LuCI disable/enable lifecycle，以及 STAGE55 reboot loop。

---

### 3. 整个项目：从 OEM firmware 到 Production Final

**[03-《从一台不能正常工作的路由器，到 Production Final：DIR-X3260 OpenWrt 逆向工程全记录》.md](03-%E3%80%8A%E4%BB%8E%E4%B8%80%E5%8F%B0%E4%B8%8D%E8%83%BD%E6%AD%A3%E5%B8%B8%E5%B7%A5%E4%BD%9C%E7%9A%84%E8%B7%AF%E7%94%B1%E5%99%A8%EF%BC%8C%E5%88%B0%20Production%20Final%EF%BC%9ADIR-X3260%20OpenWrt%20%E9%80%86%E5%90%91%E5%B7%A5%E7%A8%8B%E5%85%A8%E8%AE%B0%E5%BD%95%E3%80%8B.md)**

主题：

> **从一台不能正常工作的路由器，到 Production Final：DIR-X3260 OpenWrt 逆向工程全记录**

这是项目总回顾：OEM SHRS、recovery、OpenWrt、MT7622、MT7915 LED、known-good baseline、Internet regression、最终硬件 gate、production freeze、documentation 和 upstream preparation。

---

### 4. Upstream：为什么“我的机器修好了”还不够

**[04-《怎样把一个“只在自己机器上有效”的修复，整理成可以 Upstream 的 Linux,mt76 Patch》.md](04-%E3%80%8A%E6%80%8E%E6%A0%B7%E6%8A%8A%E4%B8%80%E4%B8%AA%E2%80%9C%E5%8F%AA%E5%9C%A8%E8%87%AA%E5%B7%B1%E6%9C%BA%E5%99%A8%E4%B8%8A%E6%9C%89%E6%95%88%E2%80%9D%E7%9A%84%E4%BF%AE%E5%A4%8D%EF%BC%8C%E6%95%B4%E7%90%86%E6%88%90%E5%8F%AF%E4%BB%A5%20Upstream%20%E7%9A%84%20Linux,mt76%20Patch%E3%80%8B.md)**

主题：

> **怎样把一个“只在自己机器上有效”的修复，整理成可以 Upstream 的 Linux/mt76 Patch**

最终 upstream-preparation 分类：

```text
A1   → UPSTREAM CANDIDATE
A2   → HOLD
IOC  → HOLD
A3   → DOWNSTREAM ONLY
B1   → ALREADY UPSTREAM
B2   → SEPARATE REVIEW REQUIRED
```

---

## 与 `docs/` 技术文档的区别

原有文件继续保留在原位置：

```text
docs/
├── Build-Restore-and-Reproduce.md
├── Flash-and-Recovery-Guide.md
├── MT7622-2G-Root-Cause-and-Fix.md
├── MT7915-5G-LED-OEM-Fix.md
├── Reverse-Engineering-Timeline.md
└── articles/
```

`docs/*.md` 更偏向 technical reference、最终证据、刷机/恢复和复现；`docs/articles/*.md` 更偏向长篇阅读、逆向过程、工程推理和项目回顾。

不要为了目录整齐移动原有文档，以免破坏已经发布的 Discussion 或其他相对链接。

---

## Production Final

```text
branch:
dirx3260-production-final

commit:
b6edaaf4e5df8596a742975741428217d24d11e3

tree:
40e472af26dbf5679a45da21f053ca0eccf3ec2a

tag:
dirx3260-production-final-20260906
```

Sysupgrade：

```text
SHA256:
1598fbe2d5c62516abde2fb520e480a721f261e740e6dbf04de15cb8c4a84a71
```

Recovery：

```text
SHA256:
2d63f73def5e90fb5fa84c74965a26231577b3d4d8c6fe00b8d439a7a71d6097
```

---

## 阅读建议

```text
理解 2.4 GHz 根因
    → 第 1 篇

理解 5 GHz LED 逆向
    → 第 2 篇

理解整个项目
    → 第 3 篇

研究 upstream 方法
    → 第 4 篇
```

> **不要让结论走得比证据更远。**

从 OEM binary、寄存器和 MCU command，到实体设备、Production Final，再到 upstream scope review，这条原则贯穿了整个 DIR-X3260 项目。

---

# 扩展文章

前四篇主要记录 DIR-X3260 的核心技术问题、逆向过程与 upstream 工程；下面三篇补充刷机恢复、OEM 固件格式以及 Production Final 的工程方法。

## 5. OpenWrt 刷机 / Recovery / 救砖

**[05-《刷机 - Recovery - 救砖》](05-%E3%80%8A%E5%88%B7%E6%9C%BA%20-%20Recovery%20-%20%E6%95%91%E7%A0%96%E3%80%8B.md)**

主题：

> **D-Link DIR-X3260 A1 OpenWrt 完整刷机、Recovery 与救砖指南**

主要内容：

- D-Link Recovery 与 OpenWrt 两个不同管理网段
- Recovery：`192.168.0.xxx`
- OpenWrt：`192.168.1.xxx`
- `sysupgrade -T`
- `sysupgrade` / `sysupgrade -n`
- Known-Good Baseline
- Reboot Loop 后的恢复流程
- 2.4 GHz / 5 GHz / Internet / LED 完整验证
- 冷启动验证
- 为什么重新编译的 binary 不能自动继承 Production Final 身份

这篇文章面向实际刷机和恢复，是整个系列中最偏实用操作的一篇。

---

## 6. D-Link SHRS 固件完整逆向

**[06-《SHRS 固件完整逆向》](06-%E3%80%8ASHRS%20%E5%9B%BA%E4%BB%B6%E5%AE%8C%E6%95%B4%E9%80%86%E5%90%91%E3%80%8B.md)**

主题：

> **如何逆向 D-Link SHRS 固件：从加密容器到可验证的 Firmware Round-Trip**

完整分析链：

```text
SHRS
  ↓
0x6dc header
  ↓
AES-128-CBC
  ↓
SHA-512
  ↓
4096-bit RSA
  ↓
13-byte footer
  ↓
FIT
  ↓
DTB
  ↓
SquashFS
  ↓
OEM imgdecrypt
  ↓
Firmware Round-Trip
```

这篇文章记录整个项目最早期的重要突破：如何从 D-Link OEM 加密 firmware 中恢复真正的 Linux firmware，并最终验证 container、hash、signature 与内部 payload 之间的关系。

后续 MT7622 / MT7915 OEM driver reverse engineering，正是建立在这一步之上。

---

## 7. 从 Reboot Loop 到 Production Gate

**[07-《Production Gate 工程方法》](07-%E3%80%8AProduction%20Gate%20%E5%B7%A5%E7%A8%8B%E6%96%B9%E6%B3%95%E3%80%8B.md)**

主题：

> **一次 Reboot Loop 教会我们的事：从 Known-Good Baseline 到 Production Gate**

核心原则：

```text
能编译
≠
能启动

能启动
≠
网络正常

网络正常
≠
Production Final
```

文章以本项目真实经历的：

```text
STAGE55 reboot loop
        ↓
Known-Good rollback
        ↓
Internet regression
        ↓
STAGE72 candidate
        ↓
STAGE73 provenance
        ↓
STAGE74 physical hardware validation
        ↓
Production Freeze
```

为主线，说明为什么嵌入式 firmware 必须建立：

- Known-Good Baseline
- Recovery path
- Binary SHA256
- Source provenance
- 重复状态转换测试
- Cold boot validation
- Release freeze
- Archive / backup
- Evidence boundary

这也是整个 DIR-X3260 项目最后形成的工程方法论。

---

# D-Link 官方资料

为了方便核对 DIR-X3260 A1 的官方硬件规格，本目录同时保存 D-Link 原始资料：

- **[DIR-X3260 A1 Datasheet](DIR-X3260_REVA1_Datasheet_v1.01_%28WW%29.pdf)**
- **[DIR-X3260 A1 User Manual](DIR-X3260_REVA1_Manual_v1.01_%28WW%29.pdf)**

> 产品规格以 D-Link 官方 Datasheet 和 User Manual 为依据。  
> MT7622、MT7915、WPDMA、ownership、OEM binary 以及 MT7915 MCU LED protocol 等实现细节，则来自本项目的 OEM firmware / binary 逆向、OpenWrt runtime 分析与实体硬件验证。

---

# 完整文章系列

```text
01  MT7622 Wi-Fi 根因
    DriverOwn / IOC / WPDMA

02  MT7915 5G LED MCU 逆向
    OEM Andes MCU protocol

03  DIR-X3260 OpenWrt 逆向工程全记录
    OEM → OpenWrt → Production Final

04  Upstream Linux/mt76 Patch 工程
    Downstream fix → upstream candidate

05  刷机 / Recovery / 救砖
    Safe flashing / rollback / hardware validation

06  SHRS 固件完整逆向
    Encryption / RSA / FIT / Round-Trip

07  Production Gate 工程方法
    Known-Good / provenance / freeze
```

至此，DIR-X3260 系列文章形成完整的三条主线：

```text
Firmware Reverse Engineering
        │
        ├── 06 SHRS
        ├── 01 MT7622
        └── 02 MT7915 LED

Project / Production Engineering
        │
        ├── 03 项目全记录
        ├── 05 Recovery / 救砖
        └── 07 Production Gate

Upstream Engineering
        │
        └── 04 Linux / mt76 Patch
```

> **不要让结论走得比证据更远。**
>
> 从 OEM firmware、寄存器和 MCU command，到实体设备、Production Final，再到 upstream scope review，这条原则贯穿了整个 DIR-X3260 OpenWrt 项目。
